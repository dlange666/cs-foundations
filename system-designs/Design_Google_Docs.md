# Design Google Docs

## 1. Functional Requirements

1. 多用户实时协同编辑（同一文档多人同时输入）。
2. 支持基础富文本能力（段落、加粗、标题、列表）。
3. 文档自动保存与历史版本回溯。
4. 权限控制（owner/editor/viewer）。
5. 在线状态展示（谁在线、光标位置、选区）。
6. 评论与建议模式（可后续扩展）。

## 2. Non-Functional Requirements

1. 低延迟：本地输入即时反馈，远端同步尽量 < 200ms。
2. 高可用：编辑服务可水平扩展，单点故障不影响整体。
3. 高一致性：最终一致，且每个客户端可收敛到同一文档状态。
4. 持久化：操作日志与快照持久化，支持 crash recovery。
5. 安全：鉴权、鉴权后细粒度授权、传输加密、审计日志。

## 3. Capacity Estimation (example)

1. DAU: 10M
2. 并发在线编辑文档: 500K
3. 每个活跃协作者平均 1 op/s（insert/delete/format）
4. 峰值写入操作: 500K op/s

结论：核心瓶颈在实时操作广播与冲突合并，存储更适合“操作日志 + 周期快照”模式。

## 4. High-Level Architecture

```text
Client(Web/Mobile)
  -> API Gateway
    -> Auth Service
    -> Document Metadata Service (doc info + ACL)
    -> Collaboration Service (WebSocket, OT/CRDT engine)
    -> Presence Service (online users, cursor)
    -> Versioning Service (snapshot + history)
    -> Storage Layer
       - Redis (hot doc state / pub-sub)
       - Kafka (operation stream)
       - Object Store / DB (snapshot + metadata)
```

### Data Flow

1. 客户端本地编辑生成 operation（op），先本地应用（optimistic UI）。
2. op 通过 WebSocket 发到 Collaboration Service。
3. 服务端做变换与排序（OT transform 或 CRDT merge）后广播给其他协作者。
4. op 追加写入 Kafka，再异步落盘。
5. 周期性生成 snapshot，减少重放成本。

## 5. Core Data Model

### Document

```text
doc_id
title
owner_id
acl(versioned)
latest_snapshot_id
created_at, updated_at
```

### Operation Log

```text
op_id
doc_id
user_id
base_version
op_type (insert/delete/format)
payload (position, text, attributes...)
client_ts, server_ts
```

### Snapshot

```text
snapshot_id
doc_id
version
content_blob
created_at
```

## 6. Consistency Strategy: OT vs CRDT

### OT (Operational Transform)

优点：
1. 工程上在中心化服务模型中较成熟。
2. 对文本编辑场景高效，消息体通常较小。

挑战：
1. transform 函数组合复杂，正确性验证成本高。
2. 离线/弱网恢复与跨版本重放逻辑复杂。

### CRDT (Conflict-free Replicated Data Type)

优点：
1. 数学上更容易保证最终收敛。
2. 对离线编辑与多副本更友好。

挑战：
1. 元数据开销更高，内存与网络成本可能更大。
2. 实现文本 CRDT 时，索引与压缩策略较复杂。

### 选型建议

1. 先做中心化实时协同（类似 Google Docs 的在线模式）：优先 OT。
2. 强离线优先、多端长时间分叉：考虑 CRDT。

## 7. Partitioning & Scaling

1. 按 `doc_id` 做一致性哈希分片，让同一文档会话尽量落在同一协同节点。
2. 使用 sticky routing，减少跨节点状态同步。
3. 热点大文档可拆分为 segment（段落/块级），分块并行同步。
4. 广播采用 pub-sub（Redis/NATS/Kafka）支持跨实例 fan-out。

## 8. Reliability & Recovery

1. 写前日志（WAL）/消息日志先持久化再确认，防止 op 丢失。
2. 客户端维护未确认 op 队列，超时重试 + 去重（op_id 幂等）。
3. 周期 snapshot + operation compaction，控制恢复时间。
4. 文档级限流与背压，避免单热点拖垮集群。

## 9. Security

1. JWT/OAuth2 鉴权，服务端二次校验 ACL。
2. 文档分享链接支持过期时间和权限范围。
3. 全链路 TLS，加密存储敏感元数据。
4. 审计日志记录编辑、分享、权限变更。

## 10. API Sketch

### HTTP

1. `POST /docs` 创建文档
2. `GET /docs/{docId}` 获取元信息和最近快照
3. `POST /docs/{docId}/share` 设置分享权限

### WebSocket

1. `join_doc(docId, token)`
2. `submit_op(op)`
3. `ack(op_id, version)`
4. `remote_op(op)`
5. `presence_update(user, cursor)`

## 11. Interview Deep-Dive Points

1. 如何保证“本地即时输入”和“全局一致”同时成立？
2. 大文档重连后如何快速恢复（snapshot + incremental ops）？
3. 如何做版本历史与“命名版本”？
4. OT transform 正确性如何测试（property-based tests）？
5. CRDT 元数据膨胀如何压缩？

## 12. Notes

1. OT: Operational Transform
2. CRDT: Conflict-free Replicated Data Type
