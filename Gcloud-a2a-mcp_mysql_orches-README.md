Gcloud setup


* **MySQL** runs locally on the VM via Docker Compose
* **MCP Server** (port `10110`) talks to MySQL with a **read-only** user
* **DB Tool Agent** (port `10111`) proxies MCP tools
* **Orchestrator (Planner)** (port `10112`) calls the Agent and an LLM (Vertex or Ollama)

---

# 0) One-time setup (folders & env)

```bash
mkdir -p ~/a2a-stack/{compose,mcp_server,a2a_agent,orchestrator_llm}
cd ~/a2a-stack
```

---

# 1) Start **MySQL** (Docker Compose)

**compose/docker-compose.yml**

```yaml
services:
  mysql:
    image: mysql:8.4
    container_name: mcp_mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: salesdb
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppass
    ports:
      - "3306:3306"
    volumes:
      - ./init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -prootpass || exit 1"]
      interval: 5s
      timeout: 3s
      retries: 30
```

**compose/init/01\_schema.sql**

```sql
CREATE TABLE IF NOT EXISTS customers (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(120) UNIQUE,
  city VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE IF NOT EXISTS orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  status ENUM('PENDING','PAID','CANCELLED') DEFAULT 'PENDING',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

**compose/init/02\_data.sql**

```sql
INSERT INTO customers (name, email, city) VALUES
('Asha Mehta','asha@example.com','Mumbai'),
('Ravi Kumar','ravi@example.com','Pune'),
('Neha Singh','neha@example.com','Delhi');

INSERT INTO orders (customer_id, amount, status) VALUES
(1, 1200.50, 'PAID'),
(2,  499.00, 'PENDING'),
(1,  849.90, 'PAID');
```

**compose/init/03\_readonly.sql**

```sql
CREATE USER IF NOT EXISTS 'readonly'@'%' IDENTIFIED BY 'readonlypass';
GRANT SELECT ON salesdb.* TO 'readonly'@'%';
FLUSH PRIVILEGES;
```

**Bring up MySQL**

```bash
cd ~/a2a-stack/compose
docker compose up -d
docker logs -f mcp_mysql   # wait until “ready for connections”
```

**Quick sanity**

```bash
docker exec -it mcp_mysql mysql -uroot -prootpass -e "SHOW DATABASES;"
docker exec -it mcp_mysql mysql -ureadonly -preadonlypass -e "USE salesdb; SHOW TABLES;"
```

---

# 2) Run **MCP Server** (→ MySQL) on :10110

**mcp\_server/server.py** (use the file you already have)

**mcp\_server/.env**

```
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=salesdb
DB_USER=readonly
DB_PASS=readonlypass
MCP_HOST=0.0.0.0
MCP_PORT=10110
```

**Start it**

```bash
cd ~/a2a-stack/mcp_server
python -m venv .venv && source .venv/bin/activate
pip install fastapi uvicorn pydantic mysql-connector-python python-dotenv
uvicorn server:app --host ${MCP_HOST:-0.0.0.0} --port ${MCP_PORT:-10110}
```

**Test MCP directly**

```bash
curl -s -X POST http://127.0.0.1:10110/mcp \
  -H 'Content-Type: application/json' \
  -d '{"id":"1","method":"db.list_tables","params":{}}' | jq
```

---

# 3) Run **DB Tool Agent** (→ MCP) on :10111

**a2a\_agent/agent.py** (use the file you already have)

**a2a\_agent/.env**

```
AGENT_HOST=0.0.0.0
AGENT_PORT=10111
MCP_URL=http://127.0.0.1:10110/mcp
```

**Start it**

```bash
cd ~/a2a-stack/a2a_agent
python -m venv .venv && source .venv/bin/activate
pip install fastapi uvicorn httpx python-dotenv
uvicorn agent:app --host ${AGENT_HOST:-0.0.0.0} --port ${AGENT_PORT:-10111}
```

**Test Agent → MCP**

```bash
curl -s -X POST http://127.0.0.1:10111/tools/invoke \
  -H 'Content-Type: application/json' \
  -d '{"tool_name":"mysql_list_tables","arguments":{}}' | jq
```

---

# 4) Run **Orchestrator (LLM Planner)** on :10112

**Pick ONE LLM path**:

### A) Vertex AI (Gemini) – API key (simple)

```bash
export LLM_PROVIDER=vertex
export GOOGLE_API_KEY="PASTE_API_KEY"
export VERTEX_MODEL="gemini-1.5-flash-002"
export VERTEX_LOCATION="us-central1"
```

### B) Ollama (local/remote)

```bash
export LLM_PROVIDER=ollama
export OLLAMA_BASE_URL=http://127.0.0.1:11434    # or http://<HOST_IP>:11434
export OLLAMA_MODEL=llama3.1:8b-instruct
```

**orchestrator\_llm/.env**

```
ORCH_HOST=0.0.0.0
ORCH_PORT=10112
DB_TOOL_ENDPOINT=http://127.0.0.1:10111/tools/invoke
# plus the LLM env above
```

**Start it**

```bash
cd ~/a2a-stack/orchestrator_llm
python -m venv .venv && source .venv/bin/activate
pip install fastapi uvicorn httpx pydantic python-dotenv google-generativeai
uvicorn orchestrator_llm:app --host ${ORCH_HOST:-0.0.0.0} --port ${ORCH_PORT:-10112} --log-level debug
```

---

# 5) End-to-end **curl tests** (from the VM)

> For debugging, first run **without** `| jq` to see raw JSON/any error text.

**A) List tables**

```bash
curl -s -X POST http://127.0.0.1:10112/chat \
  -H 'Content-Type: application/json' \
  -d '{"message":"list tables"}' | jq
```

**B) Describe a table**

```bash
curl -s -X POST http://127.0.0.1:10112/chat \
  -H 'Content-Type: application/json' \
  -d '{"message":"describe customers"}' | jq
```

**C) Natural ask → planner picks `mysql_query`**

```bash
curl -s -X POST http://127.0.0.1:10112/chat \
  -H 'Content-Type: application/json' \
  -d '{"message":"total paid amount by customer, highest first"}' | jq
```

---

## Quick troubleshooting (most common)

* **`jq: parse error`** → the response wasn’t JSON (likely a 4xx/5xx HTML error). Re-run **without** `| jq` to see the real error.
* **LLM returns code fences** (`json …`): ensure your orchestrator sanitizes fences and/or sets `response_mime_type="application/json"` for Vertex; keep a strict JSON-only system prompt.
* **Pydantic rows type errors**:

  * For `list tables`, either allow `rows: List[str]` **or** don’t set `rows` at all for that tool.
  * For `describe`/`query`, unwrap MCP results to lists (`columns`/`rows`) before putting into `rows`.
* **ConnectError to LLM**: check `OLLAMA_BASE_URL` or Vertex API key/ADC scopes.
* **Agent/MCP not reachable**: confirm each process is running on its port with `ss -lntp | grep 1011`.

---

