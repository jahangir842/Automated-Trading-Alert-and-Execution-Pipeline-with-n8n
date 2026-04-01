# Automated Trading Alert & Execution Pipeline

A robust, event-driven trading pipeline built with n8n. This system is designed to ingest JSON webhook payloads from trading algorithms (such as TradingView), route signals dynamically based on conditional logic, and securely execute trades via exchange APIs (e.g., Bitunix) using cryptographic signatures. 

## 🏗 Architecture Overview
* **Host OS:** Ubuntu
* **Automation Engine:** n8n
* **Database Backend:** PostgreSQL (ensures reliable, persistent execution logging)
* **Deployment:** Docker Compose (Infrastructure as Code)
* **Pipeline Flow:** Webhook (Ingestion) ➔ Switch (Router) ➔ Code/HTTP (Execution) ➔ Telegram (Notification)

---

## 🚀 Phase 1: Infrastructure Setup

To ensure secure secrets management and persistent data storage, the environment is deployed using Docker Compose with an attached PostgreSQL database.

### 1. Environment Configuration
Secrets and database credentials are strictly kept out of version control. Create a `.env` file in the root directory:

```env
# Database Configuration
POSTGRES_USER=n8n_db_user
POSTGRES_PASSWORD=your_secure_db_password
POSTGRES_DB=n8n

# n8n Configuration
N8N_ENCRYPTION_KEY=your_random_secure_encryption_key
GENERIC_TIMEZONE=Asia/Karachi
```

### 2. Docker Compose (`docker-compose.yml`)
This configuration links the PostgreSQL database and the n8n application via an isolated internal Docker network, utilizing dynamic variable substitution for security.

```yaml

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n_app
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - trading_net
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    container_name: n8n_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - trading_net
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -h localhost -U ${POSTGRES_USER} -d ${POSTGRES_DB}']
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  n8n_data:
  postgres_data:

networks:
  trading_net:

```

**Deployment Command:**
```bash
docker compose up -d
```

---

## ⚙️ Phase 2: Workflow Logic

The core event-driven architecture processes signals in four distinct stages. The UI is accessible at `http://localhost:5678`.

### 1. Ingestion (Webhook Node)
The entry point is a POST Webhook secured with header authentication to prevent unauthorized execution. It listens for incoming JSON signals.

**Expected Payload:**
```json
{
  "action": "buy",
  "asset": "BTCUSDT",
  "price": 65000,
  "size": 0.01
}
```

**Local Testing:**
Simulate a trading signal directly from the Ubuntu terminal:
```bash
curl -X POST http://localhost:5678/webhook-test/trading-signal \
     -H "Content-Type: application/json" \
     -d '{"action": "buy", "asset": "BTCUSDT", "price": 65000, "size": 0.01}'
```

### 2. Routing (Switch Node)
To handle complex trading strategies, the payload routes through a Switch node evaluating the `$json.body.action` parameter. Sub-10ms execution times ensure minimal slippage.
* **Route 1:** Evaluates if `action` is equal to `buy`.
* **Route 2:** Evaluates if `action` is equal to `sell`.

### 3. Execution (Code & HTTP Nodes)
Before contacting the exchange API, a custom Code node generates an HMAC SHA256 cryptographic signature using the exchange's API Secret. The HTTP Request node then pushes the formatted, signed POST request to the exchange (e.g., Bitunix API).

### 4. Notification (Telegram Integration)
A final state-check node alerts the administrator via Telegram. 
* **Success:** 🟢 *Executed BUY on BTCUSDT at $65,000.*
* **Failure:** 🔴 *FAILED to execute BUY. API Error: [Error Details].*

---

## 🧪 Testing & Deployment Strategy

1. **Local Mocking:** Utilize `curl` or Postman to fire test JSON payloads and verify routing logic.
2. **Paper Trading:** Point the HTTP Request node to an exchange Testnet API to validate cryptographic signatures without risking capital.
3. **Live Deployment:** Expose the local n8n instance via Cloudflare Tunnels or Ngrok to accept live webhooks from TradingView alerts.

## 📥 Importing this Project
To replicate this workflow, deploy the infrastructure using the instructions in Phase 1, then import the `workflow.json` file provided in this repository directly into your n8n canvas.
