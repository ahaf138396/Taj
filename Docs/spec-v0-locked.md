# TAJ v0.1 — Spec اجرایی (Product-grade Skeleton) — نسخه قفل‌شده
**نام محصول:** تاج — تونل ارتباطی جویا  
**Scope این سند:** قراردادها و تصمیم‌های معماری برای شروع پیاده‌سازی v0.1  
**استک قفل‌شده:** Gateway+Agent (Go) ، Control Plane (FastAPI + Postgres)  
**Transport v0.1:** WebSocket/TLS + **yamux** (multiplexing)  

---

## 0) تصمیم‌های قفل‌شده (Final Lock)
این‌ها از اینجا به بعد «ثابت» فرض می‌شوند مگر با دلیل جدی:

1) **DNS مستقل نداریم**؛ Cloudflare/هر DNS دیگر فقط **DNS-only**.  
2) v0.1 فقط **TCP tunneling** (SSH/DB/…)؛ HTTP routing از v0.2.  
3) Agent با **Device Token** از Control یک **Session JWT کوتاه‌مدت** می‌گیرد.  
4) Gateway Session JWT را فقط با **JWKS** کنترل **Verify** می‌کند (بدون DB).  
5) Multiplex روی اتصال Agent↔Gateway با **yamux**.  
6) هر **TCPRoute** در v0.1 به‌صورت صریح به **یک Agent** بایند می‌شود (`routes.agent_id`).  
7) Header باینری ابتدای هر stream برای انتخاب route: **Magic + route_uuid + conn_id**.  
8) Gateway برای ساختن Listenerها، از Control یک **routes snapshot** می‌کشد (pull هر 10s).  
9) اولین stream بازشده از Agent به Gateway، **Control Stream** است (JSON messages).  

---

## 1) اهداف و غیر اهداف

### اهداف v0.1
- تونل پایدار **TCP**
- جداسازی کامل **Control Plane** و **Data Plane**
- Multi-tenant از روز اول (Org-scoped همه چیز)
- امنیت پایه Product-grade: Device Token + Session JWT کوتاه‌مدت + Audit
- Agent به‌صورت سرویس (systemd) + reconnect خودکار
- دیپلوی ساده: یک VPS (Gateway) + یک سرور پشت NAT (Agent)

### غیر اهداف v0.1
- DNS مستقل ❌
- CDN/Proxy ابری ❌
- HTTP routing + Custom Domain ❌ (می‌رود v0.2/v0.3)
- Billing واقعی ❌
- HA چند گیت‌وی (فقط طراحی آماده) ❌

---

## 2) واژگان (Terminology)
- **Org**: سازمان/tenant
- **User**: کاربر انسانی پنل (RBAC)
- **Agent**: برنامه روی ماشین مشتری/جویا (outbound)
- **Gateway**: سرویس public روی VPS (Static IP)
- **Tunnel**: کانال منطقی (container برای routeها)
- **Route**: قانون مسیریابی (v0.1 فقط TCPRoute)
- **Policy**: محدودیت‌ها (ip allowlist, max conns, timeouts…)
- **Device Token**: توکن بلندمدت Agent برای گرفتن Session از Control
- **Session JWT**: توکن کوتاه‌مدت برای احراز Agent نزد Gateway
- **JWKS**: کلیدهای عمومی کنترل برای verify JWT در Gateway

---

## 3) معماری کلان
```
          (DNS only)             (Static IP)
DNS Provider ───────────►  TAJ Gateway  ───────────► Internet Users
                               ▲   │
                               │   ▼ (WS/TLS + yamux)
                             TAJ Agent  ───────────► Local Services (SSH/DB/…)
                       (Outbound only, behind NAT)
```

### جداسازی لایه‌ها
- **Control Plane (FastAPI)**: مدیریت هویت‌ها/تنظیمات/صدور Session
- **Data Plane (Gateway)**: عبور ترافیک و verify Session JWT (بدون DB)
- **Agent (Go)**: اتصال پایدار، رجیستر routeها، فوروارد به local

---

## 4) دامنه و DNS (تصمیم قفل)
- DNS روی Cloudflare یا هر DNS دیگر می‌ماند (فقط **DNS only**)
- v0.1: HTTP نداریم
- v0.2: دامنه سیستمی `*.taj.jooya.dev` → A به IP گیت‌وی + Wildcard TLS
- v0.3: Custom Domain

---

## 5) مدل امنیت و احراز هویت

### 5.1 Agent Device Token (بلندمدت)
- هنگام ساخت Agent در Control، یک `device_token` ساخته و **فقط یک‌بار** نمایش داده می‌شود.
- در DB فقط `sha256(device_token)` ذخیره می‌شود.
- قابل rotate/revoke.
- پیشنهاد فرمت: `taj_dt_<base64url(32bytes)>`

### 5.2 Session JWT (کوتاه‌مدت)
- Agent با Device Token از Control، Session JWT می‌گیرد (مثلاً 15 دقیقه).
- Agent با Session JWT به Gateway وصل می‌شود.
- Gateway Session JWT را با JWKS کنترل verify می‌کند.

**Claimهای قفل‌شده (minimum):**
- `iss`: `taj-control`
- `aud`: `taj-gateway`
- `sub`: `<agent_id>`
- `org_id`: `<org_id>`
- `exp`, `iat`
- `jti`: برای revoke/telemetry (اختیاری اما توصیه‌شده)
- `scope`: `["agent:connect"]`

**Gateway MUST verify:**
- signature (JWKS)
- `iss`, `aud`
- `exp`
- سازگاری `sub`/`org_id` با HELLO

### 5.3 JWKS
- Control: `GET /v1/.well-known/jwks.json`
- Gateway cache: 10 دقیقه
- اگر `kid` ناشناخته بود: یک بار refresh فوری + retry verify

### 5.4 RBAC (User panel)
- نقش‌ها: `owner | admin | dev | viewer`
- حداقل:
  - owner/admin: CRUD همه منابع
  - dev: CRUD tunnel/route/policy و مشاهده agent
  - viewer: فقط read

### 5.5 Audit (قفل)
- تغییر منابع: create/update/delete
- رخدادهای اتصال: agent.connected / agent.disconnected
- متا: `remote_ip`, `gateway_id`, `user_agent/version`, `route_id`

---

## 6) مدل داده (Postgres) — v0.1 (قفل)

> همه جداول `org_id` دارند (multi-tenant واقعی).

### orgs
- `id uuid pk`
- `name text`
- `slug text unique`
- `plan text default 'free'`
- `created_at timestamptz`

### users
- `id uuid pk`
- `org_id uuid fk`
- `email text unique`
- `password_hash text`
- `role text`
- `created_at timestamptz`

### agents
- `id uuid pk`
- `org_id uuid fk`
- `name text`
- `device_token_hash text`
- `status text` (active/revoked)
- `last_seen_at timestamptz`
- `created_at timestamptz`

### tunnels
- `id uuid pk`
- `org_id uuid fk`
- `name text`
- `description text null`
- `created_at timestamptz`

### policies
- `id uuid pk`
- `org_id uuid fk`
- `ip_allowlist cidr[] null`
- `rate_limit_rps int null`
- `max_conns int null`
- `idle_timeout_s int null`
- `created_at timestamptz`

### routes  (v0.1: فقط TCP)
- `id uuid pk`
- `org_id uuid fk`
- `tunnel_id uuid fk`
- `agent_id uuid fk`  ← **قفل: هر route به یک agent**
- `type text` = 'tcp'
- `name text`
- `public_port int`  ← **unique per gateway**
- `local_addr text` (مثل `127.0.0.1:22`)
- `policy_id uuid fk null`
- `enabled bool default true`
- `created_at timestamptz`

**Constraints (قفل):**
- `unique(public_port)` (در سطح یک Gateway instance)  
  - v0.1 تک‌گیت‌وی است، پس unique ساده کافی است.  
  - v1+ اگر multi-gateway شد، `gateway_id` اضافه می‌کنیم و unique روی `(gateway_id, public_port)` می‌رود.

### audit_events
- `id uuid pk`
- `org_id uuid fk`
- `actor_user_id uuid null`
- `actor_agent_id uuid null`
- `action text`
- `resource_type text`
- `resource_id uuid null`
- `meta jsonb`
- `created_at timestamptz`

### sessions (اختیاری اما توصیه‌شده)
- `id uuid pk`
- `org_id uuid fk`
- `agent_id uuid fk`
- `jwt_id text`
- `expires_at timestamptz`
- `revoked_at timestamptz null`
- `created_at timestamptz`

---

## 7) API Control Plane (FastAPI) — v0.1 (حداقل لازم)

### Health
- `GET /health`

### Auth user
- `POST /v1/auth/login` → user JWT
- `POST /v1/auth/logout`

### JWKS
- `GET /v1/.well-known/jwks.json`

### Agents
- `POST /v1/agents` → ایجاد Agent + **device_token (one-time)**
- `GET /v1/agents`
- `GET /v1/agents/{agent_id}`
- `POST /v1/agents/{agent_id}/rotate-token`
- `POST /v1/agents/{agent_id}/revoke`

### Sessions (Agent → Gateway)
- `POST /v1/agents/{agent_id}/sessions`
  - Auth: Header `X-TAJ-DEVICE-TOKEN: <token>`
  - Output: `{ "session_jwt": "...", "expires_in": 900 }`

### Tunnels
- `POST /v1/tunnels`
- `GET /v1/tunnels`
- `GET /v1/tunnels/{tunnel_id}`

### Routes (TCP)
- `POST /v1/tunnels/{tunnel_id}/routes`
  - body شامل `agent_id`, `public_port`, `local_addr`, `policy_id?`
- `GET /v1/tunnels/{tunnel_id}/routes`
- `PATCH /v1/routes/{route_id}`

### Policies
- `POST /v1/policies`
- `GET /v1/policies`
- `PATCH /v1/policies/{policy_id}`

### Audit
- `GET /v1/audit?...`

### Gateway routes snapshot (قفل)
Gateway برای ساخت listeners و map:
- `GET /v1/gateway/routes-snapshot`
  - Auth: `Authorization: Bearer <gateway_token>` (یک secret در config)
  - Output (نمونه):
    ```json
    {
      "generated_at": "2025-12-13T00:00:00Z",
      "routes": [
        {
          "route_id": "...",
          "org_id": "...",
          "agent_id": "...",
          "public_port": 2222,
          "local_addr": "127.0.0.1:22",
          "policy": {"ip_allowlist": null, "max_conns": 200, "idle_timeout_s": 60}
        }
      ]
    }
    ```

---

## 8) پروتکل TAJ/1 (Gateway ↔ Agent)

### 8.1 Transport
- WebSocket over TLS
- مسیر: `wss://<gateway-host>/taj/ws`
- WS Ping/Pong: هر 20s (برای تشخیص قطع ارتباط)

### 8.2 Multiplex (قفل)
- بعد از برقراری WS، یک **yamux session** ساخته می‌شود.
- هر اتصال ورودی = یک stream داخل yamux.
- **Agent نقش “client” yamux** و **Gateway نقش “server” yamux** (قفل).

### 8.3 Control Stream (قفل)
- Agent **بلافاصله بعد از ایجاد yamux session** یک stream باز می‌کند.
- آن stream، **Control Stream** است و فقط پیام‌های JSON Line-Delimited می‌برد:
  - هر پیام JSON در یک خط (با `\n` تمام می‌شود).

### 8.4 Handshake (state machine)
1) WS connect
2) yamux establish
3) Agent opens Control Stream
4) Agent → `HELLO\n`
5) Agent → `AUTH\n`
6) Gateway → `AUTH_OK\n` یا `AUTH_FAIL\n`
7) Agent → `REGISTER\n`
8) Gateway → `REGISTER_OK\n` یا `ERROR\n`
9) Heartbeat دوره‌ای

### 8.5 پیام‌های کنترلی (JSONL)
- `HELLO`
  - `{ "t":"HELLO", "v":"TAJ/1", "agent_id":"...", "org_id":"...", "cap":["tcp"], "agent_ver":"0.1.0" }`
- `AUTH`
  - `{ "t":"AUTH", "session_jwt":"..." }`
- `REGISTER`
  - `{ "t":"REGISTER", "routes":[{"route_id":"..."}], "meta":{"hostname":"...","os":"..."} }`
- `HEARTBEAT`
  - `{ "t":"HEARTBEAT", "uptime_s":123, "open_streams":7 }`
- `ERROR`
  - `{ "t":"ERROR", "code":"...", "message":"..." }`

**REGISTER rules (قفل):**
- Gateway فقط routeهایی را می‌پذیرد که:
  - در snapshot وجود دارند
  - `agent_id` آن route برابر `agent_id` در JWT باشد
  - `org_id` هم match باشد

---

## 9) جریان TCP Tunnel (v0.1) — قفل

### 9.1 Runtime Flow
1) User → `gateway_ip:public_port` وصل می‌شود.
2) Gateway با snapshot resolve می‌کند → `route_id` و `agent_id`
3) Gateway policy را apply می‌کند (ip_allowlist/max_conns/idle_timeout)
4) Gateway روی yamux به Agent یک stream جدید باز می‌کند.
5) Gateway روی ابتدای stream یک Header ثابت می‌فرستد (باینری):
   - `MAGIC(4) + ROUTE_UUID(16) + CONN_ID(4)`
6) Agent header را می‌خواند، route را map می‌کند → `local_addr`
7) Agent به `local_addr` وصل می‌شود.
8) Forward bidirectional bytes تا close.

### 9.2 Header باینری (قفل)
- `MAGIC`: 4 bytes = ASCII `"TAJ1"`
- `ROUTE_UUID`: 16 bytes (RFC4122 raw bytes)
- `CONN_ID`: uint32 big-endian (برای correlation log/audit)

> Agent MUST اگر MAGIC غلط بود stream را ببندد.

### 9.3 Failure modes (قفل)
- Agent offline → Gateway: fail سریع (timeout کوتاه) + log/audit
- Session expired → Agent refresh session + reconnect
- route mismatch / not registered → Gateway: reject stream
- local connect fail → Agent close stream

---

## 10) کانفیگ‌ها (v0.1)

### 10.1 taj-agent.yml
```yaml
agent:
  id: "AGENT_UUID"
  org: "ORG_UUID"
  name: "jooya-server"
  control_url: "https://control.jooya.dev"
  gateway_url: "wss://gw1.jooya.dev/taj/ws"
  device_token: "TAJ_DEVICE_TOKEN"

tunnels:
  - id: "TUNNEL_UUID"
    routes:
      - id: "ROUTE_UUID"
        type: "tcp"
        name: "ssh"
        public_port: 2222
        local: "127.0.0.1:22"
```

### 10.2 taj-gateway.yml
```yaml
gateway:
  id: "gw1"
  listen_ws: ":443"
  route_snapshot_url: "https://control.jooya.dev/v1/gateway/routes-snapshot"
  route_refresh_s: 10
  gateway_token: "TAJ_GATEWAY_TOKEN"

auth:
  jwks_url: "https://control.jooya.dev/v1/.well-known/jwks.json"
  jwks_cache_s: 600

tcp:
  allow_dynamic_listeners: true
  listener_backlog: 1024
```

---

## 11) Timeout/Keepalive (قفل پیشنهادی برای پیاده‌سازی)
- WS ping interval: 20s
- WS pong timeout: 10s
- Handshake deadline: 10s
- Agent heartbeat: 30s
- TCP idle timeout (default policy): 60s

---

## 12) Observability (حداقل)
- JSON logs با fields:
  - `org_id`, `agent_id`, `route_id`, `conn_id`, `remote_ip`, `gateway_id`
- Metrics (اگر گذاشتیم):
  - `taj_active_agents`
  - `taj_open_streams`
  - `taj_bytes_in_total`, `taj_bytes_out_total`
  - `taj_auth_fail_total`

---

## 13) نسخه‌بندی
- پروتکل: `TAJ/1`
- breaking changes فقط با `TAJ/2`

---

## 14) Backlog بعد از v0.1
- v0.2: HTTP routing + managed domain wildcard + ACME
- v0.3: Custom domain verify + per-domain cert
- v1.0: quota/usage + پنل minimal + hardening بیشتر
