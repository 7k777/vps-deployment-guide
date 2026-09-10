# VPS 部署与安全运维记录

> 一份面向个人学习服务器的部署、排障与安全加固笔记。

## 目录

- [基础环境](#基础环境)
- [公网服务安全原则](#公网服务安全原则)
- [常用检查命令](#常用检查命令)
- [事故与维护记录](#事故与维护记录)

## 基础环境

首次登录后先更新系统，并为重启准备主机商控制台或 VNC：

```bash
ssh <ADMIN_USER>@<SERVER_HOST>
sudo apt update
sudo apt upgrade -y
sudo reboot
```

防火墙只开放确实需要的入口。应用进程优先监听 `127.0.0.1`，通过 Nginx 的 80/443 反向代理，不要为每个后端单独开放公网端口：

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow <SSH_PORT>/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

修改 SSH 或防火墙前，必须保留旧会话，并从第二个终端验证新连接。

## 公网服务安全原则

能够执行命令、读写数据或调用第三方 API 的管理服务不应裸露在公网。至少做到：

- 使用高强度随机凭据，凭据只放在权限为 600 的环境文件或密钥系统中；
- 全程 HTTPS，日志不记录 URL 查询参数中的 token；
- 应用监听回环地址，由 Nginx、VPN、Tailscale 或身份代理提供入口；
- 后端使用专用低权限账户；确需 root 的管理工具应进一步限制入口与可执行能力；
- 避免 `shell=True` 任意命令接口；无法避免时必须认证、限并发、记录审计并缩小网络暴露面；
- 配置请求限速、单 IP 连接限制、合理的连接超时和 Fail2Ban；
- Cloudflare 代理站点要配置可信代理网段和真实客户端 IP，不能信任任意来源伪造的转发头；
- 服务停用时同时清理 Nginx、DNS、防火墙和开机自启配置。

systemd 沙箱需根据实际读写目录调整：

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

## 常用检查命令

```bash
# 资源与内核异常
uptime
free -h
df -h
journalctl -k --since "1 hour ago" | grep -Ei 'oom|killed process|segfault'

# 服务、监听与连接状态
systemctl --failed
systemctl status <SERVICE_NAME> --no-pager
ss -lntup
ss -s
ss -Htan state syn-recv '( sport = :80 or sport = :443 )'

# Nginx 与 Fail2Ban
sudo nginx -t
sudo fail2ban-client status
sudo fail2ban-client status <JAIL_NAME>
sudo fail2ban-regex /var/log/nginx/access.log /etc/fail2ban/filter.d/<FILTER>.conf

# 防火墙
sudo ufw status numbered
sudo iptables -S | grep -i f2b
```

### 判断原则

1. `active` 只代表进程在运行，不代表业务请求成功。
2. 本机 `curl` 成功不代表公网可达，应再从外部网络验证。
3. Nginx `connect() failed (111)` 通常表示上游没有监听，不等于 Nginx 被打垮。
4. 请求日志少不等于没有攻击；SYN flood 和未完成的 HTTP 连接可能尚未进入 access log。
5. 延迟升高、封禁后恢复等时间相关性只能形成线索，不能单独当作攻击类型的定案证据。

## 事故与维护记录

- [2026-08-11：VPS 失联与 Streamable HTTP](docs/2026-08-11-vps-outage-and-8002.md)
- [2026-08-13：新机部署踩坑](docs/2026-08-13-racknerd-setup-pitfalls.md)
- [2026-08-16：安全巡检与 sticker-mcp](docs/2026-08-16-security-audit-and-sticker-mcp.md)
- [2026-08-22：SSH、Fail2Ban 与维护](docs/2026-08-22-ssh-fail2ban-and-server-maintenance.md)
- [2026-08-22：SSH 配置覆盖与密钥格式](docs/2026-08-22-ssh-hardening-and-wakebridge.md)
- [2026-09-10—11：攻击排障、SYN 防护与服务恢复](docs/2026-09-10-attack-traffic-and-sticker-revival.md)

## 仍需完成的架构改进

- 把仍依赖独立公网端口的服务迁到标准 HTTPS 域名或安全隧道，再关闭侧端口；
- 隐藏源站 IP，仅允许可信代理访问 Web 入口；
- 轮换曾经出现在 URL 或历史日志中的管理 token，并同步更新客户端；
- 为关键数据库与配置建立异机备份和恢复演练。

> 文档中的主机、账户、端口和凭据均使用占位符。不要把真实密钥、出口 IP 或管理入口提交到公开仓库。

