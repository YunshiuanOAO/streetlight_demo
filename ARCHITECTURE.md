# Streetlight 架構（蜜罐部署版）

合併後的拓撲，全部 user-facing API 集中在 subscriber:8080：

```
   ┌────────────────┐
   │ 前端 / 蜜罐訪客 │
   └────────┬───────┘
            │ HTTP :8080  (UI、API、/.env bait)
            ▼
   ┌────────────────────────────────────────────┐               ┌──────────────┐
   │              subscriber                    │               │   Gateway    │
   │  - Flask + Socket.IO HMI                   │               │              │
   │  - POST /api/commands  (前端下指令)        │               │  listen :15566 ◀── server dial
   │  - POST /api/messages  (socket_server push)│               │  dial :15567 ──▶ server listen
   │  - GET  /api/runtime / /api/command_logs   │               │  dial :1883  ──▶ mqtt
   │  - /.env (honeypot bait)                   │               └──────────────┘
   │                                            │
   │  pub_client → mosquitto :1883 (proxy)      │
   │  sub_client → mosquitto :11883 (real)      │
   │  read/write → MongoDB :27017               │
   │  POST commands → socket_server :19100      │
   └────────────┬───────────┬───────────┬───────┘
                │           │           │
                ▼           ▼           ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────────────────┐
   │  Mosquitto   │ │   MongoDB    │ │           socket_server              │
   │  :1883/:11883│ │   :27017     │ │  - listen :15567 (gateway upstream)  │
   │              │ │   (auth)     │ │  - listen :19100 (HTTP /api/commands)│
   │              │ │              │ │  - dial   <gateway>:15566 (downstream│
   └──────────────┘ └──────────────┘ │  - read/write MongoDB                │
                                     │  - forward upstream → subscriber     │
                                     └──────────────────────────────────────┘
```

## 服務 / 角色

| 元件 | 對外 port | 角色 |
|------|----------|------|
| `subscriber/` | **8080** (host) | 蜜罐 HMI + 唯一 user-facing API + .env bait |
| `socket_server/` | 15567 (TCP up), 19100 (HTTP) | TCP Management；對 gateway dial 出 15566；對 mongo 讀寫 |
| `gateway/` | 15566 (TCP down listen) | 路燈閘道模擬器；listen 下行 + dial 上行 |
| `mongodb/` | 27017（內網）| 持久化 `command_logs` / `runtime`，含 auth |
| `mosquitto/` | 1883 (proxy) / 11883 (real) | MQTT broker + honeypot proxy |

## TCP 對等雙通道（gateway ↔ socket_server）

兩邊各 listen + 互相 dial：

| 通道 | listen 在 | dial 進來 | role 握手 | 用途 |
|------|----------|----------|----------|------|
| 下行 | gateway:15566 | socket_server | `ROLE manager\n` | server → gateway 推指令 |
| 上行 | socket_server:15567 | gateway | `ROLE gateway\n` | gateway → server 送回包 |

每筆 wire 是一行 hex 字串 + `\n`。Heartbeat 兩邊都送（30s 一次空白行）。

## 用戶下指令的完整流程

```
前端 ──POST /api/commands {"data":"<hex>"}──▶ subscriber:8080
                                                   │
                                                   ├─ 寫 mongo: command_logs(stage=accepted, source=frontend)
                                                   │
                                                   └─ HTTP POST → socket_server:19100/api/commands
                                                                       │
                                                                       ├─ 寫 mongo: command_logs(source=subscriber)
                                                                       │
                                                                       └─ 從已 dial 的下行 sockets 廣播 hex
                                                                                │
                                                                                ▼
                                                                          gateway:15566 reads
                                                                                │
                                                                  在 COMMAND_RESPONSE_MAP 找對應
                                                                                │
                                                                                ▼
                                                                  send_upstream(response_hex)
                                                                                │
                                                                                ▼
                                                                  socket_server:15567 reads
                                                                                │
                                                                                ├─ 寫 mongo: command_logs(source=gateway)
                                                                                ├─ upsert mongo: runtime[mac]
                                                                                └─ POST subscriber:/api/messages
                                                                                                  │
                                                                                                  └─ Socket.IO emit → UI
```

## DRD_10 + ACK 自動鏈

```
gateway 每 N 秒 → send_upstream(DRD_10) → socket_server → 寫 mongo + POST subscriber/api/messages
                                                                          │
                                                                          └─ subscriber.maybe_ack_drd10()
                                                                                ├─ 算 SRD_10 ack_hex
                                                                                └─ POST socket_server:19100/api/commands
                                                                                              │
                                                                                              └─ 寫 mongo + 從下行 socket 推回 gateway
gateway:15566 reads SRD_10 → cmd_byte=='81' → 直接 return（不再 ACK ACK）
```

## MongoDB Schema

### `runtime`（每 MAC 一筆，upsert）
```json
{
  "mac": "123456789A",
  "last_hex": "021C80...",
  "last_command": "DRD_10 (定時回報)",
  "last_cmd_byte": "80",
  "last_drd10_hex": "021C80...",
  "last_drd10_at": "2026-05-06T05:20:10",
  "updated_at": "2026-05-06T05:20:10"
}
```

### `command_logs`（append-only）
```json
{
  "ts": "2026-05-06T05:18:54",
  "destination": "gateway" | "server",
  "source": "frontend" | "subscriber" | "gateway",
  "hex_data": "020B52123456789A013240",
  "command": "SPW (單盞控)",
  "cmd_byte": "52",
  "mac": "123456789A",
  "stage": "accepted"   // 只在 subscriber 入口寫
}
```

## Honeypot 蜜罐誘餌（刻意洩漏）

| 弱點 | 路徑 | 攻擊者得到 | 蒐集到的 IoC |
|------|------|-----------|------------|
| `.env` 暴露 | `:8080/.env`, `/env`, `/.env.bak`, `/config.env`, `/admin/.env`, `/.env.production` | mongo creds、`SECRET_KEY`、admin/admin、broker host | `${HONEYPOT_LOGS_DIR}/envleak.log` 含 timestamp + peer IP（`X-Forwarded-For`）+ User-Agent |
| 弱口令 | `/login` admin/admin | 工控 HMI 登入、看路燈、下指令 | Flask access log |
| MongoDB | 27017 + 偷到的 creds | 全 R/W、dump、刪 collection | mongod log |
| MQTT proxy | 1883 (proxy → 11883) | publish 假指令到 `streetlight/server_to_gateway` | proxy log |
| TCP downstream proxy | 5566 (proxy → gateway:15566) | 直接送 hex 給 gateway | proxy log |

> 千萬不要在 PR 裡把 `/.env` 路徑改成 404，那會把蜜罐的眼睛挖掉。

## 服務設定速查

### gateway/ (`gateway.env`)
```env
COMM_MODE=both
SOCKET_HOST=<VM-B host>           # 上行通道目標
UPSTREAM_PORT=15567
GATEWAY_LISTEN_HOST=0.0.0.0       # 下行通道：自己 listen 等 server dial
GATEWAY_LISTEN_PORT=15566
BROKER_HOST=<MQTT broker>
BROKER_PORT=1883
GATEWAY_REPORT_INTERVAL=300
```

### socket_server/ (`socket_server.env`)
```env
UPSTREAM_PORT=15567
HTTP_PORT=19100
GATEWAYS=<gateway-host>:15566       # 下行 dial 目標（多台逗號隔開）
SUBSCRIBER_API_URL=http://<subscriber-host>:8080/api/messages
MONGO_URI=mongodb://streetlight_admin:Hn8KQZ3vR6tY9xA2@mongo:27017/streetlight?authSource=admin
MONGO_DB=streetlight
```

### mongodb/ (`mongodb.env`)
```env
MONGO_DB=streetlight
MONGO_PORT=27017
MONGO_USER=streetlight_admin
MONGO_PASS=Hn8KQZ3vR6tY9xA2
```

### subscriber/ (`subscriber.env`)
```env
COMM_MODE=both
SOCKET_SERVER_HOST=<VM-B host>
SOCKET_SERVER_HTTP_PORT=19100
BROKER_HOST=<MQTT broker>
BROKER_SUB_PORT=11883
BROKER_PUB_PORT=1883
MONGO_URI=mongodb://streetlight_admin:Hn8KQZ3vR6tY9xA2@<VM-B>:27017/streetlight?authSource=admin
MONGO_DB=streetlight
ADMIN_USER=admin
ADMIN_PASS=admin
SECRET_KEY=<random>
```
> 此檔案會被 honeypot 路徑 `/.env` 對外洩漏 — 內容刻意設成弱口令。

## 部署順序

VM-B（broker / DB / TCP / Web）：
1. `docker network create streetlight-vmb`
2. `cd mongodb && docker compose up -d`
3. `cd ../socket_server && docker compose up -d`
4. `cd ../mosquitto && docker compose up -d`
5. `cd ../subscriber && docker compose up -d`

Gateway（任一台機器或 IoT 邊緣）：
1. `cd gateway && docker compose up -d`
