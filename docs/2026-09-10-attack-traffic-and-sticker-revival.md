# 2026-09-10—11：攻击排障、SYN 防护与服务恢复

## 事故表现

多个站点出现白屏、加载超时或“服务器已停止响应”。主机仍在线，Nginx 进程正常，CPU、内存和磁盘没有耗尽，也没有 OOM 或入侵证据。

## 分层排查结果

### 主机与 Nginx

- 两台主机负载正常，无 OOM；
- Nginx active，错误日志没有足以解释全站超时的异常峰值；
- 事故时段 HTTP 请求量不高，不能支持“海量完整 HTTP 请求打满 worker”的说法。

### 应用层

- 一个已停用站点仍保留 Nginx 反代，持续产生 `connect() failed (111)`；清理空壳配置后噪音消失；
- sticker-mcp 曾处于 stopped/disabled，恢复服务并启用开机自启后正常；
- 管理 MCP 使用单 Uvicorn 进程，工具函数中存在最长 120 秒的同步 `subprocess.run`，会阻塞 ASGI 事件循环。

### 攻击流量

日志中确认存在 PHP、phpunit、pearcmd、Docker API、`.env`、`.git` 等自动扫描请求，但完整 HTTP 请求数量不足以单独解释全站超时。

最初曾将“延迟升高、请求日志少、封禁后恢复”解释为 Slowloris，并写成“已经钉死”。这个结论证据不足：

- 5 秒响应只能证明当时变慢，不能证明 worker 被慢连接占满；
- Nginx 是事件驱动架构，一个空闲连接不等于独占一个 worker 进程；
- UFW 重载与恢复时间吻合属于相关性，不足以唯一确定攻击类型；
- 事故发生时没有保存 `ss` 连接状态快照，因此无法事后对 Slowloris 定案。

后续巡检直接观察到：正常 ESTABLISHED 连接很少，但 80/443 上出现数十条 `SYN-RECV`，峰值约为 **76**，而当时 `tcp_max_syn_backlog` 只有 **128**。来源分散在多个网段。这是当时捕获到的网络层 SYN flood 证据，能够解释为什么请求尚未进入 Nginx 日志、浏览器却可能在握手阶段超时。

严谨结论：**自动扫描与 SYN flood 均真实存在；事故表现与半连接队列压力一致，但由于事故时缺少连接快照，不能把最初那次超时百分之百归因于某一种攻击。**

## 已实施处置

### Nginx

- 全站请求速率与单 IP 连接数限制；
- 缩短请求头、请求体、发送和 keep-alive 超时；
- 只启用 TLS 1.2/1.3，隐藏版本信息；
- 配置 Cloudflare 官方可信代理网段和 `CF-Connecting-IP`，让日志、限速与 Fail2Ban 使用真实客户端地址；
- 修改管理 MCP 的访问日志格式，不再记录 URL 查询参数。

> Cloudflare IP 段会变化，应从官方列表定期更新。只有可信代理地址可以改写真实客户端 IP。

### Fail2Ban 与防火墙

- 建立 `nginx-attack` jail，识别常见恶意扫描路径；
- 使用 `fail2ban-regex` 验证过滤器可命中历史样本；
- 手动封禁通过对应 jail 执行，确保规则进入前置的 Fail2Ban 链；
- 删除管理 MCP 和 tags 后端的公网放行，两个服务均只监听回环地址。

刚启动 jail 后 `Total failed: 0` 不代表配置失效：该计数只统计本次运行期间的新事件，历史日志需要用 `fail2ban-regex` 单独验证。

UFW 的顺序必须以实际链为准。不要笼统认为所有 `ufw deny from <IP>` 都必然无效；应使用 `ufw status numbered`、`iptables -S` 或 `nft list ruleset` 验证。需要把拒绝规则放到通用 ALLOW 之前时，可用：

```bash
sudo ufw insert 1 deny from <MALICIOUS_IP>
```

### SYN flood 内核缓解

```ini
# /etc/sysctl.d/99-syn-flood-hardening.conf
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.core.somaxconn = 4096
net.ipv4.tcp_synack_retries = 2
```

这些参数提高主机对中小规模半连接洪泛的承受能力，但不能替代上游清洗或隐藏源站 IP。遭遇大流量 DDoS 时，主机本地配置无法挽救已被塞满的带宽。

### 管理 MCP

- 后端从 `0.0.0.0` 改为 `127.0.0.1`；
- 删除后端端口的 UFW 公网放行；
- 同步命令改为 `asyncio.to_thread`，避免阻塞事件循环；
- 使用信号量限制并发命令数；
- Nginx HTTPS 入口和认证保持不变。

旧客户端若仍把 token 放在 URL 中，应尽快迁移到 Authorization header，随后轮换 token。仅停止新日志记录不能消除历史日志中已出现的凭据。

### tags 服务与 Supabase

tags 进程和 Nginx 一直正常，但访问数据库时报 DNS 解析失败。公共 DNS 对项目域名返回 NXDOMAIN，Supabase 控制面显示项目为 INACTIVE。恢复项目后：

- 项目状态回到健康；
- 数据表记录仍在；
- Supabase REST、本机健康检查和 Nginx 入口均返回 200；
- `.env` 权限从 644 收紧为 600；
- 增加每日只读查询的 systemd timer，降低免费项目因低活跃再次暂停的概率。

业务 `/tags` 的 401 与 Supabase key 是两套认证：Supabase key 只供后端访问数据库；客户端需先调用 `/login`，再使用登录返回的 Bearer token。当前登录 token 保存在进程内存中，服务重启后会失效，这是后续需要改进的会话设计。

## 仍待完成

- 把依赖独立公网端口的服务迁到标准 HTTPS 域名或安全隧道；
- 隐藏源站 IP，并在迁移完成后只允许可信代理访问 Web 入口；
- 轮换曾出现在 URL 或历史日志中的管理 token；
- 把内存登录会话改为可撤销、可过期的持久会话；
- 为关键数据库和配置建立异机备份。

## 排障清单

- [x] 检查负载、内存、磁盘、OOM 和异常进程
- [x] 检查 Nginx、上游端口和错误日志
- [x] 区分完整 HTTP 请求、慢连接与 SYN 半连接
- [x] 验证 Fail2Ban 过滤器和防火墙链
- [x] 收紧后端监听地址与 UFW 规则
- [x] 修复管理 MCP 的事件循环阻塞
- [x] 恢复 tags 数据库并做三层验证
- [ ] 完成源站隐藏和独立公网端口迁移
- [ ] 完成管理 token 轮换

