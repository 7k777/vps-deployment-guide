# VPS 部署完整指南 + 踩坑记录

> **一次从零开始部署个人 VPS 服务的完整记录**
>
> 涵盖：服务器初始化 · 代理配置 · MCP 服务部署 · systemd 管理 · 10 个实战踩坑

---

## 📋 目录

- [1. 基础环境配置](#1-基础环境配置)
- [2. 代理服务搭建（x-ui）](#2-代理服务搭建x-ui)
- [3. VPS MCP 服务](#3-vps-mcp-服务)
- [4. 踩坑记录（10 个）](#4-踩坑记录10-个)
- [5. 环境信息](#5-环境信息)
- [6. 常用命令](#6-常用命令)

---

## 1. 基础环境配置

### 1.1 购买 VPS 后的第一步

```bash
ssh root@你的IP
apt update && apt upgrade -y
reboot
```

> ⏳ 重启后 SSH 需要等 30~60 秒才能连上，别急

### 1.2 防火墙配置

开放必要的端口，其他全部关闭：

```bash
ufw allow 22      # SSH
ufw allow 443     # 代理
ufw allow 面板端口  # 管理面板
ufw allow 8000    # MCP 服务
ufw allow 18110   # 其他服务
ufw enable
```

---

## 2. 代理服务搭建（x-ui）

### 2.1 安装 x-ui

x-ui 是一个多协议代理管理面板，可用于反向代理、内网穿透等场景。

```bash
bash <(curl -Ls https://raw.githubusercontent.com/vaxilu/x-ui/master/install.sh)
```

安装完成后立即修改默认账户、密码和端口。

### 2.2 域名与 DNS

在 Cloudflare 添加 A 记录，将你的域名指向 VPS IP。建议开启 DNS only（灰色云）。

### 2.3 SSL 证书（acme.sh + ZeroSSL）

```bash
# 安装依赖
apt install -y socat

# 安装 acme.sh
curl https://get.acme.sh | sh

# 注册账户
~/.acme.sh/acme.sh --register-account -m your@email.com

# 申请证书
~/.acme.sh/acme.sh --issue -d your-domain.com --standalone

# 安装证书
~/.acme.sh/acme.sh --install-cert -d your-domain.com \
  --key-file /etc/ssl/your-domain.com.key \
  --fullchain-file /etc/ssl/fullchain.cer
```

### 2.4 面板配置

浏览器访问 `http://VPS-IP:面板端口` → 入站管理 → 添加入站：

| 参数 | 建议值 |
|------|--------|
| 协议 | vless |
| 端口 | 443 |
| 传输 | tcp |
| TLS | 开启 |
| 证书路径 | `/etc/ssl/fullchain.cer` |
| 密钥路径 | `/etc/ssl/your-domain.com.key` |

---

## 3. VPS MCP 服务

通过 MCP（Model Context Protocol）让 AI 助手可以直接操作 VPS。

### 3.1 安装依赖

```bash
pip3 install "mcp<2.0" uvicorn starlette
```

> ⚠️ **注意**：直接 `pip3 install mcp` 会装 v2 版本，API 不兼容。用 `"mcp<2.0"` 降级到 v1。

### 3.2 服务端脚本

在 `/root/mcp-server.py` 中创建 MCP 服务，提供 `run_command` 工具：

```python
from starlette.applications import Starlette
from starlette.routing import Route
from starlette.responses import JSONResponse
import uvicorn, subprocess

async def handle_sse(request):
    from mcp.server import Server
    from mcp.server.starlette import SseServerTransport
    from mcp.server.models import InitializationOptions

    server = Server("vps-mcp")

    async def handle_run_command(args):
        cmd = args["command"]
        try:
            r = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=30)
            return {"stdout": r.stdout, "stderr": r.stderr, "returncode": r.returncode}
        except subprocess.TimeoutExpired:
            return {"error": "Command timed out (30s)"}

    server.set_tools([{
        "name": "run_command",
        "description": "在 VPS 上执行 Shell 命令",
        "inputSchema": {
            "type": "object",
            "properties": {"command": {"type": "string"}},
            "required": ["command"]
        }
    }])

    sse = SseServerTransport("/messages/")
    async with sse.connect_sse(request.scope, request.receive, request._send) as streams:
        await server.run(streams[0], streams[1], InitializationOptions(
            server_name="vps-mcp", server_version="1.0.0",
        ))

async def health(request):
    return JSONResponse({"status": "ok"})

app = Starlette(routes=[
    Route("/sse", endpoint=handle_sse),
    Route("/health", endpoint=health),
])

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

> ⚠️ **安全提醒**：该服务支持执行 Shell 命令，仅用于个人学习环境。生产环境建议增加身份认证、权限限制和命令白名单。

### 3.3 配置 systemd 开机自启

```bash
cat > /etc/systemd/system/vps-mcp.service << 'EOF'
[Unit]
Description=VPS MCP Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 -u /root/mcp-server.py
Restart=always
RestartSec=5
WorkingDirectory=/root
Environment=TERM=xterm-256color

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable vps-mcp
systemctl start vps-mcp
```

### 3.4 客户端配置

在支持 MCP 的客户端（如 RikkaHub、Claude Desktop 等）中添加：

```
类型：SSE
URL：http://VPS-IP:8000/sse
```

---

## 4. 踩坑记录（10 个）

| # | 🕳️ 坑 | ✅ 解决方案 |
|---|--------|------------|
| 1 | SSH 密码登不上 | 用 VNC 登录后执行 `passwd` 修改密码 |
| 2 | 重启后 SSH 连不上 | 等 30~60 秒让 SSHD 启动完成 |
| 3 | `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` | `ssh-keygen -R 你的IP` 清除旧密钥缓存 |
| 4 | `apt upgrade` 弹配置界面 | 直接回车，选默认选项 |
| 5 | VPN 连不上 | `ss -tlnp \| grep 443` 检查 xray 是否在监听 |
| 6 | MCP v2 API 不兼容 | 用 `pip3 install "mcp<2.0"` 降级到 v1 |
| 7 | systemd 启动报 `TypeError` | 在 Service 中加 `Environment=TERM=xterm-256color` |
| 8 | 杀进程把自己断连 | 确保新服务先跑起来，再杀旧的 |
| 9 | 端口冲突 | 先停掉手动启动的进程，再让 systemd 接管 |
| 10 | Node.js 版本太低 | 先 `apt-get remove -y libnode-dev`，再装 20.x |

> 🗣️ **第一次做服务器运维，踩坑是正常的。每个坑都是一个学习机会。**

---

## 5. 环境信息

| 项目 | 详情 |
|------|------|
| **OS** | Ubuntu 22.04 LTS |
| **硬件** | 1 Core / 1GB RAM / 10GB SSD |
| **运行环境** | Python 3.x, Node.js 20.x |
| **主机商** | VMISS（日本东京 IIJ 线路） |

---

## 6. 常用命令

```bash
# 查看服务状态
systemctl status vps-mcp

# 查看日志
journalctl -u vps-mcp --no-pager -n 20

# 检查端口监听
ss -tlnp | grep -E "443|8000|面板端口"

# 查看资源占用
free -h && df -h
```

---

> 🕐 2026.7.29~7.30 — **小七 & 林川**
>
> 如有帮助别忘了 ⭐ Star ~
