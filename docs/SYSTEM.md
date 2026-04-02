# ToRoMe Store

Hệ thống e-commerce thời trang với AI assistant, gồm 3 thành phần chạy riêng biệt.

## Kiến trúc

```
Frontend (:3000)  ──→  Backend (:9000)  ──→  AI Orchestrator (:8000)
   React + Vite         Spring Boot              FastAPI + LangGraph
                         PostgreSQL                  │
                         Gateway /api/ai/*           ├── Search Agent (:8001)
                                                     ├── Stylist Agent (:8002)
                                                     └── Order Agent (:8003)
```

- **Frontend** gọi duy nhất tới **Backend** (port 9000)
- **Backend** xử lý trực tiếp product/user/order/payment, proxy `/api/ai/*` sang **AI Orchestrator**
- **AI Orchestrator** điều phối 3 agent qua Google A2A SDK, các agent gọi ngược lại Backend để lấy dữ liệu

## Yêu cầu

- Docker & Docker Compose
- Node.js 18+ (cho frontend)
- OpenRouter API key (cho Stylist Agent — dùng model miễn phí)

## Cấu trúc thư mục

```
ToRoMe Store/
├── backend/          Spring Boot + PostgreSQL (Docker Compose)
├── ai/               FastAPI + LangGraph + 3 Agents (Docker Compose)
├── frontend/         React + Vite + TailwindCSS (npm)
└── SYSTEM.md         File này
```

## Chạy hệ thống

### Bước 1: Backend (Terminal 1)

```bash
cd backend
docker compose up --build
```

Chờ tới khi thấy log `Started StoreApplication`. Backend chạy ở `http://localhost:9000`.

Gồm 2 container:
- `postgres` — PostgreSQL 16, port 5432
- `backend` — Spring Boot, port 9000

### Bước 2: AI Layer (Terminal 2)

```bash
cd ai
# Sửa OPENAI_API_KEY trong file .env trước khi chạy
docker compose up --build
```

Chờ tới khi tất cả agent healthy. Orchestrator chạy ở `http://localhost:8000`.

Gồm 4 container:
- `search-agent` — port 8001
- `stylist-agent` — port 8002 (cần OPENAI_API_KEY từ OpenRouter)
- `order-agent` — port 8003
- `orchestrator` — port 8000

### Bước 3: Frontend (Terminal 3)

```bash
cd frontend
npm install
npm run dev
```

Frontend chạy ở `http://localhost:3000`.

## Tài khoản test

| Username | Password | Preferences |
|----------|----------|-------------|
| demo_user | demo123 | size M, black, minimal |
| fashion_lover | demo123 | size L, navy, smart-casual |

## API Endpoints

### Backend (port 9000)

| Method | Path | Mô tả |
|--------|------|--------|
| GET | `/api/products` | Danh sách sản phẩm |
| GET | `/api/products/{id}` | Chi tiết sản phẩm |
| POST | `/api/products/vector-search` | Tìm kiếm sản phẩm |
| POST | `/api/auth/login` | Đăng nhập |
| GET | `/api/users/{id}` | Thông tin user |
| PATCH | `/api/users/profile` | Cập nhật preferences |
| POST | `/api/orders/auto-create` | Tạo đơn hàng |
| GET | `/api/orders/{id}` | Chi tiết đơn hàng |
| GET | `/api/payments/vnpay-gen?orderId=` | Tạo link thanh toán VNPay |
| POST | `/api/ai/chat` | Chat với AI (proxy → orchestrator) |
| GET | `/api/ai/conversation/{userId}` | Lịch sử hội thoại |
| GET | `/health` | Health check |

## Luồng hoạt động

### Product Flow (mua hàng)
```
Frontend → GET /api/products → Hiển thị danh sách
Frontend → GET /api/products/{id} → Xem chi tiết, thêm vào giỏ
Frontend → POST /api/auth/login → Đăng nhập
Frontend → POST /api/orders/auto-create → Tạo đơn
Frontend → GET /api/payments/vnpay-gen → Lấy link thanh toán
```

### AI Chat Flow
```
Frontend → POST /api/ai/chat → Backend proxy → Orchestrator
Orchestrator → classify intent → route to agent
Agent → gọi Backend API (products/users/orders) → trả kết quả
Orchestrator → trả response cho Frontend
```

## Env files

| File | Mô tả |
|------|--------|
| `backend/.env` | DB connection, JWT secret, AI orchestrator URL |
| `ai/.env` | OPENAI_API_KEY, agent ports, backend URL |
| `frontend/.env` | VITE_BACKEND_URL, service ports |

## Dừng hệ thống

```bash
# Terminal 1
cd backend && docker compose down

# Terminal 2
cd ai && docker compose down

# Terminal 3
Ctrl+C
```

## Tech Stack

| Thành phần | Công nghệ |
|------------|-----------|
| Frontend | React 19, TypeScript, Vite, TailwindCSS 4, Zustand |
| Backend | Spring Boot 3.4, Java 21, PostgreSQL 16, Flyway, JWT |
| AI Layer | Python, FastAPI, LangGraph, Google A2A SDK, OpenRouter |
| Infra | Docker Compose, Nginx (production) |
