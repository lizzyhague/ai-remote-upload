# 部署说明

服务只通过本机 Unix socket 提供 HTTP API，不监听 TCP。先在目标账号下把进程跑通，再让各 Remote 指向这个 socket。

真实主机路径和 `deploy/*.env`、`deploy/*.local` 不提交。

## 1. 准备

服务由**将要使用附件的那个 Unix 用户**运行。附件文件和 socket 权限是该用户私有的，对应 Remote 和 coding agent 也必须是同一账号。用专用服务账号也可以，但要在该账号下单独部署 Remote，不要靠放宽数据目录权限跨账号共享。

在该用户的登录环境中核验（要求 Node.js 24 以上）：

```bash
id -un; id -gn; printf '%s\n' "$HOME"
node --version
command -v node
```

运行用户的 HOME 用于推导默认数据目录，所以不能给单元加会挡住 HOME 的限制。

## 2. 装代码和配置

```bash
git clone https://github.com/lizzyhague/ai-remote-upload.git
cd ai-remote-upload
npm ci --include=dev
npm run typecheck && npm test

cp deploy/ai-remote-upload.env.example deploy/ai-remote-upload.env
chmod 600 deploy/ai-remote-upload.env
```

同账号共用默认路径时，环境文件可以保持注释状态。不同实例同时跑在同一账号下时，必须给每个实例设置互不重合的 `AI_REMOTE_UPLOAD_ROOT` 和 `AI_REMOTE_UPLOAD_SOCKET`。

## 3. 两种用法

### 同账号共用一个实例

同一 Unix 账号上的多个 Remote 把 `AI_REMOTE_UPLOAD_SOCKET` 指到同一个 socket。Linux 单元名可用 `ai-remote-upload.service`；macOS Label 可用一个固定值，例如 `io.example.ai-remote-upload`。

```text
codex-remote ──┐
grok-remote ──┼── Unix socket HTTP ── 本实例
relayu ───────┘
```

### 不同账号各自一个实例

同一套程序安装运行两次。每个账号有自己的进程、socket、数据目录、配置和日志。账号之间不互访附件。

```text
账号 A：某个 Remote ── socket A ── 本程序实例 A ── 数据目录 A
账号 B：某个 Remote ── socket B ── 本程序实例 B ── 数据目录 B
```

同一主机上的系统级单元名、launchd Label 和 plist 安装文件名必须能区分实例。macOS 继续用系统级 LaunchDaemon，分别填写运行用户与其 HOME，不要为了多实例改启动域或互相改账号。

## 4a. Linux + systemd

```bash
cp deploy/ai-remote-upload.service.example deploy/ai-remote-upload.service.local
```

替换本地副本里的占位符：`__RUN_USER__`、`__RUN_GROUP__`、`__RUN_HOME__`、`__APP_DIR__`（本仓库根目录）、`__UPLOAD_ENV_FILE__`、`__NODE_BIN__`、`__RUNTIME_PATH__`（含 Node 的完整 PATH）。

```bash
grep -n '__[A-Z_]*__' deploy/*.service.local   # 应无输出
sudo install -m 0644 deploy/ai-remote-upload.service.local /etc/systemd/system/ai-remote-upload.service
sudo systemctl daemon-reload
sudo systemctl enable --now ai-remote-upload.service
curl --fail --show-error --unix-socket "$HOME/.local/share/ai-remote/upload.sock" http://localhost/healthz
```

同一主机的另一个账号应安装成不同单元名，例如 `ai-remote-upload-alice.service`。各 Remote 可以对实际连接的上传实例声明 `Wants=` / `After=`，不要用 `PartOf=` / `BindsTo=` 把 Remote 的生死绑到本服务。

## 4b. macOS + launchd

用系统级 LaunchDaemon，服务不依赖图形登录；`UserName` 仍是上面那个普通用户。

```bash
cp deploy/launchd/ai-remote-upload.plist.example deploy/launchd/ai-remote-upload.plist.local
mkdir -p "$HOME/Library/Logs/ai-remote-upload" && chmod 700 "$HOME/Library/Logs/ai-remote-upload"
```

占位符比 systemd 多：`__UPLOAD_SERVICE_LABEL__`（每个实例用不同的 launchd label）、`__LOG_DIR__`、`__LOG_BASENAME__`（日志文件名，不同实例不要互相覆盖）。文件名、plist 里的 `Label` 和 `launchctl` 命令三者必须一致；下面的 `io.example.*` 只是例子。

```bash
grep -n '__[A-Z_]*__' deploy/launchd/*.plist.local   # 应无输出
plutil -lint deploy/launchd/*.plist.local
sudo install -o root -g wheel -m 0644 deploy/launchd/ai-remote-upload.plist.local \
  /Library/LaunchDaemons/io.example.ai-remote-upload.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/io.example.ai-remote-upload.plist
curl --fail --show-error --unix-socket "$HOME/.local/share/ai-remote/upload.sock" http://localhost/healthz
```

健康检查不过就不要让 Remote 依赖这个实例。

## 5. 让 Remote 连上

各 Remote 设置 `AI_REMOTE_UPLOAD_SOCKET` 指向这个实例。未部署或临时不可用时，附件操作应明确失败，纯文本会话保持可用。

从旧的 Codex Remote 仓库入口切换到本项目时：保留服务名、账号、socket 和数据目录，只改工作目录和 `ExecStart` / `ProgramArguments` 指向 `src/main.ts`。切换前要有完整 runbook；同一份正式数据目录同一时刻只能有一个实例访问。
