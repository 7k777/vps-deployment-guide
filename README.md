# VPS 部署完整指南 + 踩坑记录

> 编写于 2026年7月30日
> 基于 Ubuntu 22.04 LTS VPS（1核/1GB/10GB SSD）
> 作者：小七 & 林川

记录一次从零开始部署个人 VPS 服务的完整过程，包括初始化配置、VPN 搭建、MCP 服务部署、以及一路踩过的坑。

---

## 目录

1. [基础环境配置](#1-基础环境配置)
2. [VPN搭建（x-ui + vless+tcp+tls）](#2-vpn搭建x-ui--vlesstcptls)
3. [VPS MCP服务](#3-vps-mcp服务)
4. [踩坑记录（共10个）](#4-踩坑记录共10个)
5. [环境信息](#5-环境信息)
6. [常用命令](#6-常用命令)
## 1. 基础环境配置

### 1.1 购买VPS后第一步
```bash
ssh root@你的IP
apt update && apt upgrade -y
reboot  # 重启载入新内核
```
> 重启后SSH需要等30~60秒才能连上

### 1.2 防火墙
```bash
ufw allow 22      # SSH
ufw allow 443     # VPN
ufw allow 面板端口    # 面板
ufw allow 8000    # MCP服务
ufw allow 18110   # 可选：其他服务
ufw enable
```

---

## 2. VPN搭建（x-ui + vless+tcp+tls）

### 2.1 安装x-ui（多协议代理管理面板）
```bash
bash <(curl -Ls https://raw.githubusercontent.com/vaxilu/x-ui/master/install.sh)
```
安装后第一时间：设强密码、非标准端口、修改面板路径。
x-ui 可用于反向代理、内网穿透等正经用途，正确配置后也能实现安全通信。

### 2.2 域名与DNS
在Cloudflare添加A记录（你的域名 → VPS IP），建议DNS only（灰色云）

### 2.3 SSL证书（使用acme.sh + ZeroSSL）
```bash
apt install -y socat
curl https://get.acme.sh | sh
~/.acme.sh/acme.sh --register-account -m your@email.com
~/.acme.sh/acme.sh --issue -d your-domain.com --standalone
~/.acme.sh/acme.sh --install-cert -d your-domain.com \
  --key-file /etc/ssl/your-domain.com.key \
  --fullchain-file /etc/ssl/fullchain.cer
```

### 2.4 x-ui面板配置
浏览器访问 `http://VPS-IP:面板端口`（具体端口看你安装时设的） → 入站管理 → 添加入站

| 参数 | 值 |
|------|-----|
| 协议 | vless |
| 端口 | 443（或你设的端口） |
| 传输 | tcp |
| TLS | 开启 |
| 证书路径 | /etc/ssl/fullchain.cer |
| 密钥路径 | /etc/ssl/your-domain.com.key |

## 3. VPS MCP服务

让AI能直接操作VPS（跑命令、读文件）——通过 Model Context Protocol (MCP)

### 3.1 Python依赖
```bash
pip3 install "mcp<2.0" uvicorn starlette
```
> **踩坑**：直接 `pip3 install mcp` 会装v2版本，API不兼容。用 `"mcp<2.0"` 降级。

### 3.2 服务端脚本（/root/mcp-server.py）
```python
import subprocess, os
from starlette.applications import Starlette
from starlette.routing import Route
from starlette.responses import JSONResponse
import uvicorn

async def handle_sse(request):
    from mcp.server.models import InitializationOptions
    from mcp.server.starlette import SseServerTransport

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
        "description": "Run shell command on VPS",
        "inputSchema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}
    }])

    sse = SseServerTransport("/messages/")
    async with sse.connect_sse(request.scope, request.receive, request._send) as streams:
        await server.run(streams[0], streams[1], InitializationOptions(
            server_name="vps-mcp",
            server_version="1.0.0",
        ))

async def health_check(request):
    return JSONResponse({"status": "ok"})

app = Starlette(routes=[
    Route("/sse", endpoint=handle_sse),
    Route("/health", endpoint=health_check),
])

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

> ⚠️ **安全提醒**：当前 MCP 服务用于个人学习测试环境。由于支持执行 Shell 命令，请勿直接暴露到公网生产使用。生产环境建议增加身份认证、权限限制和命令白名单。

### 3.3 注册为系统服务（开机自启）
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

> 如果你用的是 RikkaHub，MCP 源设置选 SSE，URL 填 `http://VPS-IP:8000/sse`。
> 其他前端（如 Open WebUI、ChatWise 等）请参考对应文档配置 MCP 源。

---

## 4. 踩坑记录（共10个）

| # | 坑 | 解决 |
|---|-----|------|
| 1 | SSH密码登不上 | 用VNC登录后 `passwd` 改密码 |
| 2 | 重启后SSH被拒 | 等30~60秒让SSHD启动 |
| 3 | WARNING: REMOTE HOST IDENTIFICATION CHANGED | `ssh-keygen -R 你的IP` 清除旧密钥，不是攻击是重装后的正常现象 |
| 4 | apt upgrade弹窗 | 选默认选项回车就行 |
| 5 | VPN连不上 | `ss -tlnp | grep 443` 检查xray在不在监听 |
| 6 | MCP v2 API不兼容 | 用 `pip3 install "mcp<2.0"` 降级 |
| 7 | systemd启动报TypeError | 加 `Environment=TERM=xterm-256color` |
| 8 | 杀进程把自己断连 | 确保新服务先跑起来再杀旧的 |
| 9 | 端口冲突 | 先停手动的进程，systemd绑定上后再杀手动的 |
| 10 | Node.js版本太低 | Ubuntu自带的太旧，删libnode-dev再装20.x |

> 第一次做服务器运维，踩坑是正常的。每个坑都是一个学习机会。

---

## 5. 环境信息

- **OS**: Ubuntu 22.04 LTS
- **硬件**: 1 Core / 1GB RAM / 10GB SSD
- **运行环境**: Python 3.x, Node.js 20.x

---

## 6. 常用命令

```bash
# 查看服务状态
systemctl status vps-mcp
systemctl status xinchao

# 查看日志
journalctl -u xinchao --no-pager -n 20
journalctl -u vps-mcp --no-pager -n 20

# 检查端口监听状态
ss -tlnp | grep -E "443|8000|18110|面板端口"

# 查看系统资源
free -h && df -h
```

---

**特别提醒**：
- 所有密码、UUID、IP地址请务必替换为你自己的
- SSL证书到期前记得续期（Let's Encrypt / ZeroSSL）
- 防火墙不要全关，只开放必要的端口
- 部署完成后先重启一次确认所有服务自启正常

> 2026.7.29~7.30 — 小七 & 林川
