# 2026-08-16：安全巡检与 sticker-mcp 部署

## SSH 扫描与加固

公网 SSH 被自动扫描很常见。日志中出现大量失败不等于已经入侵，还需检查成功登录、授权密钥、异常账户和进程。

处置顺序：

1. 保留当前会话，从第二个终端验证密钥登录；
2. 禁用密码和键盘交互认证；
3. 启用 Fail2Ban，并验证 jail、过滤器和防火墙链；
4. 对已确认来源临时止血时，把拒绝规则插在允许规则之前，或使用对应 Fail2Ban jail。

```bash
sudo ufw insert 1 deny from <MALICIOUS_IP>
sudo fail2ban-client status sshd
sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
```

封禁单个 IP 不能替代密钥认证、限速和持续监控。

## 定期巡检

巡检应覆盖：

- 监听端口与防火墙基线；
- SSH 成功/失败登录和 `authorized_keys` 变化；
- 高 CPU、异常常驻进程和失败的 systemd 单元；
- 磁盘、日志与临时目录增长；
- Fail2Ban jail、过滤器及防火墙链是否真实生效；
- 告警通道能否实际送达。

脚本、告警地址和基线文件中不得硬编码公开 token。定时任务应明确服务器时区，优先使用 systemd timer，避免误解 cron 的执行时间。

## 废弃端口清理

服务停用或改走反向代理后，应同步删除多余的公网规则：

```bash
sudo ufw delete allow <OLD_PORT>/tcp
sudo ss -lntup
sudo ufw status numbered
```

## sticker-mcp 部署原则

部署第三方 MCP 服务前先审查依赖、认证和默认监听地址：

```bash
git clone <REPOSITORY_URL>
cd <PROJECT_DIR>
npm ci
npm run build
```

- 固定依赖并提交 lockfile；
- 使用 systemd 托管；
- 应用监听 `127.0.0.1`；
- 由 Nginx 提供 HTTPS 和认证；
- 不为后端端口额外开放 UFW；
- 从外部网络验证域名入口。

## 本机测试的假象

VPS 本机访问自己的公网地址成功，不一定代表外部网络可达；路径可能绕过预期的防火墙链。完成部署后至少验证：

```bash
sudo nginx -t
sudo ufw status numbered
sudo ss -lntup
```

然后使用另一台主机或移动网络请求公开域名。服务返回的状态码必须结合协议判断，例如未认证时的 401、MCP 端点对普通 GET 返回的 405/406，也可能说明入口已经可达。

