# 运维说明

更新、停止或重启本服务时，不要捎带更新或重启各 Remote。反过来，更新某个 Remote 也不要重启本服务。

## 更新

没有进行中的上传，以及没有排队、运行或等待授权的附件任务时再更新：

```bash
git pull --ff-only
npm ci --include=dev
npm run typecheck && npm test
```

Linux：

```bash
sudo systemctl restart ai-remote-upload.service
curl --fail --show-error --unix-socket "$HOME/.local/share/ai-remote/upload.sock" http://localhost/healthz
```

若单元名不是 `ai-remote-upload.service`，换成实际名称。macOS：`sudo launchctl kickstart -k system/<你的 label>`，再做同样的健康检查。

Node 直接跑 TypeScript，没有构建步骤。重启会使新的票据、上传和租约请求短暂失败；Remote 的 HTTP/WebSocket 会话后端不应因此被一起重启。服务恢复后，Remote 的新请求应能重新连接。

健康检查通过本身不等于整个链路已验收。更新后应用实际票据→上传→租约→续期→释放核对一次。

## 状态和日志

Linux：

```bash
systemctl status ai-remote-upload.service
journalctl -u ai-remote-upload.service -n 100 --no-pager
journalctl -u ai-remote-upload.service -f
```

macOS：看运行用户日志目录下配置的标准输出和错误日志。

服务反复重启时先看数据目录权限、socket 路径占用和磁盘余量，不要只盯着某个 Remote 的网页入口。

## 数据与备份

| 变量 | 内容 | 敏感度 |
| --- | --- | --- |
| `AI_REMOTE_UPLOAD_ROOT` | 附件本体、临时 `.part` 和索引库，默认 `~/.local/share/ai-remote/uploads` | 高 |
| `AI_REMOTE_UPLOAD_SOCKET` | Unix socket，默认 `~/.local/share/ai-remote/upload.sock` | 低 |

SQLite 使用 WAL，不要在服务运行时只复制主库文件而漏掉 `-wal`。可靠做法是停止服务后再复制完整数据目录，包括数据库、可能存在的 WAL 文件、`blobs/` 和 `parts/`。目录权限保持 `0700`/`0600`。

备份放在项目源码之外的私有运行位置。优先回退代码和配置；确认数据损坏并明确恢复范围之前，不要用旧备份覆盖切换后新上传的文件。

## 排障

- 纯文本可用、附件明确失败：通常是本服务未启动、socket 路径不一致，或 Remote 与本服务不在同一 Unix 账号。
- `共享上传 socket 已有服务监听`：已有实例占用该 socket。不要再对同一数据目录启动第二个进程。
- `主机可用磁盘空间低于安全门槛`：提高可用空间，或有意识地调整 `AI_REMOTE_UPLOAD_MIN_FREE_MIB`。
- 旧附件租约失败：核对附件 ID 是否仍在当前数据目录，以及 caller / 项目 / 会话绑定是否一致。

隔离测试必须同时换数据根目录和 socket。当前入口在监听 socket 之前就会打开数据库并执行启动清理。

## 回退

协议和数据库结构没有变化时，可停止新进程，把同一单元的入口恢复到保留的旧代码版本，再使用当前数据启动。新旧版本不能同时访问同一份数据目录。不同账号的实例可以独立回退。
