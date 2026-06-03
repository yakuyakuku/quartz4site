# Self-Hosting WAHA & n8n in Production: A Complete Integration & Backup Guide

Self-hosting **WAHA (WhatsApp HTTP API)** alongside **n8n** gives you full control over your customer automation pipeline, keeps your data private, and eliminates monthly subscription fees. 

This guide assumes you **already have Nginx Proxy Manager (NPM) and Cloudflare Tunnel installed and running** on your system. It explains how to deploy the core WAHA and n8n stack using the lightweight **BAILEYS** engine, connect it to your existing reverse proxy infrastructure, secure it, and set up automated backups.

---

## 1. Engine Selection: WEBJS vs. BAILEYS

When self-hosting WAHA, choosing the right WhatsApp driver (engine) is the most critical decision for server resource consumption:

| Feature | `WEBJS` Engine (Playwright/Chrome) | `BAILEYS` Engine (WebSockets) |
| :--- | :--- | :--- |
| **Technology** | Headless Chrome (via Puppeteer/Playwright) | Pure Node.js WebSocket implementation |
| **RAM Usage** | **High** (~300MB - 500MB per active session) | **Very Low** (~50MB - 100MB per active session) |
| **CPU Usage** | High (rendering pages, running layout engine) | Minimal (only processes network messages) |
| **VPS Fit** | Requires a VPS with 2GB+ RAM | Runs comfortably on a 1GB RAM $5 VPS |
| **Stability** | Very high (mimics a real browser) | High (requires updates when WhatsApp protocol changes) |
| **Persistence** | Large folder structure (browser profile) | Tiny JSON state & LevelDB files |

> [!TIP]
> **For self-hosting on lightweight VPS instances, the `BAILEYS` engine is highly recommended.** It uses a fraction of the memory and CPU, allowing you to run multiple WhatsApp sessions on cheap hardware.

---

## 2. Infrastructure Architecture (External Proxy + Internal Stack)

This setup integrates your **existing** Nginx Proxy Manager and Cloudflare Tunnel with the new WAHA/n8n stack. 

By utilizing Docker network sharing, your pre-existing NPM can route traffic to n8n and WAHA securely using their internal container hostnames. This eliminates the need to expose ports 5678 or 3000 to the public host interface.

```mermaid
graph TD
    %% Styling
    classDef ext fill:#f9f,stroke:#333,stroke-width:1px;
    classDef cf fill:#f93,stroke:#333,stroke-width:1px;
    classDef npm fill:#bbf,stroke:#333,stroke-width:1px;
    classDef waha fill:#f96,stroke:#333,stroke-width:1px;
    classDef db fill:#9f9,stroke:#333,stroke-width:1px;
    classDef redis fill:#ff9,stroke:#333,stroke-width:1px;
    classDef n8n fill:#9ff,stroke:#333,stroke-width:1px;

    %% Nodes & Flow
    WA[WhatsApp Server]:::ext <--> WAHA[WAHA API]:::waha
    User[Client Device]:::ext <-->|HTTPS| CFEdge[Cloudflare Edge]:::cf
    CFEdge <-->|Secure Tunnel| CFTunnel[Pre-existing Cloudflare Tunnel]:::cf
    CFTunnel <-->|HTTP| NPM[Pre-existing Nginx Proxy Manager]:::npm

    subgraph proxy-net [Shared Proxy Network]
        NPM
    end

    subgraph waha-net [waha-net Private Network]
        WAHA
        WH[n8n-webhook]:::n8n
        Main[n8n-main Editor]:::n8n
        Worker[n8n-worker]:::n8n
        Redis[(Redis)]:::redis
        DB[(PostgreSQL)]:::db
    end

    %% Routing Bridges
    NPM -->|n8n-main:5678| Main
    NPM -->|waha:3000| WAHA
    
    WAHA -->|Webhook| WH
    WH -->|Queue Job| Redis
    Worker -->|Pull Job| Redis
    Worker -->|API Call| WAHA
    Worker -->|State| DB
    Main -->|Config| DB
```

---

## 3. Core WAHA & n8n Stack (`docker-compose.yml`)

Save this configuration as `docker-compose.yml` in your stack directory (e.g. `~/waha-n8n-stack`). 

> [!IMPORTANT]
> This compose file references an external network called `proxy-net`. **Change this to the actual Docker network name used by your existing Nginx Proxy Manager container** (e.g., `npm_default`, `nginx-proxy`, or `proxy_network`).

```yaml
version: '3.8'

networks:
  # Private network for internal database, queue, and worker communications
  waha-net:
    driver: bridge
  # Connection to your pre-existing Nginx Proxy Manager network
  proxy-net:
    external: true

volumes:
  db_data:
  redis_data:
  n8n_shared:
  waha_data:

services:
  # ----------------------------------------------------
  # DATABASE BACKEND (PostgreSQL)
  # ----------------------------------------------------
  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-n8n_admin}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-SecretSecurePasswordChangeMe!}
      POSTGRES_DB: ${POSTGRES_DB:-n8n_prod}
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - waha-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n_admin -d n8n_prod"]
      interval: 5s
      timeout: 5s
      retries: 5

  # ----------------------------------------------------
  # QUEUE BROKER (Redis)
  # ----------------------------------------------------
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: always
    volumes:
      - redis_data:/data
    networks:
      - waha-net
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  # ----------------------------------------------------
  # WAHA (WhatsApp HTTP API using BAILEYS Engine)
  # ----------------------------------------------------
  waha:
    image: devlikeapro/waha:latest
    container_name: waha
    restart: always
    environment:
      - WAHA_ENGINE=BAILEYS # Optimized socket-based engine
      - WAHA_WEBHOOK_URL=http://n8n-webhook:5678/webhook/waha-trigger
      - WAHA_WEBHOOK_EVENTS=message,message.any,state.change
      - WAHA_API_KEY=${WAHA_API_KEY:-WAHA_SUPER_SECRET_KEY}
    volumes:
      - waha_data:/data # Critical: Stores Baileys JSON authentication keys
    networks:
      - waha-net
      - proxy-net # Attach to proxy network so NPM can route to it
    depends_on:
      - postgres
      - redis

  # ----------------------------------------------------
  # n8n MAIN CONTAINER (Editor & Scheduler)
  # ----------------------------------------------------
  n8n-main:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n-main
    restart: always
    environment:
      - N8N_HOST=${N8N_DOMAIN:-n8n.yourdomain.com}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB:-n8n_prod}
      - DB_POSTGRESDB_USER=${POSTGRES_USER:-n8n_admin}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD:-SecretSecurePasswordChangeMe!}
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY:-GenerateAStrongRandomStringHere!}
    volumes:
      - n8n_shared:/home/node/.n8n
    networks:
      - waha-net
      - proxy-net # Attach to proxy network so NPM can route to it
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  # ----------------------------------------------------
  # n8n WEBHOOK PROCESSOR (Stateless Intake)
  # ----------------------------------------------------
  n8n-webhook:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n-webhook
    restart: always
    command: webhook
    environment:
      - N8N_HOST=${N8N_DOMAIN:-n8n.yourdomain.com}
      - NODE_ENV=production
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB:-n8n_prod}
      - DB_POSTGRESDB_USER=${POSTGRES_USER:-n8n_admin}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD:-SecretSecurePasswordChangeMe!}
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY:-GenerateAStrongRandomStringHere!}
    volumes:
      - n8n_shared:/home/node/.n8n
    networks:
      - waha-net
      - proxy-net # Attach to proxy network so NPM can route webhook responses if needed externally
    depends_on:
      - n8n-main

  # ----------------------------------------------------
  # n8n WORKER (Workflow Executor)
  # ----------------------------------------------------
  n8n-worker:
    image: docker.n8n.io/n8nio/n8n:latest
    restart: always
    command: worker
    environment:
      - NODE_ENV=production
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB:-n8n_prod}
      - DB_POSTGRESDB_USER=${POSTGRES_USER:-n8n_admin}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD:-SecretSecurePasswordChangeMe!}
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY:-GenerateAStrongRandomStringHere!}
    volumes:
      - n8n_shared:/home/node/.n8n
    networks:
      - waha-net
    depends_on:
      - n8n-main
```

---

## 4. Connecting to Your Existing Nginx Proxy Manager (NPM)

Because the stack's public containers (`n8n-main`, `n8n-webhook`, and `waha`) are attached to your pre-existing NPM Docker network (`proxy-net`), NPM can route traffic to them directly using their service names.

### Step-by-Step NPM GUI Configuration

#### 1. Configure the Proxy Host for n8n
1.  Log in to your **Nginx Proxy Manager Admin Dashboard**.
2.  Navigate to **Hosts** > **Proxy Hosts** and click **Add Proxy Host**.
3.  **Domain Names:** Enter your n8n domain (e.g., `n8n.yourdomain.com`).
4.  **Scheme:** Select `http`.
5.  **Forward Hostname/IP:** Enter `n8n-main` *(this matches the container service name).*
6.  **Forward Port:** Enter `5678`.
7.  **Websockets Support:** **Toggle ON** *(CRITICAL: n8n uses WebSockets to establish real-time editor sync and node logging)*.
8.  **Block Common Exploits:** **Toggle ON**.
9.  Click **Save**.

#### 2. Configure the Proxy Host for WAHA API
1.  Click **Add Proxy Host**.
2.  **Domain Names:** Enter your WAHA domain (e.g., `waha.yourdomain.com`).
3.  **Scheme:** Select `http`.
4.  **Forward Hostname/IP:** Enter `waha` *(matches the container service name).*
5.  **Forward Port:** Enter `3000`.
6.  **Websockets Support:** **Toggle ON** *(CRITICAL: WAHA streams active connection states and session QR code updates via WebSockets)*.
7.  Click **Save**.

---

## 5. Exposing Securely with Your Cloudflare Tunnel

Since you already use Cloudflare Tunnels, you do not need to open public firewall ports (80 or 443) on your server. Your existing `cloudflared` agent container will forward external edge traffic directly to your existing Nginx Proxy Manager.

### Step-by-Step Routing Configurations

#### 1. Cloudflare Dashboard Mapping
In your **Cloudflare Zero Trust Dashboard**:
1.  Go to **Networks** > **Tunnels** and edit your active tunnel.
2.  Under **Public Hostname**, map your new subdomains to your NPM container:

| Public Hostname | Service Type | Service URL | Description |
| :--- | :--- | :--- | :--- |
| `n8n.yourdomain.com` | `HTTP` | `nginx-proxy-manager:80` | Passes traffic to NPM, which proxies it to `n8n-main:5678` |
| `waha.yourdomain.com` | `HTTP` | `nginx-proxy-manager:80` | Passes traffic to NPM, which proxies it to `waha:3000` |

> [!NOTE]
> Adjust `nginx-proxy-manager` to the container name of your NPM service as recognized inside your shared Docker network.

#### 2. Cloudflare Zero Trust Security (Protecting WAHA)
Because the WAHA container exposes control APIs that can read conversations and send messages, it should **never** be open to the public internet without an authentication layer.

1.  In Cloudflare Zero Trust, navigate to **Access** > **Applications** > **Add an Application**.
2.  Select **Self-Hosted**.
3.  Set the Application Domain to `waha.yourdomain.com`.
4.  Create a policy that blocks all public requests, only allowing access if the user authenticates via **One-Time PIN (OTP)** sent to your corporate email, or checks out successfully via an **IP Source Address** rule matching your home/office VPN.

---

## 6. Real-World Use Case: Automated Lead Responder & AI Support Bot

This section details how to implement a production-grade WhatsApp workflow: a bot that listens for incoming customer messages, checks their status in a database, triggers an AI response, simulates human typing, and replies automatically.

### Concrete Data Flow Steps

```mermaid
sequenceDiagram
    autonumber
    actor Customer as User Mobile
    participant WA as WAHA API
    participant WH as n8n Webhook
    participant DB as PostgreSQL
    participant AI as AI Agent (Ollama/OpenAI)
    participant Exec as n8n Worker

    Customer->>WA: "Hi, I need support"
    WA->>WH: Webhook POST (message payload)
    WH-->>WA: 200 OK
    Note over Exec: Worker retrieves task
    Exec->>DB: Query customer info (check if leads exists)
    DB-->>Exec: Customer records
    Exec->>AI: Send prompt (history + context)
    AI-->>Exec: Generated Response text
    Exec->>WA: POST /presence (status: online)
    Exec->>WA: POST /chat/startTyping
    Note over Exec: Wait dynamically based on text length
    Exec->>WA: POST /sendText (response message)
    WA->>Customer: Delivery message
```

### 1. Inbound Webhook Payload Format
When a customer sends a message, WAHA triggers n8n-webhook with this JSON payload:

```json
{
  "event": "message",
  "session": "default",
  "metadata": {},
  "payload": {
    "id": "true_628123456789_3EB0C34B8F12",
    "timestamp": 1717325200,
    "from": "628123456789@c.us",
    "to": "628987654321@c.us",
    "body": "Hi, I need support",
    "hasMedia": false,
    "fromMe": false,
    "type": "chat"
  }
}
```

### 2. Node-by-Node n8n Workflow Construction

#### Node 1: Webhook Trigger (from WAHA)
*   **Path:** `waha-trigger`
*   **Method:** `POST`
*   **Respond:** `Immediately (200 OK)` *(Ensures WAHA does not hold connections open or retry during heavy logic)*

#### Node 2: Check Session / Ignore Self-Messages
Add an **If Node** to ensure n8n only responds to inbound messages (preventing infinite loops where the bot replies to itself).
*   **Condition:** `{{ $json.payload.fromMe }}` is equal to `false`.

#### Node 3: Query Database (PostgreSQL Node)
Check if the contact exists in your local CRM database using an SQL query:
*   **SQL Statement:**
    ```sql
    SELECT id, first_name, subscription_status 
    FROM users 
    WHERE whatsapp_id = '{{ $json.payload.from }}';
    ```

#### Node 4: Dynamic Delay & Human Typing Simulation
Use a **Code Node** in n8n to calculate typing delay based on response length and control WAHA presence. 

```javascript
// Calculate dynamic wait time (100ms per character of the AI response)
const responseText = $("AI Agent").item.json.output || "Hello! How can I help you?";
const delayTime = Math.min(Math.max(responseText.length * 50, 1500), 5000); // Between 1.5s and 5s

return {
  delayMs: delayTime,
  to: $("Webhook Trigger").item.json.payload.from,
  message: responseText
};
```

1.  **n8n HTTP Request Node (Set Online):**
    *   **Method:** `POST`
    *   **URL:** `http://waha:3000/api/presence`
    *   **Headers:** `Authorization: Bearer YourWahaSuperSecretKeyGoesHere`
    *   **Body (JSON):**
        ```json
        {
          "presence": "online"
        }
        ```
2.  **n8n HTTP Request Node (Start Typing):**
    *   **Method:** `POST`
    *   **URL:** `http://waha:3000/api/chat/startTyping`
    *   **Body (JSON):**
        ```json
        {
          "chatId": "{{ $json.to }}"
        }
        ```
3.  **n8n Wait Node:**
    *   **Type:** `Custom Delay`
    *   **Delay:** `{{ $json.delayMs }}` ms
4.  **n8n HTTP Request Node (Send Message):**
    *   **Method:** `POST`
    *   **URL:** `http://waha:3000/api/sendText`
    *   **Body (JSON):**
        ```json
        {
          "chatId": "{{ $json.to }}",
          "text": "{{ $json.message }}"
        }
        ```

---

## 7. Self-Hosted Data Persistence & Backups

When running this stack, your database states and WhatsApp connection keys must be preserved. If your server dies and you lose the WAHA directory, **your WhatsApp accounts will disconnect, forcing you to manually re-scan the QR code for every device**.

### What Needs to be Backed Up?
1.  **WAHA Session Storage:** In `BAILEYS` mode, this is the `waha_data` directory. It contains session keys, JSON credentials, and local caching.
2.  **n8n Configuration & History:** Stored in the PostgreSQL database (`db_data` volume) and n8n shared static assets (`n8n_shared`).

### Automated Backup & Sync Script
This script performs a safe hot-backup, uploads it to your configured cloud endpoint (S3/R2/B2) using `rclone`, and fires a Discord webhook alert status notification.

#### 1. Setup the Backup Script
Create a script on your host machine named `backup.sh` in your stack folder:

```bash
#!/bin/bash
# backup.sh - Automated Production Backup Script for WAHA and n8n

# --- Configuration ---
STACK_DIR="/home/yaku/waha-n8n-stack"
BACKUP_DIR="/backups"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
RETENTION_DAYS=7
DISCORD_WEBHOOK="https://discord.com/api/webhooks/YOUR_WEBHOOK_URL"

# Load variables from .env
if [ -f "$STACK_DIR/.env" ]; then
    export $(grep -v '^#' "$STACK_DIR/.env" | xargs)
fi

DB_USER=${POSTGRES_USER:-n8n_admin}
DB_NAME=${POSTGRES_DB:-n8n_prod}

mkdir -p "$BACKUP_DIR"

# Discord Notification Helper
send_notification() {
    local message="$1"
    local color="$2" # Decimal color representation (e.g. green: 3066993, red: 15158332)
    curl -H "Content-Type: application/json" \
         -X POST \
         -d "{\"embeds\": [{\"title\": \"Backup Status\", \"description\": \"$message\", \"color\": $color}]}" \
         "$DISCORD_WEBHOOK"
}

echo "[$(date)] Starting Backup..."

# 1. Hot-export the PostgreSQL database state
if docker exec postgres pg_dump -U "$DB_USER" "$DB_NAME" > "$BACKUP_DIR/n8n_db_$TIMESTAMP.sql"; then
    echo "PostgreSQL Dump Successful."
else
    send_notification "FAILED: PostgreSQL database backup failed." 15158332
    exit 1
fi

# 2. Archive WAHA session folder & n8n shared files
if tar -czf "$BACKUP_DIR/waha_sessions_$TIMESTAMP.tar.gz" -C "$STACK_DIR" waha_data n8n_shared; then
    echo "File Volume Archiving Successful."
else
    send_notification "FAILED: File storage archiving failed." 15158332
    exit 1
fi

# Compress database dump
gzip "$BACKUP_DIR/n8n_db_$TIMESTAMP.sql"

# 3. Upload to Remote Cloud Storage using rclone (S3/B2/R2)
# Ensure rclone is configured on the host machine: 'rclone config'
if rclone copy "$BACKUP_DIR" remote:my-production-backup-bucket/backups/ --include "*$TIMESTAMP*"; then
    send_notification "SUCCESS: Backup completed and synced to Remote Cloud Storage." 3066993
else
    send_notification "WARNING: Backup created locally but Remote Cloud Sync failed." 15158332
fi

# 4. Clean up local backups older than RETENTION_DAYS
find "$BACKUP_DIR" -type f -mtime +$RETENTION_DAYS -delete

echo "[$(date)] Backup Completed successfully."
```

#### 2. Schedule the Script
Make the script executable:
```bash
chmod +x ~/waha-n8n-stack/backup.sh
```

Add it to your system's `cron` jobs to run daily at 2:00 AM. Open the crontab editor:
```bash
crontab -e
```

Add the following line:
```cron
0 2 * * * /home/yaku/waha-n8n-stack/backup.sh >> /var/log/waha_backup.log 2>&1
```

---

## 8. Disaster Recovery: Restoring Your Sessions

If you need to migrate to a new server or recover from a server crash:

1.  Set up the folder structure on the new server:
    ```bash
    mkdir -p ~/waha-n8n-stack
    ```
2.  Restore the configuration files (`docker-compose.yml` and your `.env`).
3.  Extract the session archive back into the stack directory:
    ```bash
    tar -xzf waha_sessions_YYYYMMDD_HHMMSS.tar.gz -C ~/waha-n8n-stack/
    ```
4.  Start the database container first to prepare for import:
    ```bash
    docker compose up -d postgres
    ```
5.  Restore the database dump:
    ```bash
    gunzip n8n_db_YYYYMMDD_HHMMSS.sql.gz
    docker cp n8n_db_YYYYMMDD_HHMMSS.sql postgres:/tmp/restore.sql
    docker exec -it postgres psql -U n8n_admin -d n8n_prod -f /tmp/restore.sql
    ```
6.  Start the rest of the stack:
    ```bash
    docker compose up -d
    ```

Because the BAILEYS keys in `waha_data` match the session signature, WAHA will connect instantly without needing to scan QR codes.

---

## 9. Operation & Monitoring Runbook

Once your stack is running, use these command structures to monitor, troubleshoot, and scale your services.

### Checking Docker Containers Health
To check if all services are healthy and running:
```bash
docker compose ps
```

### Viewing Logs in Real-time
If webhooks are failing, tail the logs of the webhook container:
```bash
docker logs -f n8n-webhook
```

To watch WAHA connect to the WhatsApp servers and trace incoming messages:
```bash
docker logs -f waha
```

### Inspecting active Redis Queues
If executions are slow, verify that tasks are queueing properly in Redis:
```bash
docker exec -it redis redis-cli --stat
```
To inspect the specific queue keys used by n8n:
```bash
docker exec -it redis redis-cli KEYS "bull:*"
```

### Scaling Worker Capacity Dynamically
If n8n displays queue congestion alerts, scale worker processes on the host:
```bash
docker compose up -d --scale n8n-worker=3
```
This spawns three concurrent worker containers automatically balancing the workload.
