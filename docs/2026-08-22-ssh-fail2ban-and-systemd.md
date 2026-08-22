# 2026-08-22：SSH 加固、Fail2Ban 验证与 systemd 常驻验收

> 本次记录重点不是“把软件装上”，而是把 SSH 防护和 systemd 服务逐层验证到真正生效。示例中的端口、用户与路径均为占位符，请按自己的环境替换。

## 一、先确认现状，不要看见攻击就乱关端口

公网 SSH 被自动扫描很常见。先查看防火墙和监听状态：

```bash
sudo ufw status verbose
sudo ss -tlnp
```

原则：

- 入站默认拒绝；
- 只开放确实在使用的端口；
- 不认识的端口先查监听进程，不能直接删除；
- 临时封禁单个来源可以止血，但不能代替自动防护。

```bash
sudo ufw insert 1 deny from <MALICIOUS_IP>
sudo ufw status numbered
```

## 二、为自定义 SSH 端口配置 Fail2Ban

```bash
sudo apt update
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
```

创建 `/etc/fail2ban/jail.d/sshd.local`：

```ini
[sshd]
enabled = true
port = <SSH_PORT>
filter = sshd
logpath = /var/log/auth.log
backend = auto

maxretry = 5
findtime = 10m
bantime = 1h
```

端口必须填写 SSH 实际监听端口，不能机械照抄默认值。

### 1. 测试配置后再重启

```bash
sudo fail2ban-client -t
sudo systemctl restart fail2ban
sudo systemctl status fail2ban --no-pager
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

配置测试应显示：

```text
OK: configuration test is successful
```

### 2. 验证日志过滤器

```bash
sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
```

出现 `Failregex` 命中，说明过滤器能识别当前日志格式。刚重启后的 `Total failed: 0` 不一定是故障：它统计的是本次启动后的新事件，不会自动把全部历史记录重新计入。

### 3. 验证封禁动作真正进入防火墙

只看 service 为 running 还不够。对已确认的恶意来源进行人工测试：

```bash
sudo fail2ban-client set sshd banip <MALICIOUS_IP>
sudo fail2ban-client status sshd
sudo iptables -S | grep -i f2b
```

正确结果应能看到：

- `f2b-sshd` 链已创建；
- SSH 实际端口的流量会进入该链；
- 目标来源命中后执行 `REJECT` 或 `DROP`；
- 链末尾保留 `RETURN`。

解除人工测试封禁：

```bash
sudo fail2ban-client set sshd unbanip <MALICIOUS_IP>
```

## 三、禁用 SSH 密码前，必须从新终端验证私钥

不要在唯一的 SSH 会话里直接禁密码。保留当前会话，再从本地新开终端：

```powershell
ssh -i C:\Users\<USER>\.ssh\<PRIVATE_KEY> -p <SSH_PORT> <ADMIN_USER>@<SERVER_HOST>
```

只有新会话成功进入，才说明密钥链路可用。随后查看 sshd 的有效配置：

```bash
sudo sshd -T | grep -Ei '^(permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication) '
```

推荐目标：

```text
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

修改配置后先测试语法：

```bash
sudo sshd -t
```

无输出通常表示语法通过，再平滑重载：

```bash
sudo systemctl reload ssh
```

始终保留旧会话，直到新会话验证成功。进一步限制高权限账户前，应先准备普通 sudo 用户和主机商控制台/VNC。

## 四、遇到内核升级提示，不要条件反射地远程重启

提示“当前运行内核不是预期的新内核”通常表示新内核已经安装，需要重启才能加载。

重启前检查：

```bash
uname -r
systemctl --failed
systemctl is-enabled ssh fail2ban
```

同时确认：

1. 私钥可从新终端登录；
2. 关键业务服务已设置开机自启；
3. 主机商控制台/VNC 可用；
4. 已列好重启后的验收项。

重启后复查：

```bash
uname -r
systemctl status ssh --no-pager
systemctl status fail2ban --no-pager
sudo fail2ban-client status sshd
```

## 五、用 systemd 常驻 Python 服务

示例 `/etc/systemd/system/<APP>.service`：

```ini
[Unit]
Description=<APP_DESCRIPTION>
After=network.target

[Service]
Type=simple
User=<SERVICE_USER>
WorkingDirectory=/opt/<APP>
ExecStart=/usr/bin/python3 /opt/<APP>/app.py run --interval 60
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> 能使用专用低权限用户时，不要让普通业务服务长期以高权限账户运行。

加载并启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now <APP>.service
sudo systemctl status <APP>.service --no-pager
```

查看完整日志：

```bash
sudo journalctl -u <APP>.service -n 20 --no-pager -o cat
```

验收应同时关注：

- `Loaded: loaded (...; enabled)`：已加载并开机自启；
- `Active: active (running)`：当前进程存活；
- 日志中的周期或序号连续递增；
- 相邻记录间隔符合配置；
- 本地时区正确；
- 业务规则（如安静时段）确实产生预期状态。

## 六、本次验收清单

- [x] 防火墙默认拒绝入站，保留实际业务端口
- [x] 新终端使用指定私钥登录成功
- [x] 密码认证和键盘交互认证关闭
- [x] Fail2Ban sshd jail 加载成功
- [x] 日志过滤器能够识别认证失败
- [x] 人工 banip 后，Fail2Ban 防火墙链真实生效
- [x] Python 服务由 systemd 托管并开机自启
- [x] 从业务日志验证时区、循环和安静时段
- [ ] 在可观察窗口重启，加载新内核并复查全部服务

## 经验总结

1. **手动封 IP 是止血，Fail2Ban 才是持续防护。**
2. **service 显示 running 不等于功能生效，要验证日志、jail 和防火墙链。**
3. **禁密码前必须用新终端验证私钥，旧会话不要提前关。**
4. **公网端口只保留必要项，但不认识的端口应先查进程归属。**
5. **远程重启必须有密钥、开机自启、控制台三重退路。**
6. **systemd 服务不仅要看状态，还要从业务日志验证真实行为。**
