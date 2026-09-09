# AI Remote Upload

本机共享上传服务。给 Codex Remote、Grok Remote、Relayu 这类单用户 Remote 存附件、发票据、管租约。

它是独立进程：不启动模型，不连接浏览器，不读项目白名单，也不持有会话 Worker。调用方先完成自己的登录、Origin、项目和会话校验，再通过本机 Unix socket HTTP 声明绑定信息。

同账号的多个 Remote 可以共用一个实例。不同 Unix 账号各自部署一份，实例之间不共享数据。第一版只提供同机 Unix socket，不是跨主机网络服务。

> 假设部署者是唯一受信任的使用者。不要把一个实例做成公开注册服务。

## 要求

- Linux 或 macOS
- Node.js 24 或更新版本
- 不需要 Codex CLI、Grok CLI、Claude SDK 或任何模型账号

部署说明见 [`docs/deployment.md`](docs/deployment.md)。协议见 [`docs/protocol.md`](docs/protocol.md)。

## 本机试运行

```bash
npm ci --include=dev
npm run typecheck && npm test
npm start
```

默认 Unix socket 是 `~/.local/share/ai-remote/upload.sock`，数据目录是 `~/.local/share/ai-remote/uploads/`。健康检查：

```bash
curl --fail --unix-socket "$HOME/.local/share/ai-remote/upload.sock" http://localhost/healthz
```

隔离验收时必须同时指定独立的数据根目录和 socket，不要指向正在使用的正式目录：

```bash
AI_REMOTE_UPLOAD_ROOT=/absolute/path/to/isolated/uploads \
AI_REMOTE_UPLOAD_SOCKET=/absolute/path/to/isolated/upload.sock \
npm start
```

当前 `src/main.ts` 在监听 socket 之前就会打开数据库并做启动清理。socket 被占用后启动失败，不保证没有碰到数据。

## 环境变量

| 变量 | 作用 |
| --- | --- |
| `AI_REMOTE_UPLOAD_ROOT` | 附件和索引库根目录；默认 `~/.local/share/ai-remote/uploads` |
| `AI_REMOTE_UPLOAD_SOCKET` | Unix socket 路径；默认 `~/.local/share/ai-remote/upload.sock` |
| `AI_REMOTE_UPLOAD_MIN_FREE_MIB` | 可用磁盘低于该值时暂停新上传，默认 `1024`；`0` 表示只保留操作系统报告的可用空间 |
| `XDG_DATA_HOME` | 未设置上面两个路径时，参与默认目录推导 |

真实主机路径和运行数据不要进 git。

## 文档

| 文档 | 内容 |
| --- | --- |
| [`docs/architecture.md`](docs/architecture.md) | 模块、数据流和职责边界 |
| [`docs/protocol.md`](docs/protocol.md) | Unix socket HTTP 协议 |
| [`docs/deployment.md`](docs/deployment.md) | 安装、同账号共用、不同账号多实例 |
| [`docs/operations.md`](docs/operations.md) | 更新、备份、排障和生命周期 |

## 许可证

Apache License 2.0。来源说明见 [`NOTICE`](NOTICE)。
