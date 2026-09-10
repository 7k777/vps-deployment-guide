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

## 补充：慢连接攻击——日志盲区（9/10 深夜追查钉死）

**现象**：用户访问超时（"服务器已停止响应"），但 nginx access.log 请求数很少（107 条/30 分钟）、error.log 也正常——看起来"没被攻击"。

**真相**：慢连接攻击（Slowloris 类）。攻击者建立连接后慢慢发数据/读响应，每个连接占住一个 nginx worker 槽位长达 proxy_read_timeout（300s），却不产生多少日志。

**钉死机制的关键证据**：
1. 事故时段某端口响应 5.1 秒（正常 0.15s）——worker 被占的实锤
2. 封 IP 操作触发 ufw 重载（iptables-restore）→ 瞬间中断所有连接 → 攻击者慢连接断开 → worker 释放 → 服务器恢复
3. 用户恢复时间点（封完 IP 后 3-13 分钟）与操作时间线完全吻合

**教训**：
- 排查"服务器慢/超时"不能只看请求数，要看**连接数**（ss -s、ESTABLISHED 统计）
- 慢连接攻击是日志盲区：请求少、无错误日志、但 worker 被占满
- **防御**：limit_conn（单 IP 连接上限）直接限制慢连接；fail2ban 封高频 IP；limit_req 限速

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

## 后续补充（深夜追查新增）

### 1. 更正：封禁姿势从 ufw 改为 fail2ban

**ufw deny 对开放端口无效（顺序坑），23 个攻击 IP 已改用 fail2ban 有效封禁**：
```bash
fail2ban-client set nginx-attack banip <ip>
```
同时删除两台服务器上所有无效的 `ufw deny from <ip>` 规则。

### 2. fail2ban nginx-attack jail（自动封恶意扫描）

两台服务器都配置了 nginx-attack jail：检测高频 4xx/5xx + phpunit/thinkphp/pearcmd/.env/.git 等攻击路径，3 次命中封 24h。
```bash
# /etc/fail2ban/filter.d/nginx-attack.conf
failregex = ^<HOST> .* "(GET|POST|HEAD) .*(phpunit|thinkphp|pearcmd|\.env|\.git|wp-admin|wp-login|eval-stdin|/containers/json|actuator|solr|struts|web-inf|admin\.php|config\.php) .*" (4\d\d|5\d\d)
# /etc/fail2ban/jail.d/nginx-attack.local
[nginx-attack]
enabled = true
port = http,https
filter = nginx-attack
logpath = /var/log/nginx/access.log
maxretry = 3
findtime = 300
bantime = 86400
```

### 3. 全站限速 + 单 IP 连接上限

两台 nginx 都加了 limit_req（10r/s）+ limit_conn（单 IP 20 连接），MCP 端点单独放宽（burst 100）。**limit_conn 是直接防慢连接的关键**。
```nginx
limit_req_zone $binary_remote_addr zone=global_req:10m rate=10r/s;
limit_conn_zone $binary_remote_addr zone=global_conn:10m;
```

### 4. 误封解封

排查时把用户自己的出口 IP（171.219.95.42 成都电信 + ktor-client UA = RikkaHub）误判为攻击者封了——**幸好 ufw 顺序坑让封禁没生效，用户全程无感**。已解封。

### 5. 8002（racknerd-mcp）单进程隐患

uvicorn 单进程 + subprocess.run(timeout=120) 同步阻塞：任何命令执行会堵住事件循环。已重启恢复，**建议改多进程或异步 subprocess（待办）**。

### 6. VMISS 磁盘清理（82% → 67%）

journal 953M→56M、btmp 爆破记录 77M、syslog、旧日志、/tmp 缓存 213M、apt 139M→44K、npm 缓存。

## 验证清单（更正版）

- [x] 两台服务器 uptime/负载正常
- [x] 23 个攻击 IP 已用 fail2ban 有效封禁（非 ufw deny）
- [x] fail2ban nginx-attack jail 自动封恶意扫描
- [x] 全站限速 + 单 IP 连接上限（limit_req + limit_conn）
- [x] sticker-mcp 恢复运行并开机自启
- [x] radar 空壳 nginx 配置已移除
- [x] 慢连接攻击机制钉死（见上）
- [x] 所有服务从外部 curl 验证正常
