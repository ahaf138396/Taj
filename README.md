# English 🇬🇧



# TAJ — Multi‑Tenant TCP Tunneling Platform

**TAJ** is a **multi‑tenant TCP tunneling platform** designed to provide secure access to services located behind **NAT or firewalls**.

The system establishes a persistent connection using an **outbound Agent**, a **public Gateway**, and a **Control Plane**, allowing users to access internal services such as **SSH, databases, or any other TCP service**.

The goal of **v0** is to deliver a **stable, secure, and simple TCP tunnel** with an extensible architecture.

---

# Architecture

The system consists of **three main layers**.

## Control Plane

The management service responsible for:

- Managing organizations (**multi‑tenant**)
- Managing **Agents**
- Defining **Tunnels** and **Routes**
- Issuing **Session JWTs** for Agents
- Providing **JWKS** for token validation
- Providing **route snapshots** to Gateways

### Technology

- FastAPI
- PostgreSQL
- Alembic migrations

---

## Gateway (Data Plane)

The Gateway is responsible for handling **actual traffic flow**.

### Responsibilities

- Accepting user connections on **public ports**
- Maintaining **WebSocket connections with Agents**
- **Multiplexing streams using yamux**
- **Forwarding TCP traffic**
- Enforcing **policies**

### Key Feature

The Gateway operates **without a direct dependency on the database** and receives **route information as snapshots** from the **Control Plane**.

---

## Agent

The Agent is a lightweight service that runs on the **user's machine**.

### Features

- **Outbound connection to the Gateway** (NAT‑friendly)
- Authentication using a **Device Token**
- Obtaining a **Session JWT from the Control Plane**
- **Registering routes**
- Connecting to **local services** and forwarding traffic

The Agent is implemented in **Go**.

---

# High Level Flow

1. The Agent obtains a **Session JWT** from the Control Plane using a `device_token`.
2. The Agent connects to the Gateway via **WebSocket over TLS**.
3. After connection, a **yamux session** is established.
4. The Agent **registers its routes**.
5. A user connects to `gateway_ip:public_port`.
6. The Gateway opens a **new stream over yamux**.
7. The Agent establishes a connection to the **local service**.
8. **Bidirectional TCP traffic** is transferred.

---

# Protocol

The communication protocol between the **Agent and Gateway** is defined as **TAJ/1**.

### Transport

WebSocket over TLS

### Multiplex

yamux

After the connection is established, the Agent creates a **Control Stream** through which **JSON control messages** are exchanged.

### Handshake Steps

1. WS connection
2. yamux session
3. HELLO
4. AUTH
5. REGISTER
6. HEARTBEAT

---

# Security Model

System security is based on two mechanisms.

---

## Device Token

- A **long‑lived token per Agent**
- **Displayed only once**
- A **hashed version is stored in the database**
- Can be **rotated** and **revoked**

---

## Session JWT

The Agent obtains a **short‑lived JWT** using the **Device Token**.

The Gateway validates this token using **JWKS**.

### Main Claims

- `iss`
- `aud`
- `sub (agent_id)`
- `org_id`
- `exp`
- `scope`

---

# Data Model (Simplified)

Core system tables:

- orgs
- users
- agents
- tunnels
- routes
- policies
- audit_events
- sessions

### Important Feature

All tables include `org_id` to ensure **multi‑tenant isolation**.

---

# TCP Tunnel Flow

When a user connects to a **public Gateway port**:

1. The Gateway **resolves the route**
2. **Policy enforcement** is applied
3. A **new yamux stream** is created
4. A **binary header** is sent

### Header Format
MAGIC(4) + ROUTE_UUID(16) + CONN_ID(4)

text

The Agent reads this header, **selects the appropriate route**, and connects to the **local service**.

---

# Configuration

## Agent

File:
taj-agent.yml

text

Main fields:

- agent id
- org id
- device token
- gateway url
- control url
- route definitions

---

## Gateway

File:
taj-gateway.yml

text

Main fields:

- gateway id
- websocket listen address
- jwks url
- route snapshot url
- gateway token
- listener configuration

---

# Observability

## Logs

All logs are produced in **JSON format**.

### Important Fields

- `org_id`
- `agent_id`
- `route_id`
- `conn_id`
- `remote_ip`
- `gateway_id`

---

## Metrics

Example metrics:

- `taj_active_agents`
- `taj_open_streams`
- `taj_bytes_in_total`
- `taj_bytes_out_total`
- `taj_auth_fail_total`

---

# Deployment

## Control

- FastAPI service
- PostgreSQL
- migrations via Alembic

---

## Gateway

- A **VPS with a public IP**
- **TLS certificate**
- **Firewall rules** for public ports

---

## Agent

- Install **binary**
- Provide **config file**
- Run as a **systemd service**

---

# Roadmap

## v0.1

- TCP tunneling
- Agent registration
- Gateway routing
- Policy enforcement

## v0.2

- HTTP routing
- wildcard domains
- automatic TLS (ACME)

## v0.3

- custom domains
- certificate management

## v1.0

- quota and usage metering
- minimal web panel
- security hardening

---

# Status

The project is currently in the **specification and architecture design phase**, and the implementation of **v0.1** is in progress.

# فارسی 🇮🇷

# پروژه تاج — پلتفرم تونل‌سازی TCP چندمستاجری

**پروژه تاج** یک پلتفرم **تونل‌سازی TCP چندمستاجری** است که امکان دسترسی امن به سرویس‌هایی که پشت **NAT یا دیواره آتش** قرار دارند را فراهم می‌کند.

سیستم با استفاده از یک **عامل خروجی (Agent outbound)**، یک **درگاه عمومی (Gateway)** و یک **صفحه کنترل (Control Plane)** ارتباطی پایدار ایجاد می‌کند که از طریق آن کاربران می‌توانند به سرویس‌های داخلی مانند **SSH، پایگاه داده یا هر سرویس TCP دیگر** متصل شوند.

هدف نسخه **v0** ارائه یک **تونل TCP پایدار، امن و ساده** با معماری قابل توسعه است.



# معماری

سیستم از **سه لایه اصلی** تشکیل شده است.


## صفحه کنترل (Control Plane)

سرویس مدیریتی که مسئول موارد زیر است:

- مدیریت سازمان‌ها (**چندمستاجری**)
- مدیریت **عامل‌ها (Agentها)**
- تعریف **تونل** و **مسیر (Route)**
- صدور **توکن نشست (Session JWT)** برای Agent
- ارائه **JWKS** برای اعتبارسنجی توکن‌ها
- ارائه **نمای لحظه‌ای مسیرها (Route Snapshot)** به Gateway

### فناوری‌ها

- FastAPI
- PostgreSQL
- مهاجرت‌های Alembic



## درگاه (Gateway — لایه داده)

Gateway مسئول عبور **ترافیک واقعی** است.

### وظایف

- پذیرش اتصال کاربران روی **پورت‌های عمومی**
- نگهداری اتصال **WebSocket با Agent**
- **چندجریانی‌سازی (Multiplex) استریم‌ها با yamux**
- **فوروارد کردن ترافیک TCP**
- اعمال **سیاست‌ها (Policy)**

### ویژگی مهم

Gateway بدون وابستگی مستقیم به پایگاه داده کار می‌کند و اطلاعات **مسیرها** را از **صفحه کنترل** به صورت **نمای لحظه‌ای (Snapshot)** دریافت می‌کند.



## عامل (Agent)

Agent یک سرویس سبک است که روی **ماشین کاربر** اجرا می‌شود.

### ویژگی‌ها

- اتصال **خروجی به Gateway** (مناسب برای NAT)
- احراز هویت با **توکن دستگاه (Device Token)**
- دریافت **توکن نشست (Session JWT) از Control**
- ثبت **مسیرها**
- اتصال به **سرویس‌های محلی** و فوروارد ترافیک

Agent به زبان **Go** پیاده‌سازی می‌شود.

---

# جریان کلی سیستم

1. Agent با `device_token` از Control یک **توکن نشست (Session JWT)** می‌گیرد.
2. Agent به Gateway از طریق **WebSocket روی TLS** متصل می‌شود.
3. بعد از اتصال، یک **نشست yamux** ایجاد می‌شود.
4. Agent **مسیرهای خود** را ثبت می‌کند.
5. کاربر به `gateway_ip:public_port` وصل می‌شود.
6. Gateway یک **استریم جدید روی yamux** باز می‌کند.
7. Agent اتصال را به **سرویس محلی** برقرار می‌کند.
8. ترافیک **TCP دوطرفه** منتقل می‌شود.

---

# پروتکل

پروتکل ارتباطی بین **Agent و Gateway** با نام **TAJ/1** تعریف شده است.

### لایه انتقال
WebSocket روی TLS

### چندجریانی
yamux

پس از اتصال، Agent یک **استریم کنترلی (Control Stream)** ایجاد می‌کند که پیام‌های کنترلی **JSON** روی آن رد و بدل می‌شوند.

### مراحل دست‌دهی (Handshake)

1. اتصال WebSocket
2. نشست yamux
3. HELLO
4. AUTH
5. REGISTER
6. HEARTBEAT

---

# مدل امنیتی

امنیت سیستم بر اساس دو مکانیزم است.

---

## توکن دستگاه (Device Token)

- یک **توکن بلندمدت برای هر Agent**
- فقط **یک بار نمایش داده می‌شود**
- نسخه **هش‌شده در پایگاه داده ذخیره می‌شود**
- قابل **چرخش (Rotate)** و **ابطال (Revoke)**

---

## توکن نشست (Session JWT)

Agent با استفاده از **Device Token** یک **JWT کوتاه‌مدت** دریافت می‌کند.

Gateway این توکن را با **JWKS** اعتبارسنجی می‌کند.

### ادعاهای اصلی (Claims)

- `iss`
- `aud`
- `sub (agent_id)`
- `org_id`
- `exp`
- `scope`

---

# مدل داده (ساده‌شده)

جداول اصلی سیستم:

- orgs
- users
- agents
- tunnels
- routes
- policies
- audit_events
- sessions

### ویژگی مهم

تمام جداول دارای `org_id` هستند تا **جداسازی چندمستاجری** تضمین شود.

---

# جریان تونل TCP

زمانی که یک کاربر به یک **پورت عمومی Gateway** متصل می‌شود:

1. Gateway **مسیر را تعیین (Resolve)** می‌کند
2. **سیاست‌ها (Policy)** اعمال می‌شوند
3. یک **استریم yamux جدید** ایجاد می‌شود
4. یک **هدر باینری** ارسال می‌شود

### فرمت هدر

MAGIC(4) + ROUTE_UUID(16) + CONN_ID(4)

Agent این هدر را می‌خواند، **مسیر مناسب را انتخاب می‌کند** و به **سرویس محلی** متصل می‌شود.

---

# پیکربندی

## Agent

فایل:

taj-agent.yml

موارد اصلی:

- شناسه Agent
- شناسه سازمان
- توکن دستگاه
- نشانی Gateway
- نشانی Control
- تعریف مسیرها

---

## Gateway

فایل:

taj-gateway.yml

موارد اصلی:

- شناسه Gateway
- نشانی گوش‌دادن WebSocket
- نشانی JWKS
- نشانی دریافت Snapshot مسیرها
- توکن Gateway
- پیکربندی Listener

---

# پایش‌پذیری (Observability)

## لاگ‌ها

تمام لاگ‌ها به صورت **JSON** تولید می‌شوند.

### فیلدهای مهم

- `org_id`
- `agent_id`
- `route_id`
- `conn_id`
- `remote_ip`
- `gateway_id`

---

## متریک‌ها

نمونه متریک‌ها:

- `taj_active_agents`
- `taj_open_streams`
- `taj_bytes_in_total`
- `taj_bytes_out_total`
- `taj_auth_fail_total`

---

# استقرار

## Control

- سرویس FastAPI
- پایگاه داده PostgreSQL
- اجرای migration با Alembic

---

## Gateway

- یک **VPS با IP عمومی**
- **گواهی TLS**
- **قوانین دیواره آتش** برای پورت‌های عمومی

---

## Agent

- نصب **فایل اجرایی (Binary)**
- فایل **پیکربندی**
- اجرای **سرویس systemd**

---

# نقشه راه

## v0.1

- تونل TCP
- ثبت Agent
- مسیردهی Gateway
- اعمال Policy

## v0.2

- مسیردهی HTTP
- دامنه‌های wildcard
- TLS خودکار (ACME)

## v0.3

- دامنه سفارشی
- مدیریت گواهی

## v1.0

- سهمیه و اندازه‌گیری مصرف
- پنل وب حداقلی
- تقویت امنیت

---

# وضعیت

این پروژه در حال حاضر در مرحله **مشخصات فنی (Specification) و طراحی معماری** قرار دارد و پیاده‌سازی **v0.1** در دست انجام است.
