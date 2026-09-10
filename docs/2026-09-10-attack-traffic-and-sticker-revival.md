# 攻击流量排障与 sticker-mcp 复活（2026-09-10 实战）

## 现象

用户访问多个服务时浏览器显示「服务器已停止响应」（白屏等待后超时），怀疑服务器宕机或数据丢失。

## 排查过程

1. **先确认服务端状态**：`uptime` 负载 0.00，两台服务器都活着，nginx active，端口都在监听。
2. **从外部验证服务**：curl 各 HTTPS 端口——大部分正常，个别端口响应变慢。
3. **查 nginx 错误日志**（关键证据）：
   ```
   tail -20 /var/log/nginx/error.log | grep \[error\]
   ```
   发现大量 `connect() failed (111)` 和来自同一 IP 的密集请求。

## 根因

自动化扫描攻击：攻击者 IP 对服务器发起 PHP 漏洞探测（phpunit、thinkphp、pearcmd、Docker API 探测等），请求量巨大，**挤占 nginx worker 连接**，导致正常用户请求排队超时，浏览器表现为「服务器已停止响应」。

这不是服务器宕机，是攻击流量造成的假象。

## 处置

1. **封禁攻击 IP**（两台服务器都执行）：
   ```bash
   for ip in <攻击者IP列表>; do ufw deny from $ip; done
   ```
   通过 `awk '{print $1}' access.log | sort | uniq -c | sort -rn | head` 找出高频 IP，两台服务器共封 20 个。
2. **确认 fail2ban 在跑**：`systemctl is-active fail2ban`，作为自动防线。
3. **封禁后复测**：服务响应时间从超时恢复到 0.5~0.7s。

## 顺带挖出的隐藏问题

### 1. sticker-mcp 服务长期停摆

- nginx 转发 8447 → 127.0.0.1:3000 一直报 111（连接失败）
- 查 `systemctl status sticker-mcp`：服务 **disabled 且 dead**，日志显示 8/15 被 stop 后再没启动
- 处置：`systemctl enable sticker-mcp && systemctl start sticker-mcp`，验证 3000 端口监听、8447 返回 406（MCP 端点正常响应）

### 2. 空壳站点残留

- radar.newkis.cc 的 nginx 配置还在，但后端（3017 端口）早已没有服务
- 攻击者持续扫描该域名，nginx 日志刷满 111 错误
- 处置：`rm /etc/nginx/sites-enabled/ai-needs-radar && nginx -t && systemctl reload nginx`

## 教训

1. **「服务器停止响应」先查服务端再怪网络**：uptime + nginx error log 是第一步证据。
2. **111 connect failed = 后端没监听**：nginx 转发失败时查 systemd 服务状态，别只盯着 nginx。
3. **disabled 的服务会被遗忘**：定期巡检 `systemctl list-units --all` 检查有没有服务悄悄停着。
4. **攻击流量会挤占 worker**：封 IP 立竿见影，fail2ban 是基础、手动封高频攻击者更快。
5. **废弃站点要拆干净**：服务停了，nginx 配置和 DNS 记录也要一起清理，否则攻击者天天来敲门。
6. **同一攻击团伙会同时打多台服务器**：一台被攻击时，另一台也要查。

## 补充：ufw deny 顺序坑（当晚最大发现）

**ufw 的 `deny from <ip>` 规则如果加在 ALLOW 端口规则之后，对开放端口完全无效。**

iptables 按顺序匹配：从被封 IP 到开放端口的连接会**先命中 ACCEPT（放行）**，DENY 规则根本轮不到。实测两台服务器手动封的 23 个攻击 IP 全部被这个坑废掉。

**正确封禁姿势**：
```bash
# ❌ 无效（顺序坑）：ufw deny from <ip>
# ✅ 有效（f2b 链在 INPUT 最前）：fail2ban-client set <jail> banip <ip>
fail2ban-client set nginx-attack banip 43.110.38.5
```

**误封教训**：把用户自己的出口 IP（成都电信 + ktor-client UA = RikkaHub 特征）误判为攻击者封了——幸好顺序坑让封禁没生效，用户全程无感。**封 IP 前必须查归属和 UA**；5G 移动网络出口 IP 动态变化，别拿 IP 当封禁依据。

## 验证清单

- [x] 两台服务器 uptime/负载正常
- [x] 20 个攻击 IP 已封禁（ufw deny）
- [x] fail2ban active
- [x] sticker-mcp 恢复运行并开机自启
- [x] radar 空壳 nginx 配置已移除
- [x] 所有服务从外部 curl 验证正常
