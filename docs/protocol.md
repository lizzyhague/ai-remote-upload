# 本机协议

以下 API 只在 Unix socket 上提供。协议版本是 `/v1`。不兼容变化需要另定迁移，不在提取或日常修复里改这些路径和字段的语义。

JSON 错误统一为：

```json
{
  "error": {
    "code": "stable_machine_code",
    "message": "可读说明"
  }
}
```

`caller` 只能是 `codex`、`grok` 或 `claude`。调用方由 Remote 后端填写，不能相信浏览器传来的 caller、项目、会话或服务器路径。

## 健康检查

`GET /healthz` 返回 `{ "status": "ok" }`。健康检查通过不等于整条上传链路已验收。

## 创建票据

`POST /v1/tickets`

```json
{
  "caller": "grok",
  "projectId": "projects/example",
  "sessionId": "session-id",
  "originalName": "screen.png",
  "declaredMime": "image/png",
  "expectedSize": 12345
}
```

返回票据、过期时间和不含路径的公开元数据。票据是 bearer secret，不应写入 URL、日志或持久化浏览器存储。票据有效 10 分钟，只能使用一次；SQLite 只保存票据 SHA-256。

## 上传字节

`POST /v1/uploads`

请求头必须带 `X-Upload-Ticket` 和 `Content-Length`，正文是原始文件字节。返回：

```json
{
  "attachment": {
    "id": "UUID",
    "caller": "grok",
    "projectId": "projects/example",
    "sessionId": "session-id",
    "originalName": "screen.png",
    "declaredMime": "image/png",
    "detectedMime": "image/png",
    "kind": "image",
    "size": 12345,
    "sha256": "...",
    "createdAtMs": 0,
    "expiresAtMs": 0
  }
}
```

这个响应可以转给浏览器，其中没有主机路径。

服务端同时核对票据大小、HTTP `Content-Length` 和实际字节数。单文件最多 25 MiB。上传时使用 `wx` 创建临时文件，边写边计算 SHA-256，完整接收并 `fsync` 后原子重命名。

PNG、JPEG、GIF 和 WebP 通过文件签名识别为图片；PDF 也通过签名识别。其余内容按普通文件处理。客户端声明的 MIME 和扩展名不能单独把文件升级为图片。

## 创建和维护任务租约

`POST /v1/leases` 接收 `caller`、`projectId`、`sessionId`、`ownerId` 和 `attachmentIds`。服务再次核对所有附件的绑定和过期时间，返回租约以及带绝对路径的 `attachments`。一条消息最多引用 100 个附件。

租约响应是本机后端专用数据。绝对路径不能进入浏览器协议、对外 HTTP 响应或公开日志。Remote 固化公开元数据和附件 ID，路径只在启动任务时短暂使用。

- `POST /v1/leases/<leaseId>/renew`，正文 `{ "ownerId": "..." }`
- `POST /v1/leases/<leaseId>/release`，正文 `{ "ownerId": "..." }`

任务租约有效 15 分钟。任务排队、运行和等待审批期间都要保留租约；完成、失败或中断后释放。Remote 崩溃时未释放的租约会自然过期，避免附件永久无法清理。

## 保留和清理

- 完成附件保留 30 天
- `.part` 文件一小时后可清理
- 服务启动时补做清理，运行期间每小时清理
- 过期清理只删没有有效租约的附件
- 定时清理也会删除“文件已重命名、元数据尚未提交”这一崩溃窗口留下的一小时以上孤立 blob
- 默认至少保留 1 GiB 可用磁盘，`AI_REMOTE_UPLOAD_MIN_FREE_MIB` 可以覆盖

以上是共享层限制。各 Remote 可以另有自己的适配限制，例如某条消息的文本合计大小。

## 调用方职责

1. 先核对该 Remote 的真实项目 ID、会话 ID、鉴权和消息幂等机制。
2. 配置与本实例相同的 `AI_REMOTE_UPLOAD_SOCKET`。
3. 在已经认证的 WebSocket 上申请票据；caller 由后端固定。
4. 用当前 Remote 自己的同源上传入口流式转交票据和字节，不先把文件读进内存，也不要为本服务新增公网或 Tailscale 入口。
5. 前端保存并发送附件 ID；发送消息前不得自动提交。
6. 后端接受任务时创建租约并持久化公开附件元数据；启动任务时才使用租约返回的路径。
7. 按所用 coding agent 的真实协议映射附件，并做内容级验收；文件存在不等于模型理解了内容。
