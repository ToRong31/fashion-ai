# 🚀 AGENTIC E-COMMERCE BLUEPRINT: FASHION STORE (MICROSERVICE + MULTI-AI AGENTS)

Blueprint này chia hệ thống thành **3 repository độc lập** nhằm đảm bảo khả năng mở rộng, tách biệt trách nhiệm và dễ deploy.

1. **Frontend Repository** – giao diện người dùng
2. **Backend Repository** – business microservices
3. **AI Repository** – hệ thống AI Agents hoạt động độc lập

---

# 1. SYSTEM ARCHITECTURE

Hệ thống gồm ba tầng chính:

**Frontend Layer**

* React + TypeScript
* TailwindCSS
* Chat UI để giao tiếp với AI Assistant
* Gọi API tới Backend và AI Orchestrator

**Backend Layer (Business Services)**

Các microservice Spring Boot xử lý nghiệp vụ:

* User management
* Product catalog
* Order management
* Payment integration

Backend đóng vai trò **nguồn dữ liệu chính (Source of Truth)** cho toàn bộ hệ thống.

**AI Layer**

Tầng AI gồm nhiều **Agent microservices độc lập**, được điều phối bởi một **Orchestrator sử dụng LangGraph**.

Các agent giao tiếp với nhau thông qua **Google A2A SDK**.

---

# 2. FRONTEND REPOSITORY

Frontend chịu trách nhiệm:

* hiển thị sản phẩm
* quản lý giỏ hàng
* giao tiếp với AI Assistant
* gửi request tới Backend API

Frontend có hai luồng chính:

**Product Flow**

User → Frontend → Backend API → Database

**AI Assistant Flow**

User → Frontend → AI Orchestrator → AI Agents → Backend Services

---

# 3. BACKEND REPOSITORY (SPRING BOOT MICROSERVICES)

Backend được thiết kế theo **microservice architecture**.

Các service chính gồm:

### User Service

Quản lý:

* đăng ký
* đăng nhập
* hồ sơ người dùng
* preferences (long-term memory cho AI)

### Product Service

Quản lý:

* danh mục sản phẩm
* metadata sản phẩm
* vector search cho semantic search

### Order Service

Quản lý:

* tạo đơn hàng
* trạng thái đơn
* auto-order từ AI

### Payment Service

Tích hợp thanh toán:

* VNPay
* tạo payment link
* xác nhận thanh toán

---

# 4. DATABASE DESIGN (POSTGRES + PGVECTOR)

Database sử dụng **PostgreSQL** với extension **pgvector** để hỗ trợ semantic search.

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    password_hash TEXT,
    preferences JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    price DECIMAL(10,2),
    stock_quantity INT,
    embedding vector(1536),
    metadata JSONB
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    status VARCHAR(20),
    vnpay_ref VARCHAR(100),
    total_amount DECIMAL(10,2)
);
```

---

# 5. BACKEND APIs

Các AI Agents sẽ sử dụng các API backend làm **Tools**.

### Product APIs

```
POST /api/products/vector-search
GET /api/products/{id}
GET /api/products
```

### User APIs

```
POST /api/auth/login
GET /api/users/{id}
PATCH /api/users/profile
```

### Order APIs

```
POST /api/orders
POST /api/orders/auto-create
GET /api/orders/{id}
```

### Payment APIs

```
GET /api/payments/vnpay-gen?orderId=
```

---

# 6. AI SYSTEM OVERVIEW

AI Layer được thiết kế theo **multi-agent architecture**.

Các thành phần chính:

**AI Orchestrator**

* FastAPI
* LangGraph
* quyết định agent nào sẽ xử lý yêu cầu
* quản lý global state

**Search Agent**

Chịu trách nhiệm:

* semantic product search
* truy vấn vector database

**Stylist Agent**

Chịu trách nhiệm:

* đề xuất outfit
* kết hợp nhiều sản phẩm
* sử dụng metadata và user preferences

**Order Agent**

Chịu trách nhiệm:

* tạo đơn hàng
* gọi payment service
* trả về payment link

Tất cả agent **không truy cập trực tiếp vào conversation history**.

---

# 7. AI COMMUNICATION (GOOGLE A2A SDK)

Tất cả giao tiếp giữa các agent sử dụng **MessagePacket**.

Nguyên tắc:

* Orchestrator gửi request
* Worker agent xử lý
* Worker trả response

Workers **không đọc state toàn cục**.

---

# 8. ORCHESTRATOR STATE

LangGraph quản lý trạng thái chung.

```python
from typing import TypedDict, Dict

class MasterState(TypedDict):
    raw_user_input: str
    user_id: str
    context_to_send: Dict
    next_node: str
    final_response: str
```

---

# 9. ORCHESTRATOR LOGIC

Orchestrator phân tích prompt và chọn agent phù hợp.

```python
from langgraph.graph import StateGraph, END
from google.a2a import AgentClient, MessagePacket

search_client = AgentClient(agent_id="search_agent")
stylist_client = AgentClient(agent_id="stylist_agent")
order_client = AgentClient(agent_id="order_agent")

def orchestrator_node(state: MasterState):

    text = state["raw_user_input"].lower()

    if "buy" in text:
        return {
            "next_node": "order_node",
            "context_to_send": {"query": text}
        }

    if "style" in text:
        return {
            "next_node": "stylist_node",
            "context_to_send": {"query": text}
        }

    return {
        "next_node": "search_node",
        "context_to_send": {"query": text}
    }
```

---

# 10. SEARCH AGENT LOGIC

Search agent thực hiện semantic search.

```python
@app.post("/agent/search")

def search_products(query):

    results = backend.vector_search(query)

    return results
```

Agent sử dụng backend tool:

```
POST /api/products/vector-search
```

---

# 11. STYLIST AGENT LOGIC

Stylist agent phân tích:

* user prompt
* product metadata
* user preferences

Ví dụ:

User prompt:

```
"I need an outfit for winter meeting"
```

Agent trả về:

* blazer
* shirt
* trousers
* shoes

---

# 12. ORDER AGENT LOGIC

Order agent thực hiện:

1. xác định sản phẩm cần mua
2. tạo order
3. trả payment link

Tools sử dụng:

```
POST /api/orders/auto-create
GET /api/payments/vnpay-gen
```

---

# 13. MEMORY DESIGN

Long-term memory được lưu trong database.

Cột:

```
users.preferences
```

Ví dụ:

```
{
 "size": "XL",
 "favorite_color": "black",
 "style": "minimal"
}
```

AI có thể cập nhật memory bằng:

```
PATCH /api/users/profile
```

---

# 14. ISOLATED MEMORY RULE

Worker agents **không được truy cập conversation history**.

Chỉ nhận dữ liệu đã được lọc:

```
context_to_send
```

Ví dụ:

```
{
 "query": "black winter jacket"
}
```

Điều này đảm bảo:

* isolation
* bảo mật
* giảm context size

---

# 15. END-TO-END FLOW

Ví dụ user prompt:

```
"I want a black jacket for winter"
```

Luồng xử lý:

User → Frontend → AI Orchestrator → Search Agent → Product Service → Response

---

Ví dụ mua hàng:

```
"I want to buy this jacket"
```

Luồng:

User → Frontend → Orchestrator → Order Agent → Order Service → Payment Service → VNPay Link

---

# 16. EXTENSIBILITY

Kiến trúc này dễ mở rộng.

Có thể thêm agent mới như:

* Recommendation Agent
* Inventory Agent
* Trend Analysis Agent
* Marketing Agent

Chỉ cần đăng ký agent mới với Orchestrator mà **không cần thay đổi backend**.
s