# VPS 部署完整指南 + 踩坑记录

> **一次从零开始部署个人 VPS 服务的完整记录**
>
> 涵盖：服务器初始化 · 网络配置 · MCP 服务部署 · systemd 管理 · 10 个实战踩坑

---

## 📋 目录

- [1. 基础环境配置](#1-基础环境配置)
- [2. 网络服务配置（x-ui）](#2-网络服务配置x-ui)
- [3. VPS MCP 服务](#3-vps-mcp-服务)
- [4. 踩坑记录（10 个）](#4-踩坑记录10-个)
- [5. 环境信息](#5-环境信息)
- [6. 常用命令](#6-常用命令)
- [7. 2026-08-11 更新：VPS失联 + SSE根治](#7-2026-08-11-更新vps失联--sse根治)
- [8. 2026-08-16 更新：安全巡检 + 表情包 MCP](#8-2026-08-16-更新安全巡检--表情包-mcp)
- [9. 2026-08-22 更新：SSH 加固 + Fail2Ban 验证](#9-2026-08-22-更新ssh-加固--fail2ban-验证)

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
ufw allow 443     # 网络
ufw allow 面板端口  # 管理面板
ufw allow 8000    # MCP 服务
ufw allow 18110   # 其他服务
ufw enable
```

---

## 2. 网络服务配置（x-ui）

### 2.1 安装 x-ui

x-ui 是一个多协议网络管理面板，可用于反向网络、内网穿透等场景。

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

## 3. VPS 管理服务的安全原则

通过 MCP 或其他远程管理接口操作 VPS 时，**不要把能够执行任意 Shell 命令的接口直接暴露到公网**。即使是个人学习服务器，也至少需要：

- 身份认证：使用高强度随机 token、mTLS 或可信身份代理；
- HTTPS：不要通过明文 HTTP 传输管理凭据；
- 最小权限：服务使用专用低权限账户，避免长期以 root 运行；
- 命令限制：优先提供明确、可审计的工具，不接受任意 `shell=True` 输入；
- 网络限制：能走内网、VPN、Tailscale 或 SSH 隧道时，不直接开放公网端口；
- 日志与限流：记录调用来源、操作和结果，并限制请求频率；
- 密钥保护：token 与私钥放入环境变量或权限受限的配置文件，不提交到 GitHub。

systemd 托管服务时，建议加入专用用户和基础沙箱限制：

```ini
[Service]
User=<SERVICE_USER>
Group=<SERVICE_GROUP>
WorkingDirectory=/opt/<SERVICE_NAME>
ExecStart=/usr/bin/python3 /opt/<SERVICE_NAME>/app.py
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/<SERVICE_NAME>
```

> 具体限制需结合程序所需目录调整。若服务启动失败，先查看 `journalctl -u <SERVICE_NAME>`，不要为了省事直接改回 root。

客户端只保存服务地址与认证信息；公开文档统一使用 `<SERVER_HOST>`、`<PORT>`、`<TOKEN>` 等占位符，不记录真实入口。


---

## 4. 踩坑记录（10 个）

| # | 🕳️ 坑 | ✅ 解决方案 |
|---|--------|------------|
| 1 | SSH 密码登不上 | 用 VNC 登录后执行 `passwd` 修改密码 |
| 2 | 重启后 SSH 连不上 | 等 30~60 秒让 SSHD 启动完成 |
| 3 | `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` | `ssh-keygen -R 你的IP` 清除旧密钥缓存 |
| 4 | `apt upgrade` 弹配置界面 | 直接回车，选默认选项 |
| 5 | 网络连不上 | `ss -tlnp \| grep 443` 检查服务是否在监听 |
| 8 | 杀进程把自己断连 | 确保新服务先跑起来，再杀旧的 |
| 9 | 端口冲突 | 先停掉手动启动的进程，再让 systemd 接管 |
| 10 | Node.js 版本太低 | 先 `apt-get remove -y libnode-dev`，再装 20.x |
| 12 | 面板显示"运行中"但公网全不通 | IP 被墙 / 机房线路抽风——发工单、可尝试换 IP；也可能是临时抽风会自动恢复。**教训：别把重要服务+数据全押一台 VPS（单点故障）** |

> 🗣️ **第一次做服务器运维，踩坑是正常的。每个坑都是一个学习机会。**

---

## 5. 环境信息

| 项目 | 详情 |
|------|------|
| **OS** | Ubuntu 22.04 LTS |
| **运行环境** | Python 3.x, Node.js 20.x |

---

## 6. 常用命令

```bash
# 查看服务状态
systemctl status vps-mcp

# 查看日志
journalctl -u vps-mcp --no-pager -n 20

# 检查端口监听
ss -tlnp | grep -E "<SSH_PORT>|443|<SERVICE_PORT>"

# 查看资源占用
free -h && df -h
```

---

---

## 7. 2026-08-11 更新：VPS失联 + SSE根治

8月11日实录：VPS 突然公网失联（面板显示运行中但 ping/SSH 全超时，梯子跟着断）+ 8000 SSE 频繁断连的根治方案（换成 Streamable HTTP，带 token 认证）。

📄 完整记录见：[docs/2026-08-11-vps-outage-and-8002.md](docs/2026-08-11-vps-outage-and-8002.md)

**一句话总结**：
1. **VPS "运行中"但连不上** = IP 被墙或机房线路抽风 → 发工单 / 等恢复 / 换 IP；准备备用节点防单点故障
2. **SSE 老断** = 换 Streamable HTTP（`uvicorn.run(mcp.streamable_http_app())` + 关 DNS 重绑定保护 + 加 Bearer 认证）

---

## 8. 2026-08-16 更新：安全巡检 + 表情包 MCP

8月16日实录：深夜巡检抓出 **3.4 万次 SSH 暴力破解**（封 IP + 禁密码 + fail2ban + 建每周自动巡检），并把 **sticker-mcp**（AI 发表情包）从零部署上线。最大的坑：**UFW 假象**——nginx 报错导致命令链中断、`ufw allow` 没执行，本机自测 200 是假的（本地流量绕过防火墙），必须从外部机器测公网。

📄 完整记录见：[docs/2026-08-16-security-audit-and-sticker-mcp.md](docs/2026-08-16-security-audit-and-sticker-mcp.md)

**一句话总结**：
1. **巡检要制度化**（crontab + 基线对比 + 告警），别靠"我记得"
2. **禁密码前先验证密钥**，封 IP 要封源头
3. **新端口必须外部测试**——本机自测 200 ≠ 公网通

---

## 9. 2026-08-22 更新：SSH 加固 + Fail2Ban 验证

8 月 22 日实录：发现 SSH 密码尝试后，完成 Fail2Ban 配置与防火墙链实际验证；从新终端验证私钥登录，确认密码认证关闭；并整理了内核升级后的安全重启与复查顺序。公开文档中的端口、用户与地址均使用占位符。

📄 完整记录见：[docs/2026-08-22-ssh-fail2ban-and-server-maintenance.md](docs/2026-08-22-ssh-fail2ban-and-server-maintenance.md)

**一句话总结**：
1. **先验证密钥，再禁密码**，始终保留旧会话作为退路
2. **Fail2Ban 不能只看 running**，还要验证 jail、过滤器与防火墙链
3. **远程重启先准备退路**，重启后复查 SSH、Fail2Ban、防火墙与监听端口

---

> 🕐 2026.7.29~7.30 — **小七 & 林川**
>
> 如有帮助别忘了 ⭐ Star ~

> 📝 **我会一直更新这个指南**，踩到新坑就记上来，希望能帮到更多第一次折腾 VPS 的人。
