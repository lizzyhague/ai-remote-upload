# 架构说明

AI Remote Upload 是本机附件存储服务。Remote 网页后端通过 Unix socket 调用它；浏览器和模型进程都不直接连这个 socket。

```text
浏览器
  │ 已认证 WebSocket 申请票据
  │ 同源 POST /attachments/upload 发送原始字节
  ▼
当前 Remote 后端
  │ Unix socket HTTP；流式转交，不整体缓冲
  ▼
ai-remote-upload
  ├── metadata.sqlite
  ├── parts/<随机 ID>.part
  └── blobs/<分片>/<附件 ID>.<安全扩展名>
```

三个 Remote 地位相同：各自负责网页鉴权、上传代理、项目与会话绑定、租约维护及模型输入。每个上传服务实例只负责自己数据目录里的票据、文件落盘、元数据、租约存储与清理。

## 模块

| 文件 | 负责什么 |
| --- | --- |
| `src/main.ts` | 进程入口：打开存储、启动清理、监听 socket、处理退出信号 |
| `src/server.ts` | Unix socket 上的 HTTP API |
| `src/store.ts` | SQLite、落盘、票据、租约和清理 |
| `src/paths.ts` | 按运行账号和环境变量解析数据目录与 socket |
| `src/types.ts` | 协议类型、限制常量和错误形状 |
| `src/client.ts` | 参考客户端，用于协议测试；Remote 可保留各自的薄客户端 |

服务使用 Node 内置模块。不导入任何 Remote、CLI 或模型 SDK。

## 数据

默认路径由运行账号的 `HOME` 或 `XDG_DATA_HOME` 推导：

- 存储根目录：`~/.local/share/ai-remote/uploads/`
- Unix socket：`~/.local/share/ai-remote/upload.sock`
- SQLite：`~/.local/share/ai-remote/uploads/metadata.sqlite`

目录权限是 `0700`，数据库和附件是 `0600`，socket 是 `0600`。因此每个实例与其调用方使用同一个 Unix 账号。不同账号分别部署，不要靠放宽目录或 socket 权限跨账号共享。

同一份程序可以运行多次。每个实例分别持有进程、socket、数据库、blobs、parts、配置与日志。设置 `AI_REMOTE_UPLOAD_ROOT` 或 `AI_REMOTE_UPLOAD_SOCKET` 时，必须保证不同实例的最终数据根目录和 socket 不重合。

## 不做的事

- 不鉴权浏览器，不签发 cookie，不检查 Origin
- 不读取项目白名单，不绑定具体会话 Worker
- 不把文件内容交给模型；路径只在本机租约响应里返回给 Remote 后端
- 不把服务端绝对路径作为公开附件字段
- 不提供 TCP、Tailscale 或公网入口
