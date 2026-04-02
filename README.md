# ToRoMe Store — AI-Powered Vietnamese Fashion E-Commerce

A modern Vietnamese fashion e-commerce platform featuring an AI shopping assistant built with a multi-agent system. Three independent layers communicate via REST APIs: a React + Vite frontend, a Spring Boot microservices backend, and a FastAPI + LangGraph AI layer.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Frontend (:3000)                                                │
│  React 19 · Vite · TailwindCSS 4 · Zustand                      │
│  AI Chat Widget · Product Grid · Shopping Cart · VNPay Checkout │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTP / REST
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  Backend (:9000)                                                 │
│  Spring Boot 3.4 · PostgreSQL (3 DBs) · Elasticsearch            │
│  JWT Auth · Flyway Migrations · API Gateway                      │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │ user-service │  │product-svc   │  │ order-svc    │            │
│  │   (:9001)    │  │  (:9002)     │  │  (:9003)     │            │
│  │  Auth/JWT    │  │  Products    │  │  Cart/Orders │            │
│  │  Profiles    │  │  Vector Search│  │  Checkout    │            │
│  └──────────────┘  │  + ES sync    │  └──────────────┘            │
│                    └──────────────┘                               │
│                                                                  │
│  ┌──────────────┐           ┌──────────────────────────────┐     │
│  │payment-svc   │           │  AI Layer (:8000)             │     │
│  │  (:9004)     │           │  FastAPI · LangGraph · A2A    │     │
│  │  VNPay       │◄──────────│  Multi-Agent Orchestrator     │     │
│  └──────────────┘           │                                │     │
│                             │  ┌──────────┐ ┌──────────┐    │     │
│                             │  │ Search   │ │ Stylist   │    │     │
│                             │  │ Agent    │ │ Agent     │    │     │
│                             │  │ (:8001)  │ │ (:8002)   │    │     │
│                             │  └──────────┘ └──────────┘    │     │
│                             │  ┌──────────────────┐         │     │
│                             │  │ Order Agent       │         │     │
│                             │  │ (:8003)          │         │     │
│                             │  └──────────────────┘         │     │
│                             └────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### Data Flow

- **Product browsing**: Frontend → `GET /api/products` → Gateway → product-service → PostgreSQL / Elasticsearch
- **Auth**: Frontend → `POST /api/auth/login` → Gateway → user-service → JWT issued
- **Cart & Orders**: Frontend → `/api/cart`, `/api/orders` → Gateway → order-service → PostgreSQL
- **AI Chat**: Frontend → `POST /api/ai/chat` → Gateway (strips `/api`) → Orchestrator → Worker Agents (A2A) → Backend APIs
- **Payment**: Frontend → `GET /api/payments/vnpay-gen` → Gateway → payment-service → VNPay sandbox URL

---

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Frontend | React 19, TypeScript, Vite 6 | TailwindCSS 4, Zustand 5 |
| Backend | Spring Boot 3.4, Java 21 | PostgreSQL 16, Elasticsearch 8.17 |
| AI Layer | Python 3.13, FastAPI | LangGraph, Google A2A SDK |
| Gateway | Spring Cloud Gateway | JWT (jjwt 0.12.6) |
| Auth | Stateless JWT | BCrypt password hashing |
| Payment | VNPay sandbox | Docker Compose |

---

## Project Structure

```
ToRoMe Store/
├── fashion-ai-frontend/          # React 19 + Vite + TailwindCSS 4
│   ├── src/
│   │   ├── api/                  # Axios API clients (auth, products, cart, orders, chat)
│   │   ├── components/           # UI: cart, chat, layout, product
│   │   ├── pages/                # HomePage, ProductsPage, ProductDetailPage, CartPage, LoginPage
│   │   ├── stores/               # Zustand: authStore, cartStore, chatStore
│   │   └── types/                # TypeScript interfaces
│   ├── docs/                     # API docs, component docs, state management docs
│   └── Dockerfile                # Multi-stage: node → nginx
│
├── fashion-ai-backend/            # Spring Boot microservices
│   ├── common/                   # Shared: JwtService, JwtAuthFilter, SecurityConfig, GlobalExceptionHandler
│   ├── gateway-service/           # Spring Cloud Gateway (:9000), routes all traffic
│   ├── user-service/              # Auth, user profiles, preferences (:9001)
│   ├── product-service/           # Product catalog, Elasticsearch sync (:9002)
│   ├── order-service/             # Shopping cart, order creation (:9003)
│   ├── payment-service/           # VNPay payment link generation (:9004)
│   ├── infra/                     # DB Docker Compose, shared env
│   └── docs/                      # Service-level documentation
│
├── fashion-ai-agent/              # FastAPI multi-agent AI layer
│   ├── services/
│   │   ├── orchestrator/          # Intent routing, planning, conversation memory (:8000)
│   │   ├── search/                # Semantic product search (:8001)
│   │   ├── stylist/               # Outfit recommendations (:8002)
│   │   └── order/                 # Cart & order processing (:8003)
│   ├── shared/
│   │   ├── backend_client.py      # Async httpx client for Spring Boot APIs
│   │   ├── config.py              # Pydantic settings from env
│   │   └── base_agent/            # BaseAgent, Skill, Tool, Executor, Memory
│   └── docs/                      # Agent architecture docs, A2A protocol
│
└── README.md                      # This file
```

---

## Features

### Customer-facing
- Product browsing with responsive grid (2–4 columns)
- Semantic search via Elasticsearch vector search
- Size/color/style filters
- Persistent shopping cart (stored in database, survives browser close)
- VNPay sandbox checkout
- User registration and login with JWT auth
- User preference profile (size, color, style)

### AI Shopping Assistant
- Floating chat widget on all pages
- Natural language product search ("find me a navy blazer")
- Outfit recommendations based on user preferences and occasion
- Cart operations via chat ("add this to cart", "add items 1, 3, 5")
- Order creation and payment link generation
- Conversation memory (last 3 pairs + summary of older history)
- Multi-agent coordination: orchestrator routes to search/stylist/order agents

---

## Quick Start

### Prerequisites
- Docker & Docker Compose
- Node.js 18+
- OpenRouter API key (for Stylist Agent)

### Step 1 — Backend (Terminal 1)

```bash
cd fashion-ai-backend
docker compose up --build
```
Waits for `Started StoreApplication`. Services: gateway (:9000), 4 microservices, 3 PostgreSQL, 1 Elasticsearch.

### Step 2 — AI Layer (Terminal 2)

```bash
cd fashion-ai-agent
# Configure OPENAI_API_KEY in .env first
docker compose up --build
```
Services: search-agent (8001), stylist-agent (8002), order-agent (8003), orchestrator (8000).

### Step 3 — Frontend (Terminal 3)

```bash
cd fashion-ai-frontend
npm install
npm run dev
```

Open http://localhost:3000

---

## Environment Configuration

### Backend (`fashion-ai-backend/.env`)

| Variable | Description |
|----------|-------------|
| `POSTGRES_USER/PASSWORD` | DB credentials |
| `JWT_SECRET` | HMAC-SHA256 signing key (min 256 bits) |
| `AI_BASE_URL` | AI orchestrator URL (default `http://host.docker.internal:8000`) |

### AI Layer (`fashion-ai-agent/.env`)

| Variable | Description |
|----------|-------------|
| `OPENAI_API_KEY` | OpenRouter API key |
| `BACKEND_BASE_URL` | Backend gateway URL (default `http://host.docker.internal:9000`) |
| `ORCHESTRATOR_MAX_CONVERSATION_HISTORY` | Memory size (default 20) |
| `ORCHESTRATOR_ENABLE_MULTI_AGENT` | Enable multi-agent planning (default true) |

### Frontend (`fashion-ai-frontend/.env`)

| Variable | Description |
|----------|-------------|
| `VITE_BACKEND_URL` | Backend gateway URL (default `http://localhost:9000`) |
| `VITE_ORCHES_URL` | AI orchestrator URL (default `http://localhost:8000`) |

---

## Test Accounts

| Username | Password | Preferences |
|----------|----------|-------------|
| `demo_user` | `demo123` | Size M, black, minimal |
| `fashion_lover` | `demo123` | Size L, navy, smart-casual |

---

## API Endpoints

### Gateway (port 9000)

| Method | Path | Service | Auth | Description |
|--------|------|---------|------|-------------|
| `POST` | `/api/auth/register` | user | No | Register new user |
| `POST` | `/api/auth/login` | user | No | Login, returns JWT |
| `GET` | `/api/users/{id}` | user | JWT | Get user profile |
| `PATCH` | `/api/users/profile` | user | JWT | Update preferences |
| `GET` | `/api/products` | product | No | List all products |
| `GET` | `/api/products/{id}` | product | No | Product details |
| `POST` | `/api/products/vector-search` | product | No | Semantic search |
| `POST` | `/api/products/batch` | product | No | Bulk fetch by IDs |
| `GET` | `/api/cart` | order | No | Get cart by userId |
| `POST` | `/api/cart/items` | order | No | Add to cart |
| `PUT` | `/api/cart/items/{id}` | order | No | Update quantity/size |
| `DELETE` | `/api/cart/items/{id}` | order | No | Remove item |
| `DELETE` | `/api/cart` | order | No | Clear cart |
| `POST` | `/api/orders` | order | No | Create order |
| `POST` | `/api/orders/auto-create` | order | No | Create + VNPay ref |
| `POST` | `/api/orders/checkout` | order | No | Cart → Order + VNPay |
| `GET` | `/api/orders/{id}` | order | No | Get order |
| `GET` | `/api/payments/vnpay-gen` | payment | No | Generate VNPay URL |
| `POST` | `/api/ai/chat` | gateway → AI | No | Chat with AI assistant |
| `GET` | `/health` | gateway | No | Health check |

---

## Key Design Decisions

### Multi-Agent Architecture
The AI layer uses Google A2A SDK for agent-to-agent communication. The Orchestrator receives user messages, classifies intent, and routes to specialized agents (Search, Stylist, Order). Each agent has its own skill definitions and tool sets.

### Database-per-Service
Three separate PostgreSQL instances — no cross-DB foreign keys. Application-level referential integrity enforced via `RestClient` calls between services.

### Vector Search with Fallback
product-service tries Elasticsearch first for semantic search, then falls back to PostgreSQL `ILIKE` keyword search, then falls back to `findAll` if both fail.

### Persistent Cart
Shopping cart is stored in `cart_items` table (not session or localStorage), so it survives browser closes and device changes.

### AI Chat Bridge
The AI orchestrator returns `action` fields (`add_to_cart`, `add_multiple_to_cart`) in responses. The frontend `chatStore.sendMessage()` parses these and directly calls `cartStore.addItem()`, bridging the AI and cart domains.

---

## Documentation

Detailed docs for each layer:

- **Frontend** (`fashion-ai-frontend/docs/`)
  - `README.md` — Project overview, tech stack, routing, features
  - `api-clients.md` — API modules (auth, products, cart, orders, chat)
  - `state-management.md` — Zustand stores (auth, cart, chat)
  - `components-pages.md` — All pages and components

- **Backend** (`fashion-ai-backend/docs/`)
  - `README.md` — Architecture, quick start
  - `gateway-service.md` — API Gateway routing, CORS
  - `user-service.md` — Auth, JWT, user profile
  - `product-service.md` — Product CRUD, Elasticsearch
  - `order-service.md` — Shopping cart, order creation
  - `payment-service.md` — VNPay integration
  - `common.md` — JWT, exceptions, security configs

- **AI Layer** (`fashion-ai-agent/docs/`)
  - `README.md` — Agent system overview
  - `orchestrator.md` — Intent classification & routing
  - `search-agent.md` — Semantic product search
  - `stylist-agent.md` — Outfit recommendations
  - `order-agent.md` — Order processing agent
  - `shared-components.md` — Backend client, models, skills system

---

## Ports Summary

| Service | Port | Description |
|---------|------|-------------|
| Frontend | 3000 | React + Vite dev server |
| Gateway | 9000 | Spring Cloud Gateway — single entry |
| User Service | 9001 | Auth, profiles |
| Product Service | 9002 | Products, Elasticsearch |
| Order Service | 9003 | Cart, orders |
| Payment Service | 9004 | VNPay |
| Elasticsearch | 9200 | Vector search index |
| Orchestrator | 8000 | AI entry point |
| Search Agent | 8001 | Product search |
| Stylist Agent | 8002 | Outfit recommendation |
| Order Agent | 8003 | Cart & orders |
| PostgreSQL (user) | 5433 | User database |
| PostgreSQL (product) | 5436 | Product database |
| PostgreSQL (order) | 5437 | Order database |
