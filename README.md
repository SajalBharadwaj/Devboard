# 📊 DevBoard — Advanced (UI + Go + Postgres)

🚀 **DevBoard** is a fully containerized, production-ready full-stack developer dashboard application. This project serves as an architectural blueprint demonstrating how a modern **React (Vite)** frontend, a high-performance **Go API** backend, and a **PostgreSQL** relational database seamlessly interconnect and communicate via a dedicated Docker network layer.

---

## 🏗️ System Architecture & Data Flow

Every component is modularized and operates in isolation, orchestrating data traffic through structured port forwards and proxies:

```text
 🌐 Browser ──[Port 8080]──> ⚛️ Frontend (React/Vite)
                                   │
                           (Proxies /api requests)
                                   ▼
 🚀 Backend (Go API) ──[Internal Network]──> 🐘 Database (Postgres)
```

1. **Frontend:** A optimized React application that serves the user interface and proxies all inbound traffic prefixed with `/api` directly to the backend layer.
2. **Backend:** A lightweight, high-concurrency Go application responsible for evaluating business logic and executing database CRUD queries.
3. **Database:** An official PostgreSQL container configured to automatically seed relational tables with mock schema and initial datasets upon cold start.

📌 *Note: Authentication mechanisms and AI components are omitted on purpose to isolate and highlight multi-container microservice networking.*

---

## 🛠️ Prerequisites

You do not need **Node.js, Go, or Postgres** installed on your host OS. Everything executes natively inside standard isolated environments. The only requirement is:
* **Docker** & **Docker Compose** (or Docker Desktop)

---

## 🧑‍💻 Part 1 — The Manual Routing Configuration (Deep Dive)

To understand container discovery and overlay networks, execute these commands step-by-step from your root `~/devboard` directory:

### Step 1: Initialize the Isolated Docker Network
Containers can only discover each other dynamically via service names if they are bound to the same software-defined network:
```bash
docker network create devboard-net
```

### Step 2: Compile Local Images
```bash
docker build -t devboard-frontend ./frontend
docker build -t devboard-backend ./backend
```

### Step 3: Launch the Database Layer
We explicit name this container `postgres` so the application can resolve its address over DNS. The host volume mount mounts data migrations instantly on initialization:
```bash
docker run -d --name postgres --network devboard-net \
  -e POSTGRES_USER=devboard \
  -e POSTGRES_PASSWORD=devboard \
  -e POSTGRES_DB=devboard \
  -v "$PWD/init/postgres":/docker-entrypoint-initdb.d:ro \
  -p 5432:5432 \
  postgres:16-alpine
```

### Step 4: Launch the API Backend Engine
```bash
docker run -d --name backend --network devboard-net \
  -e PORT=8080 \
  -e POSTGRES_URL="postgres://devboard:devboard@postgres:5432/devboard?sslmode=disable" \
  -p 8081:8080 \
  devboard-backend
```

### Step 5: Launch the UI Frontend Wrapper
The app listens on port `4173` inside the virtual ecosystem, which we expose out to host port `8080`:
```bash
docker run -d --name frontend --network devboard-net \
  -p 8080:4173 \
  devboard-frontend
```

### Step 6: Verify Network Topology
* **Frontend Dashboard UI:** Navigate to [http://localhost:8080](http://localhost:8080)
* **API Gateway Verification:** Execute `curl http://localhost:8081/health` (Expects HTTP 200 OK)

### Step 7: Teardown Infrastructure
```bash
docker rm -f frontend backend postgres
docker network rm devboard-net
```

---

## ⚡ Part 2 — Automated Orchestration: Docker Compose

Docker Compose streamlines production deployment by managing network graphs, target environment injections, and dependency boot order from a declarative `docker-compose.yml` manifest.

1. **Generate Environment Config (One-Time Task):**
   ```bash
   cp .env.example .env
   ```
2. **Bring the Full Infrastructure Up In Background (Detached):**
   ```bash
   docker compose up -d --build
   ```
3. **Graceful Stack Teardown:**
   ```bash
   docker compose down
   ```

---

## 🚀 Part 3 — Pipeline Shortcuts via Makefile

For environments supporting GNU Automation utilities, standard operations are aliased into clean, atomic scripts:

| Command | Operational Objective |
| :--- | :--- |
| `make` | Audits and prints all available project macros |
| `make setup` | Instantiates environment variables using target templates |
| `make up` | Automatically runs builds and initializes active components |
| `make down` | Terminates infrastructure and unbinds associated networks |
| `make logs` | Establishes a standard I/O stream tailing real-time errors |
| `make reset` | Purges persistent volumes and forces a data reset |
| `make smoke` | Executes health check assertions to verify stack status |

---

## 🔌 API Reference Map

The browser proxies requests through `/api/...`; the underlying Go runtime captures and processes them at root paths:

| Method | Endpoint Path | Functionality Target |
| :--- | :--- | :--- |
| **GET** | `/projects` | Fetches collection of current active projects |
| **POST** | `/projects` | Appends a new project structure to storage |
| **GET** | `/tasks?project_id=N` | Extracts specific array of tasks belonging to Project N |
| **POST** | `/tasks` | Commits a new task instance |
| **PATCH**| `/tasks/:id` | Modifies properties (e.g., status flags) of targeted task |
| **GET** | `/search?q=&project_id=N`| Executes query search filtering tasks by keywords |
| **GET** | `/health` | Heartbeat endpoint validating backend viability |

---

## 📁 Repository Layout Strategy

```text
.
├── docker-compose.yml   # Multi-container declaration mapping frontend, backend & postgres
├── Makefile             # Macro orchestrator (make up, make down, make reset)
├── .env.example         # Target properties blueprint tracked by Version Control
├── .env                 # Active localized configuration containing secrets (Git-ignored)
├── frontend/            # React + Vite application directory (Container Port: 4173)
├── backend/             # Compiled Golang source runtime along with its Dockerfile
└── init/postgres/       # Pre-built database migrations and initialization data seeds
```
