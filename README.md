# Uptime

[English](README.en-US.md) | 简体中文

Uptime 是一套使用 Go 编写的轻量级、自托管 Linux 监控系统。它包含服务端（`uptimed`）、客户端（`uptime-agent`），以及一个基于 Chart.js 的响应式 Web 仪表盘。

## 功能特性

- 监控 CPU、内存、磁盘、网络、进程、连接数、系统负载和运行时间
- 提供最新状态、紧凑的二进制历史记录、图表和客户端离线队列
- 使用 HMAC-SHA256 身份验证，并通过时间戳和 nonce 防止重放攻击
- 支持 Gotify 告警、systemd 服务和 `/healthz` 健康检查
- 支持 Linux amd64/arm64，无需数据库或 Node.js 服务端运行时
- 客户端不监听端口，也不提供远程 Shell 或命令执行接口

## 安装

请以 root 用户运行。默认情况下，安装脚本会从 **最新的 GitHub Release** 下载与系统架构匹配的二进制文件，并使用该 Release 中的 `SHA256SUMS` 校验文件完整性。

安装服务端：

```sh
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-server.sh | sh
```

添加节点并重启服务端：

```sh
su -s /bin/sh uptime -c '/usr/local/bin/uptimed -config /etc/uptime/server.json -add-node "US VPS 01"'
systemctl restart uptimed
```

使用注册码安装客户端：

```sh
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-agent.sh | sh -s -- https://status.example.com
```

安装脚本会以不回显的方式读取注册码，避免其进入 Shell 历史或作为进程参数暴露。无人值守安装时，请将注册码写入仅 root 可读的文件，并设置 `REGISTRATION_CODE_FILE=/path/to/code`。

默认设置：

- 服务端监听地址：`127.0.0.1:62369`
- 服务端数据目录：`/opt/uptime`
- 客户端离线队列：`/var/lib/uptime-agent/spool.jsonl`
- 客户端上报间隔：60 秒

自定义安装示例：

```sh
# 服务端端口
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-server.sh | PORT=8080 sh

# 服务端数据目录
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-server.sh | DATA_DIR=/srv/uptime-data sh

# 客户端离线队列目录
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-agent.sh | DATA_DIR=/srv/uptime-agent-data sh -s -- https://status.example.com
```

生产环境请使用 HTTPS 反向代理。仅当目标明确使用 `127.0.0.1` 或 `::1` 时，客户端才允许使用明文 HTTP 连接。请妥善保管注册码，避免泄露。

## 更新

再次运行安装脚本时，脚本会在更新前请求确认，并保留现有配置、服务端历史记录和客户端离线队列。无人值守更新可使用：

```sh
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-server.sh | REINSTALL=1 sh
curl --proto '=https' --proto-redir '=https' --tlsv1.2 -fsSL https://github.com/marklee369/uptime_go/releases/latest/download/install-agent.sh | REINSTALL=1 sh
```

默认的 `VERSION=latest` 会跟随最新 Release，并不会固定到某个特定版本。仅在需要明确指定版本时，才将 `VERSION` 设置为对应的 Release 标签。

## 配置

配置示例位于 [`config.example.json`](config.example.json)、[`agent.example.json`](agent.example.json) 和 [`systemd/`](systemd/) 中。

- 服务端配置：`/etc/uptime/server.json`，权限模式为 `0600`
- 客户端配置：`/etc/uptime/agent.json`，权限模式为 `0600`
- `offline_after_seconds`：5–86400 秒，默认值为 180
- `history_days`：历史记录保留天数
- `trusted_proxies`：允许提供 `CF-Connecting-IP`、`X-Real-IP` 或 `X-Forwarded-For` 的代理 IP/CIDR 白名单；反向代理必须覆盖这些请求头
- `country_code`：可选，用于手动指定国家或地区代码
- `CF-IPCountry`：请求经过 Cloudflare 时会自动使用该请求头

请勿让多个服务端实例共用同一个数据目录。修改服务端配置或添加节点后，请重启 `uptimed`。

## Gotify 告警

在 `server.json` 中配置 `gotify.url`、`gotify.token` 和 `gotify.priority`。除回环地址测试外，Gotify 必须使用 HTTPS。

以下告警会自动去重：

- CPU 使用率连续 10 分钟大于或等于 50%
- 内存使用率大于或等于 90%
- 上传速度连续 24 小时高于配置的阈值
- 节点离线时间超过 `offline_after_seconds`

可使用 `alerts.disable_upload` 关闭上传告警，或通过 `alerts.upload_min_mib_per_second` 调整上传速度阈值。

## 构建与发布

需要 Go 1.24 或更高版本：

```sh
make test
make build
```

推送符合语义化版本规范的标签后，工作流会运行测试，并发布带校验和的 Linux amd64/arm64 二进制文件：

```sh
git tag v1.4.0
git push origin v1.4.0
```

Release 资源会发布到公开仓库 [`marklee369/uptime_go`](https://github.com/marklee369/uptime_go)，发布工作流不会覆盖已有资源。请使用同一 Release 中的 `SHA256SUMS` 校验下载的二进制文件和安装脚本。

私有源码仓库需要配置名为 `UPTIME_GO_TOKEN` 的 Actions Secret。建议使用仅对 `marklee369/uptime_go` 具有 **Contents: Read and write** 权限的细粒度令牌，并确保公开仓库已经初始化默认分支。

## API 与安全

- `POST /api/v1/ingest` — 接收经过签名的客户端报告
- `GET /api/v1/overview` — 获取公开节点的最新状态
- `GET /api/v1/nodes/{id}/history?since=UNIX&until=UNIX` — 获取节点历史记录
- `GET /api/v1/status` — 获取监控服务端状态
- `GET /healthz` — 健康检查

数据上报请求使用 HMAC-SHA256 签名、五分钟时间戳窗口和一次性 nonce。如果客户端与服务端时钟存在偏差，客户端会学习并复用服务端时间偏移，同时保留离线队列样本的相对时间。服务端进程运行期间的时间轴不会回拨，但 TLS 证书校验和准确的历史时间仍依赖 NTP。公开响应使用明确的字段白名单，拒绝跨站浏览器 API 请求，验证节点 ID，并确保用户输入无法直接选择历史记录文件路径。

## 致谢

终端风格的信息架构受到 [CF-Server-Monitor](https://github.com/huilang-me/CF-Server-Monitor) 和 [LTstats](https://github.com/lukastautz/ltstats) 的启发。图表使用 [Chart.js](https://www.chartjs.org)。本项目的 Go 服务端、客户端、存储和前端均为独立实现。

## 卸载

警告：以下命令会永久删除服务端历史记录或客户端离线队列。如果安装时自定义了 `DATA_DIR`，请将命令中的默认数据目录替换为实际路径。

卸载服务端：

```sh
systemctl disable --now uptimed.service 2>/dev/null || true
rm -f /etc/systemd/system/uptimed.service
rm -f /etc/systemd/system/multi-user.target.wants/uptimed.service
rm -rf /etc/systemd/system/uptimed.service.d
rm -f /usr/local/bin/uptimed /etc/uptime/server.json
rm -rf /opt/uptime
userdel uptime 2>/dev/null || true
systemctl daemon-reload
systemctl reset-failed uptimed.service 2>/dev/null || true
if [ ! -e /etc/uptime/agent.json ]; then rmdir /etc/uptime 2>/dev/null || true; fi
```

卸载客户端：

```sh
systemctl disable --now uptime-agent.service 2>/dev/null || true
rm -f /etc/systemd/system/uptime-agent.service
rm -f /etc/systemd/system/multi-user.target.wants/uptime-agent.service
rm -rf /etc/systemd/system/uptime-agent.service.d
rm -f /usr/local/bin/uptime-agent /etc/uptime/agent.json
rm -rf /var/lib/uptime-agent
userdel uptime-agent 2>/dev/null || true
systemctl daemon-reload
systemctl reset-failed uptime-agent.service 2>/dev/null || true
if [ ! -e /etc/uptime/server.json ]; then rmdir /etc/uptime 2>/dev/null || true; fi
```
