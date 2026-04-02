# ToRoMe Store — Phân Tích Hệ Thống Multi-Agent

> Phân tích toàn diện: AI layer + Spring Boot Backend + React Frontend
> Phục vụ: maintain, scale, debug production, phát triển agent mới
> Date: 2026-04-02

---

## Mục lục

1. [Tổng quan kiến trúc từng layer](#1-tổng-quan-layer)
2. [Sơ đồ luồng dữ liệu end-to-end](#2-sơ-đồ-luồng-dữ-liệu)
3. [Phân tích hệ thống Agent](#3-phân-tích-hệ-thống-agent)
4. [Memory & Context management](#4-memory--context)
5. [Technical Debt & Điểm yếu](#5-technical-debt--điểm-yếu)
6. [Khả năng mở rộng](#6-khả-năng-mở-rộng)
7. [Đề xuất cải thiện](#7-đề-xuất-cải-thiện)
8. [Kết luận](#8-kết-luận)

---

## 1. Tổng quan từng Layer

### 1.1 `fashion-ai-agent/` — AI Layer (Python / FastAPI)

#### Kiến trúc Agent

```
Orchestrator (:8000)  ── A2A ──→  Search Agent (:8001)
                                ── A2A ──→  Stylist Agent (:8002)
                                ── A2A ──→  Order Agent (:8003)
```

**4 Agent** + 1 Orchestrator làm entry point duy nhất.

#### Orchestrator — Routing & Planning

Điểm vào: `POST /chat` endpoint (`main.py`)

**2 mode routing**:

| Mode | Cơ chế | Trigger |
|------|--------|---------|
| **LLM-based** | OpenAI tool-calling → `send_message` tool → gọi worker agent | Mặc định |
| **Keyword fallback** | Regex match `tags`/`examples` → gọi thẳng agent | LLM không gọi tool ở iteration đầu |

**Dual abstraction** — mỗi layer đều có 2 cách hoạt động:

```
Orchestrator layer:
  RoutingAgent     — LLM routing + A2A send_message
  PlanningAgent    — Tạo ExecutionPlan (SINGLE/SEQUENTIAL/PARALLEL)
  PlanExecutor     — Chạy plan, truyền context giữa các step
  ConversationManager — Smart memory (3 pairs + summary)
  OrchestrationSkill — route_to_agent + plan_and_execute tools

Worker agent layer:
  BaseAgent        — Skill registry + A2A card builder
  SkillBasedExecutor — ReAct loop per session (max 8 iterations)
  Skill            — Metadata + Tool definitions + execute_tool
```

**Orchestrator internal flow**:
```
/chat request
  → SkillBasedExecutor._tool_calling_loop
    → OrchestrationSkill.execute_tool("plan_and_execute" hoặc "route_to_agent")
      → RoutingAgent.send_message("Agent X", task)
        → A2A HTTP → worker:800X
          → Worker: SkillBasedExecutor._tool_calling_loop
            → Skill.execute_tool → BackendClient → backend API
          → Worker trả TextPart + DataPart về Orchestrator
      → Trả ChatResponse { response, agent_used, data }
```

#### 3 Worker Agents

| Agent | Port | Skill | Tools | Backend gọi |
|-------|------|-------|-------|-------------|
| **Search** | 8001 | `ProductSearchSkill` | `search_products` | `POST /api/products/vector-search` |
| **Stylist** | 8002 | `OutfitRecommendationSkill` | `search_products`, `get_product_catalog`, `get_user_preferences` | `POST /api/products/vector-search`, `GET /api/products`, `GET /api/users/{id}` |
| **Order** | 8003 | `OrderProcessingSkill` HOẶC `OrderWithSearchSkill` (env flag) | `search_products`, `add_to_cart`, `add_multiple_to_cart`, `create_order`, `get_payment_link` | Cart, Order, Payment APIs |

#### Cấu trúc file AI layer

```
fashion-ai-agent/
├── services/
│   ├── orchestrator/
│   │   ├── main.py                    # FastAPI app + lifespan init
│   │   ├── routing_agent.py           # LLM routing + A2A send + keyword fallback
│   │   ├── planning_agent.py          # ExecutionPlan creation (SINGLE/SEQ/PARALLEL)
│   │   ├── plan_executor.py           # Execute plan, pass context between steps
│   │   ├── conversation.py           # SmartConversationManager (3 pairs + summary)
│   │   ├── schemas.py                 # ChatRequest/ChatResponse Pydantic
│   │   └── skills/
│   │       └── orchestration.py      # OrchestrationSkill (LLM dùng tools để route)
│   ├── search/
│   │   ├── main.py, agent.py         # A2A server entry
│   │   └── skills/product_search.py  # Vector search skill
│   ├── stylist/
│   │   ├── main.py, agent.py
│   │   └── skills/outfit_recommendation.py
│   └── order/
│       ├── main.py, agent.py
│       └── skills/
│           ├── order_processing.py    # Direct backend search
│           └── order_with_search.py   # A2A delegation to Search Agent
└── shared/
    ├── backend_client.py              # HTTP client cho backend APIs
    ├── config.py                      # Settings (LLM, Backend URL)
    └── base_agent/
        ├── agent.py                   # BaseAgent container
        ├── skill.py                  # Skill ABC
        ├── executor.py                # SkillBasedExecutor (ReAct loop)
        ├── memory.py                  # AgentMemory + MemoryStore
        └── tool.py                    # ToolDefinition
```

#### ExecutionPlan — 3 modes

```python
# planning_agent.py — ExecutionMode enum
SINGLE:      1 agent
SEQUENTIAL:  agent → agent → agent (context passed)
PARALLEL:    agent + agent + agent (concurrent)
```

**Sequential context passing** — PlanExecutor `_build_task_with_context` truyền product_ids từ Search Agent output sang Order Agent input:
```
Task gửi đến Order Agent:
  "Add to cart [SYSTEM: JWT_TOKEN=xxx] [user_id=1]
   Products found: 1. ID:5 - White Shirt - $29
   Product IDs: [5]"
```

---

### 1.2 `fashion-ai-backend/` — Spring Boot Microservices

#### 5 Services + 1 Gateway

```
                           ┌── user-service (:9001)  → PostgreSQL (user)
                           ├── product-service (:9002) → PostgreSQL + Elasticsearch
Gateway (:9000)  ──────────┼── order-service (:9003)  → PostgreSQL (cart + orders)
  Spring Cloud Gateway     └── payment-service (:9004)
  StripPrefix: /api/ai/* → :8000
```

#### Gateway routing (`application.yml`)

```yaml
routes:
  - Path=/api/auth/**     → user-service:9001
  - Path=/api/users/**    → user-service:9001
  - Path=/api/products/** → product-service:9002
  - Path=/api/orders/**   → order-service:9003
  - Path=/api/cart/**     → order-service:9003
  - Path=/api/payments/** → payment-service:9004
  - Path=/api/ai/**       → AI_ORCHESTRATOR_URL (:8000), StripPrefix=2
```

#### Key Observations

| Aspect | Detail |
|--------|--------|
| Inter-service comms | REST (Feign/WebClient) — service gọi thẳng nhau |
| ES sync | `ElasticsearchSyncService` — sync product từ PostgreSQL → ES sau mỗi save |
| JWT | Single secret shared, `JwtAuthFilter` validate token |
| Security | `permitAll()` trên tất cả `/api/**` — **KHÔNG có auth check ở gateway** |
| VNPay | 1 endpoint: `GET /api/payments/vnpay-gen?orderId=` |

---

### 1.3 `fashion-ai-frontend/` — React 19 + Vite + TailwindCSS

#### 3 Zustand Stores độc lập

| Store | Gọi API | Quản lý |
|-------|---------|---------|
| `authStore` | `POST /api/auth/login` | User login/logout, JWT → localStorage |
| `cartStore` | `/api/cart/*` (REST) | Cart items — **server là source of truth** |
| `chatStore` | `POST /api/ai/chat` | Messages + **AI response → cart sync** |

#### AI → Cart sync (chatStore.ts)

```typescript
// Frontend parse action string từ AI response rồi tự gọi cart API
if (action === 'add_to_cart') {
  await useCartStore.getState().addItem({ user_id, product_id, quantity: 1 });
}
if (action === 'add_multiple_to_cart') {
  for (const item of items) {
    await useCartStore.getState().addItem({...});
  }
}
```

---

## 2. Sơ đồ luồng dữ liệu End-to-End

### Flow A: "Show me black jackets" (Single Agent)

```
1. FE: sendMessage("black jackets")
   → POST /api/ai/chat { user_id, message, session_id }
   → Gateway strips /api → forwards :8000/chat

2. Orchestrator: /chat endpoint
   → SkillBasedExecutor._tool_calling_loop
   → OrchestrationSkill.execute_tool("route_to_agent")
   → RoutingAgent.run() → LLM chọn "Search Agent"

3. RoutingAgent.send_message("Search Agent", "black jackets [user_id=1]")
   → A2A HTTP POST → search:8001

4. Search Agent: SkillBasedExecutor.execute()
   → ProductSearchSkill.execute_tool("search_products", {query})
   → BackendClient.vector_search("black jackets", top_k=5)
   → POST :9000/api/products/vector-search
   → Backend: ES semantic search → products

5. Search Agent trả TextPart + DataPart(products) về Orchestrator

6. Orchestrator trả ChatResponse { response, agent_used, data }
   → Gateway → FE
   → chatStore hiển thị message + store products
```

### Flow B: "Find white shirt and add to cart" (Sequential Multi-Agent)

```
PlanningAgent tạo ExecutionPlan(mode=SEQUENTIAL, steps=[search, order])

PlanExecutor._execute_sequential([

  Step 1: Search Agent
    → vector_search("white shirt")
    → trả products

  Step 2: Order Agent
    → PlanExecutor._build_task_with_context đính kèm product_ids vào task
    → Task = "Add to cart [user_id=1] Products found: ... Product IDs: [3,4]"
    → BackendClient.add_to_cart() → order-service
])
```

### Flow C: "Show me shoes and pants" (Parallel Multi-Agent)

```
PlanExecutor._execute_parallel([
  Step 1: Search Agent ("shoes")
  Step 2: Search Agent ("pants")
])
→ asyncio.gather() → concurrent execution
→ Combine text: "\n\n---\n\n".join(results)
```

---

## 3. Phân tích hệ thống Agent

### 3.1 Orchestrator

**Điểm mạnh:**
- Agent card auto-discovery — tự discover worker agents khi startup
- Dual routing (LLM + keyword fallback) — graceful degradation
- Context injection: `[user_id=X]` + `[SYSTEM: JWT_TOKEN=xxx]` vào mọi task
- Multi-mode planning đủ linh hoạt cho use cases từ đơn giản đến phức tạp

**Điểm yếu:**
- Mỗi request gây ra **2 LLM calls**: 1 ở Orchestrator (routing), 1 ở worker (execution) → latency ~2-4s
- Không có retry/exponential backoff khi worker fail
- Tool deduplication by name có thể conflict nếu 2 skills cùng tên tool

### 3.2 Worker Agents — ReAct Loop

```
for iteration in range(8):
    LLM response = openai.chat.completions.create(messages + tools)
    if tool_calls:
        result = await skill.execute_tool(tool_name, args)
        messages.append({"role": "tool", "content": result.content})
    else:
        break
```

**Điểm mạnh:** Self-contained, stateless giữa requests. Skill registry cho phép extend dễ dàng.

**Điểm yếu:**
- Mỗi worker agent gọi LLM riêng → 1 multi-agent request = 3-4 LLM calls
- `OrderProcessingSkill` định nghĩa `search_products` riêng, gọi trực tiếp `BackendClient.vector_search` — **violates single responsibility**

### 3.3 A2A Communication

Đúng chuẩn Google A2A SDK:
- `/.well-known/agent-card.json` discovery
- `A2AClient.send_message()` với `SendMessageRequest`
- `DefaultRequestHandler` + `InMemoryTaskStore`
- Response: `TextPart` + `DataPart` (structured data)

---

## 4. Memory & Context

### 4.1 In-Memory Only — CRITICAL RISK

```
SmartConversationManager (orchestrator):
  dict[user_id → list[Message]]
  Sliding window: last 3 pairs (6 msg) full + older → 1 summary
  ❌ KHÔNG persist — chết khi container restart

AgentMemory (per worker):
  dict[session_id → AgentMemory]
  max 10 messages/session
  ❌ KHÔNG persist

MemoryStore._cleanup_old_sessions:
  Có hàm cleanup NHƯNG KHÔNG được gọi tự động
```

### 4.2 JWT Token Passing — Security Anti-Pattern

```
Frontend → Authorization header → Gateway → /chat
         ↓
main.py extract → memory.collected_data["_token"]
         ↓
skill_executor._tool_calling_loop
         ↓
task += f"[SYSTEM: JWT_TOKEN={token}]"  ← ⚠️ Plain text in logs
         ↓
OrderProcessingSkill: JWT_TOKEN_PATTERN = re.compile(r"\[SYSTEM:\s*JWT_TOKEN=([^\]]+)\]")
         ↓
BackendClient.set_context_token(token) → Bearer header
```

---

## 5. Technical Debt & Điểm yếu

### Critical (Cần fix ngay)

| # | Issue | Location | Risk |
|---|-------|----------|------|
| C1 | **Dual-write inconsistency** | `chatStore.ts:36-92` | AI báo success nhưng cart API fail → user không biết. Chat hiển "added" nhưng cart trống |
| C2 | **JWT in plaintext message** | `routing_agent.py:160`, `order_processing.py:12` | Token visible in logs at DEBUG level → token leak |
| C3 | **In-memory memory only** | `conversation.py`, `memory.py` | Container restart = mất toàn bộ conversation |
| C4 | **Search tool duplicated** | `OrderProcessingSkill` vs `OrderWithSearchSkill` | 2 cách search khác nhau, unclear khi nào dùng cái nào |
| C5 | **No auth at gateway** | `SecurityConfig.java:35` | Tất cả API public — ai cũng tạo orders với bất kỳ user_id nào |

### High Priority

| # | Issue | Location | Risk |
|---|-------|----------|------|
| H1 | **No retry/circuit breaker** | Orchestrator + workers | Service down → cascading failure |
| H2 | **Frontend hardcodes action strings** | `chatStore.ts:42-51` | Magic strings `"add_to_cart"` — break nếu AI đổi format |
| H3 | **Order Agent có 2 mutually exclusive skills** | `agent.py:35-48` | `ORDER_AGENT_USE_A2A_SEARCH` flag không clear |
| H4 | **No observability** | Toàn AI layer | Không tracing (OpenTelemetry), không metrics, không structured logging đầy đủ |
| H5 | **ES là single point of failure** | `product-service` | ES down → semantic search fail → toàn bộ product discovery fail |

### Medium Priority

| # | Issue | Location | Risk |
|---|-------|----------|------|
| M1 | **Multiple LLM calls/request** | Orchestrator + workers | ~2-4 LLM calls/request × 100 users = cost explosion |
| M2 | **No streaming response** | `AgentCapabilities(streaming=False)` | User chờ "Thinking..." 4-8s rồi nhận full response |
| M3 | **Cleanup sessions never called** | `MemoryStore.cleanup_old_sessions` | Memory leak theo thời gian |
| M4 | **`context` variable undefined in scope** | `order_processing.py:159` | `context.get("user_id")` — `context` not defined |
| M5 | **ES sync is synchronous** | `ElasticsearchSyncService` | Sync đồng bộ — ES chậm → product save chậm |

---

## 6. Khả năng mở rộng

### Thêm agent mới

**Rất dễ** — chỉ cần:
1. Tạo `services/new_agent/{agent,main}.py` theo pattern
2. Register skill với `BaseAgent`
3. Thêm vào `docker-compose.yml`
4. Thêm agent card URL vào orchestrator env vars
5. **Không sửa code hiện có**

### Thay model

**Trung bình** — đổi `LLMSettings` trong `.env` hoặc `main.py` khởi tạo. Mỗi agent có own LLM config riêng.

### Scale user

**Khó** — cần:
- Redis-backed session (hiện tại in-memory → session lost across replicas)
- A2A load balancing (worker agents là single instance)
- PostgreSQL connection pooling tuning
- ES replicas

---

## 7. Đề xuất cải thiện

### Quick Wins (1-2 days)

**Q1: Fix dual-write inconsistency**
```typescript
// Option A: FE không tự gọi cart, chỉ hiển thị suggestion + button "Add to cart"
// AI trả product_ids → FE render button → user click → gọi cart API
// Option B: Gọi cart API rồi mới hiển thị "added" (pessimistic)
```

**Q2: Move JWT token sang header**
```python
# Thay vì: task += f"[SYSTEM: JWT_TOKEN={token}]"
# Worker agent đọc từ HTTP headers: X-Auth-Token
```

**Q3: Enable session cleanup**
```python
# Trong orchestrator lifespan hoặc background task
async def periodic_cleanup():
    while True:
        await asyncio.sleep(3600)
        _memory_store.cleanup_old_sessions(max_age_seconds=3600)
```

### Medium Refactor (1-2 weeks)

**M1: Add auth middleware at gateway**
```java
// Thay permitAll() bằng JWT validation
.requestMatchers("/api/cart/**", "/api/orders/**").hasAuthority("ROLE_USER")
```

**M2: Unified search qua A2A**
```python
# OrderProcessingSkill nên dùng A2ASearchClient như OrderWithSearchSkill
# Không gọi BackendClient.vector_search trực tiếp
```

**M3: Add Redis for session persistence**
```python
# Thay _memory_store = {} bằng Redis
# Cho phép scale orchestrator ra multiple replicas
```

**M4: Add streaming response (SSE)**
```python
# GET /api/ai/chat/stream
# SSE: agent_started → tool_calls → tool_results → agent_completed
# Frontend: render chunks real-time
```

**M5: Add OpenTelemetry tracing**
```python
# trace_id từ FE → Gateway → Orchestrator → Worker → Backend
```

### Long-term Architecture (1-2 months)

**L1: Deterministic routing thay vì LLM routing**
```
Compute embedding của user message
Compare với pre-computed skill embeddings
Route thẳng → O(1) decision
→ Giảm latency ~1-2s/request
```

**L2: Persistent conversation storage**
```sql
CREATE TABLE conversation_messages (
    id BIGSERIAL,
    user_id BIGINT,
    role VARCHAR(20),
    content TEXT,
    products JSONB,
    agent_used VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW()
);
```

**L3: Circuit breaker + retry**
```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=1, max=10))
async def _send_to_agent_with_retry(...):
    ...
```

---

## 8. Kết luận

### Level hiện tại: **Early Production** (không phải prototype, chưa production-ready)

```
Prototype    ←───│───────────│───────────→  Enterprise
                │           │
            Current      Target
             Status    (6-12 tháng)
```

### Điểm mạnh thực sự:

1. **Clean agent architecture** — BaseAgent + Skill pattern rất dễ extend
2. **Google A2A SDK** — đúng chuẩn industry cho multi-agent communication
3. **Separation of concerns** — FE chỉ giao tiếp 1 endpoint, không biết gì về agents
4. **Multi-mode planning** — đủ linh hoạt cho use cases hiện tại
5. **Microservices backend** — services độc lập

### Fix trước khi production:

```
Priority 1: C1 (Dual-write) + C5 (No auth) + C3 (In-memory memory)
            → Security + reliability

Priority 2: H4 (No observability) + H1 (No retry)
            → Cannot debug production

Priority 3: C4 (Duplicate search) + M1 (Multiple LLM calls)
            → Code quality + cost
```

### Scale path:

- **~100-500 concurrent users**: Hiện tại + Redis session + auth middleware
- **500-2000**: Multiple orchestrator replicas + Redis + ES replicas + connection pooling
- **2000+**: Thêm message queue (Kafka/RabbitMQ), event-driven architecture

---

## Unresolved Questions

1. **VNPay webhook**: Ai xử lý VNPay return/callback? Không thấy endpoint `/vnpay-return`
2. **JWT TTL**: Token expiration time là bao lâu? Không thấy config
3. **ES seed data**: Initial product data từ đâu? `ElasticsearchSyncService` chỉ sync khi có save
4. **Rate limiting**: Không có rate limit trên any API — AI orchestrator có thể bị abuse
