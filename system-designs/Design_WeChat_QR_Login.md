# System Design: 微信扫码登录

## 1. 目标与范围

目标：用户在 PC Web 端通过微信扫码完成登录，做到低延迟、安全、可追踪、可扩展。

范围：
1. 覆盖“展示二维码 -> 手机确认 -> Web 登录成功”的主链路。
2. 覆盖账号绑定、幂等、重试、过期、风控。
3. 不展开微信开放平台申请流程与 UI 细节。

## 2. Functional Requirements

1. 未登录用户访问 Web 时可看到微信登录二维码。
2. 用户用微信扫码后，Web 端可实时感知状态变化（已扫码/已确认/已取消/已过期）。
3. 用户在手机端确认后，Web 端完成登录并建立站内会话。
4. 首次登录支持“微信 openid/unionid 与站内账号绑定”。
5. 二维码支持过期与刷新。
6. 全链路可观测（日志、指标、审计）。

## 3. Non-Functional Requirements

1. 安全性：防重放、防 CSRF、防 token 泄漏、防会话劫持。
2. 可用性：登录核心链路高可用，局部故障可降级。
3. 时延：扫码状态更新尽量 < 1s 到达 Web。
4. 一致性：同一二维码只允许成功登录一次（exactly-once 语义通过幂等实现）。

## 4. High-Level Architecture

```text
Web Client
  -> API Gateway
    -> Auth Service
    -> QR Session Service
    -> WeChat OAuth Adapter
    -> Account Binding Service
    -> Session Service
    -> Risk Control Service
    -> Redis (qr state / nonce / short ttl)
    -> DB (user, binding, audit)
```

组件职责：
1. `QR Session Service`：创建二维码会话、维护状态机、过期控制。
2. `WeChat OAuth Adapter`：封装微信接口调用（换 token、取用户标识）。
3. `Account Binding Service`：处理首次绑定与账号查找。
4. `Session Service`：签发站内 session/JWT。
5. `Risk Control Service`：频控、设备指纹、IP 异常检测。

## 5. Core Data Model

### qr_login_session

```text
session_id (uuid)
state (INIT | SCANNED | CONFIRMED | CANCELED | EXPIRED | COMPLETED)
qr_nonce (random)
scene_token (one-time)
client_id (web device id)
redirect_uri
expires_at
wechat_openid (nullable)
wechat_unionid (nullable)
user_id (nullable)
version (for CAS)
created_at, updated_at
```

### user_wechat_binding

```text
id
user_id
unionid
openid
app_id
status
created_at, updated_at
UNIQUE (app_id, unionid)
UNIQUE (app_id, openid)
```

### login_audit_log

```text
event_id
session_id
event_type
ip
user_agent
device_id
result
created_at
```

## 6. 状态机设计

```text
INIT -> SCANNED -> CONFIRMED -> COMPLETED
  |        |           |
  |        -> CANCELED |
  -> EXPIRED <---------
```

状态约束：
1. `EXPIRED`/`COMPLETED` 为终态，不可逆。
2. 任意状态迁移必须带版本号（CAS），防并发覆盖。
3. 同一 `session_id` 只能成功一次，重复确认返回幂等成功或已失效。

## 7. 时序流程

## 7.1 生成二维码

1. Web 调 `POST /auth/wechat/qr-sessions`。
2. 服务端生成 `session_id + scene_token + nonce`，写 Redis（TTL 120s）。
3. 返回二维码内容（建议仅包含短期 `scene_token`，不暴露内部主键）。

## 7.2 手机扫码与确认

1. 用户微信扫码，微信回调或跳转到业务确认页。
2. 后端校验 `scene_token` 有效期与签名，状态置 `SCANNED`。
3. 用户点击“确认登录”，后端置 `CONFIRMED` 并通过微信接口换取 `openid/unionid`。

## 7.3 Web 实时拿结果

两种实现：
1. WebSocket/SSE（推荐）：服务端推送状态变更。
2. 轮询：`GET /auth/wechat/qr-sessions/{id}/status` 每 1-2s 拉取。

当状态 `CONFIRMED`：
1. 查绑定表命中则取 `user_id`。
2. 未命中则走注册/绑定流程。
3. 签发站内 session（HttpOnly + Secure + SameSite）。
4. 状态更新为 `COMPLETED`。

## 8. Security Design

1. 二维码票据一次性 + 短 TTL（如 120s）。
2. `scene_token` 使用 HMAC/JWS 签名，防篡改。
3. 会话 cookie 仅走 HTTPS，开启 `HttpOnly` 和 `SameSite=Lax/Strict`。
4. 登录确认需绑定扫码设备上下文（device/ip 风险校验）。
5. 回调接口做来源校验、timestamp + nonce 去重，防重放。
6. 全链路审计日志，便于风控追查。

## 9. 风控与反滥用

1. 按 IP/device/account 维度限流。
2. 异地异常登录触发二次验证（短信/邮箱/站内确认）。
3. 高频失败二维码会话自动熔断或延迟响应。
4. 黑名单（设备指纹/IP 段）动态拦截。

## 10. 扩展性与高可用

1. QR 会话存 Redis，按 `session_id` 分片。
2. 状态变更通过消息总线广播，支持多实例推送。
3. 关键写操作落库前后都要幂等（`request_id`）。
4. 失败补偿：确认成功但回写失败时，靠异步任务重放审计日志修复状态。

## 11. API Sketch

### HTTP

1. `POST /auth/wechat/qr-sessions` 创建二维码会话
2. `GET /auth/wechat/qr-sessions/{sessionId}/status` 查询状态
3. `POST /auth/wechat/callback/scan` 微信扫码回调
4. `POST /auth/wechat/callback/confirm` 微信确认回调
5. `POST /auth/wechat/bind` 首次绑定账号

### Event

1. `qr.scanned`
2. `qr.confirmed`
3. `qr.completed`
4. `qr.expired`

## 12. 失败场景与处理

1. 二维码过期：返回 `EXPIRED`，前端自动刷新二维码。
2. 用户取消确认：返回 `CANCELED`，前端提示重试。
3. 回调重复：基于 `session_id + event_type + request_id` 幂等去重。
4. 微信接口超时：短重试 + 熔断 + 降级提示。
5. 绑定冲突（unionid 已绑定其他账号）：拒绝并触发安全告警。

## 13. 面试可深挖点

1. 如何保证“只登录一次”？（状态机 + CAS + 幂等键）
2. 为什么二维码里不直接放 user_id？
3. WebSocket 与轮询如何取舍？
4. Redis 丢数据后如何恢复会话一致性？
5. 如何设计跨端登录风控策略？
