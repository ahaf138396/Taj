# TAJ v0.1 — Spec اجرایی (Product‑grade Skeleton)
**نام محصول:** تاج — تونل ارتباطی جویا  
**Scope این سند:** تعریف قراردادها و تصمیم‌های معماری برای شروع پیاده‌سازی v0.1  
**استک قفل‌شده:** Gateway+Agent (Go) ، Control Plane (FastAPI + Postgres)  
**Transport v0.1:** WebSocket/TLS + yamux (multiplexing)

---

## 1) اهداف و غیر اهداف

### اهداف v0.1
- تونل پایدار **TCP** (برای SSH/DB و هر سرویس TCP)
- جداسازی کامل **Control Plane** و **Data Plane**
- Multi‑tenant از روز اول (Org‑scoped همه چیز)
- امنیت پایه Product‑grade: Device Token + Session JWT کوتاه‌مدت + Audit
- قابلیت نصب/اجرای Agent به‌صورت سرویس (systemd) و reconnect خودکار
- قابلیت دیپلوی در یک VPS (Gateway) + یک سرور پشت NAT (Agent)

### غیر اهداف v0.1
- DNS مستقل (Authoritative DNS) ❌
- CDN/Proxy ابری ❌
- HTTP routing کامل و Custom Domain (می‌رود v0.2/v0.3) ❌
- Billing واقعی (فقط آماده‌سازی مدل‌ها/Hookها) ❌
- HA چند گیت‌وی (فقط طراحی آماده، پیاده‌سازی بعدی) ❌

---

## 2) واژگان (Terminology)
- **Org**: سازمان/حساب کاربری سطح بالا (tenant)
- **User**: کاربر انسانی پنل (RBAC)
- **Agent**: برنامه‌ای که روی ماشین مشتری/جویا نصب می‌شود و outbound وصل می‌شود
- **Gateway**: سرویس public روی VPS با Static IP که ترافیک را می‌پذیرد و فوروارد می‌کند
- **Tunnel**: کانال منطقی (container برای routeها)
- **Route**: قانون مسیریابی (در v0.1 فقط TCPRoute)
- **Policy**: محدودیت‌ها (ip allowlist, max conns, timeouts…)
- **Device Token**: توکن بلندمدت Agent برای گرفتن Session از Control
- **Session JWT**: توکن کوتاه‌مدت برای احراز Agent نزد Gateway
- **JWKS**: کلیدهای عمومی کنترل برای verify کردن JWT در Gateway

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
- **Data Plane (Gateway)**: عبور ترافیک و verify session JWT (بدون تماس دائم با DB)
- **Agent (Go)**: اتصال پایدار، رجیستر routeها، فوروارد به local

---

## 4) دامنه و DNS (تصمیم قفل)
- DNS روی Cloudflare یا هر DNS دیگر می‌ماند (فقط **DNS only**)
- برای v0.1 نیاز به HTTP نداریم؛ برای v0.2 دامنه سیستمی:
  - `*.taj.jooya.dev` → A به IP گیت‌وی + Wildcard TLS
- **Custom Domain**: v0.3

---

## 5) مدل امنیت و احراز هویت

### 5.1 Agent Device Token (بلندمدت)
- هنگام ساخت Agent در Control، یک `device_token` ساخته و **فقط یک‌بار** نمایش داده می‌شود.
- در DB فقط `hash(device_token)` ذخیره می‌شود.
- قابل rotate/revoke.

### 5.2 Session JWT (کوتاه‌مدت)
- Agent با Device Token از Control، یک Session JWT می‌گیرد (مثلاً 15 دقیقه).
- Agent با Session JWT به Gateway وصل می‌شود.
- Gateway Session JWT را با **JWKS** کنترل verify می‌کند (بدون تماس DB).

### 5.3 JWKS
- Control: `GET /v1/.well-known/jwks.json`
- Gateway کلیدها را cache می‌کند (مثلاً 10 دقیقه) و در صورت خطا fallback دارد.

### 5.4 RBAC (User panel)
- نقش‌ها: `owner | admin | dev | viewer`
- حداقل:
  - owner/admin: CRUD همه منابع
  - dev: CRUD تونل/route و مشاهده agent
  - viewer: فقط read

### 5.5 Audit
- تمام تغییرات منابع + رخدادهای اتصال Agent در `audit_events` ثبت می‌شود.

---

## 6) مدل داده (Postgres) — v0.1 (خلاصه)

> همه جداول `org_id` دارند (multi‑tenant واقعی).

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
- `type text` = 'tcp'
- `name text`
- `public_port int` (unique per gateway instance)
- `local_addr text` (مثل `127.0.0.1:22`)
- `policy_id uuid fk null`
- `enabled bool default true`
- `created_at timestamptz`

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

### sessions  (اختیاری برای revoke / telemetry)
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
- `POST /v1/agents/{agent_id}/rotate-token` → device_token جدید
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
- `GET /v1/tunnels/{tunnel_id}/routes`
- `PATCH /v1/routes/{route_id}` (enable/disable/update local_addr/public_port)

### Policies
- `POST /v1/policies`
- `GET /v1/policies`
- `PATCH /v1/policies/{policy_id}`

### Audit
- `GET /v1/audit?resource_type=&action=&from=&to=`

> نکته: در v0.1 gateway می‌تواند route map را از Control pull کند:
- `GET /v1/gateway/routes-snapshot` (protected by gateway credential)

---

## 8) پروتکل TAJ/1 (Gateway ↔ Agent)

### 8.1 Transport
- WebSocket over TLS
- مسیر: `wss://<gateway-host>/taj/ws`

### 8.2 Multiplex
- بعد از برقراری WS، روی connection یک **yamux session** ساخته می‌شود.
- هر اتصال ورودی (مثلاً TCP به پورت عمومی) = یک stream داخل yamux.
- مزیت: battle‑tested + ساده‌سازی شدید.

### 8.3 Control Stream (stream 0)
- روی یک stream مخصوص (اولین stream یا stream جدا) پیام‌های کنترلی JSON رد و بدل می‌شوند.
- سایر streamها باینری raw برای forward TCP خواهند بود.

### 8.4 Handshake (state machine)
1) WS connect
2) Agent → `HELLO`
3) Agent → `AUTH {session_jwt}`
4) Gateway → `AUTH_OK {conn_id}` یا `AUTH_FAIL`
5) Agent → `REGISTER {routes:[...], agent_meta:{...}}`
6) Gateway → `REGISTER_OK {effective_routes:[...]}`
7) Heartbeat دوره‌ای

### 8.5 پیام‌های کنترلی (JSON)
- `HELLO`
  - `{ "t":"HELLO", "v":"TAJ/1", "agent_id":"...", "org_id":"...", "cap":["tcp"] }`
- `AUTH`
  - `{ "t":"AUTH", "session_jwt":"..." }`
- `REGISTER`
  - `{ "t":"REGISTER", "routes":[{"route_id":"...","type":"tcp"}], "meta":{"hostname":"...","os":"..."} }`
- `HEARTBEAT`
  - `{ "t":"HEARTBEAT", "uptime_s":123, "open_streams":7 }`
- `ERROR`
  - `{ "t":"ERROR", "code":"...", "message":"..." }`

---

## 9) جریان TCP Tunnel (v0.1)

### 9.1 Preconditions
- Control: route تعریف شده (public_port, local_addr) و به tunnel/org وصل است.
- Agent همان route را در config دارد یا از Control دریافت می‌کند (v0.1: config‑driven).

### 9.2 Runtime Flow
1) User → `gateway_ip:public_port` وصل می‌شود.
2) Gateway route را resolve می‌کند → `agent_id`
3) Gateway روی yamux یک stream جدید باز می‌کند.
4) Gateway در ابتدای stream، یک header کوچک می‌فرستد:
   - `ROUTE <route_id>\n` (یا باینری TLV)
5) Agent stream را می‌گیرد، route_id را map می‌کند → `local_addr`
6) Agent به `local_addr` وصل می‌شود.
7) Forward bidirectional bytes تا close

### 9.3 Policy enforcement (Gateway-side)
- `max_conns` per route/org
- `idle_timeout_s`
- `ip_allowlist` (بر اساس src ip user)

### 9.4 Failure modes
- Agent offline → Gateway باید سریع fail کند (ECONNREFUSED/timeout)
- Session expired → Agent refresh و reconnect
- route mismatch → ERROR و close stream
- backpressure → محدودیت buffer و قطع graceful

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
  listen_ws: ":443"
  public_ip: "X.X.X.X"
  route_snapshot_url: "https://control.jooya.dev/v1/gateway/routes-snapshot"
  route_refresh_s: 10

auth:
  jwks_url: "https://control.jooya.dev/v1/.well-known/jwks.json"
  jwks_cache_s: 600

tcp:
  # gateway روی این پورت‌ها listen می‌کند (از snapshot می‌آید)
  allow_dynamic_listeners: true
```

---

## 11) Deployment (v0.1)
- Control:
  - FastAPI + Postgres + Alembic migrations
- Gateway:
  - یک VPS با Static IP
  - TLS cert (فعلاً می‌تواند self‑managed باشد؛ ACME در v0.2)
  - Portهای TCP مورد نیاز باید در firewall باز شوند.
- Agent:
  - نصب باینری + config + systemd unit

---

## 12) Observability (حداقل)
- JSON logs (timestamp, level, org_id, agent_id, route_id, conn_id)
- Metrics (اختیاری v0.1 اما توصیه‌شده):
  - active_agents
  - open_streams
  - bytes_in/out per route
  - auth_fail_count

---

## 13) نسخه‌بندی و سازگاری
- پروتکل: `TAJ/1`
- Agent و Gateway باید `v` را در HELLO ارسال کنند.
- breaking changes فقط با `TAJ/2`

---

## 14) Backlog بعد از v0.1
- v0.2: HTTP routing + managed domain wildcard + ACME
- v0.3: Custom domain verify + per-domain cert
- v1.0: quota/usage metering + پنل minimal + hardening بیشتر
