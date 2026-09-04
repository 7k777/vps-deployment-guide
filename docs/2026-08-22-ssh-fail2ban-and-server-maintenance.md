# 2026-08-22：SSH 加固、Fail2Ban 验证与内核升级准备

> 本篇只记录 VPS 本身的安全与维护。示例中的端口、用户、地址均为占位符，请按自己的服务器替换。

## 一、先确认防火墙与监听状态

公网 SSH 被自动扫描很常见。发现密码尝试后，先检查现状，不要直接删除不认识的规则：

```bash
sudo ufw status verbose
sudo ss -tlnp
```

原则：

- 入站默认拒绝；
- 只开放确实在使用的端口；
- 不认识的端口先查监听进程与服务归属；
- 手动封禁单个来源只能止血，不能代替持续防护。

临时封禁已确认的恶意来源：

```bash
sudo ufw insert 1 deny from <MALICIOUS_IP>
sudo ufw status numbered
```

## 二、为自定义 SSH 端口配置 Fail2Ban

安装并启用：

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

这里表示：同一来源在 10 分钟内认证失败 5 次，封禁 1 小时。若 SSH 不使用默认端口，`port` 必须填写实际监听端口。

### 1. 测试配置再重启

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

### 2. 验证过滤器能识别日志

```bash
sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
```

出现 `Failregex` 命中，说明过滤器能识别当前日志格式。

刚重启后 `Total failed: 0` 不一定表示配置失效：该计数针对本次 jail 运行期间的新事件，历史日志不会自动全部重新计入。

### 3. 验证封禁动作真正进入防火墙

只看 `fail2ban.service` 为 running 还不够。可对已确认的恶意来源做一次人工测试：

```bash
sudo fail2ban-client set sshd banip <MALICIOUS_IP>
sudo fail2ban-client status sshd
sudo iptables -S | grep -i f2b
```

正确结果应能证明：

- `f2b-sshd` 链已经创建；
- SSH 实际端口的流量会进入该链；
- 被封来源命中后执行 `REJECT` 或 `DROP`；
- 链末尾保留 `RETURN`。

测试完成后解除人工封禁：

```bash
sudo fail2ban-client set sshd unbanip <MALICIOUS_IP>
```

## 三、禁用 SSH 密码前先验证私钥

不要在唯一的 SSH 会话中直接关闭密码。保留当前会话，从本地另开一个终端：

```powershell
ssh -i C:\Users\<USER>\.ssh\<PRIVATE_KEY> -p <SSH_PORT> <ADMIN_USER>@<SERVER_HOST>
```

只有新会话成功进入，才说明密钥链路可用。随后查看 sshd 的有效配置：

```bash
sudo sshd -T | grep -Ei '^(permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication) '
```

关键目标：

```text
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```

修改 SSH 配置后，先测试语法：

```bash
sudo sshd -t
```

无输出通常表示语法通过，再平滑重载：

```bash
sudo systemctl reload ssh
```

旧会话必须保留到新会话验证成功。若进一步限制高权限账户，应先准备普通 sudo 用户以及主机商控制台或 VNC 作为应急入口。

## 四、内核升级提示与重启准备

系统更新时若提示当前运行内核不是最新安装版本，通常表示新内核已安装，但需要重启才能加载，并不等于更新失败。

远程重启前检查：

```bash
uname -r
systemctl --failed
systemctl is-enabled ssh fail2ban
```

同时确认：

1. 私钥能从新终端正常登录；
2. 关键服务器服务已设置开机自启；
3. 主机商控制台或 VNC 可用；
4. 已记录重启后需要检查的端口和服务。

重启后复查：

```bash
uname -r
systemctl status ssh --no-pager
systemctl status fail2ban --no-pager
sudo fail2ban-client status sshd
sudo ufw status verbose
sudo ss -tlnp
```

## 五、本次服务器验收清单

- [x] 防火墙默认拒绝入站，实际业务端口未被误删
- [x] 新终端使用指定私钥登录成功
- [x] 密码认证与键盘交互认证关闭
- [x] Fail2Ban 的 sshd jail 加载成功
- [x] 过滤器能够识别 SSH 认证失败
- [x] 人工 `banip` 后，Fail2Ban 防火墙链真实生效
- [ ] 在可观察窗口重启，加载新内核并复查 SSH、Fail2Ban、防火墙和监听端口

## 经验总结

1. **手动封 IP 是止血，Fail2Ban 才是持续防护。**
2. **服务显示 running 不等于功能生效，要继续验证日志、jail 与防火墙链。**
3. **禁密码前必须用新终端验证私钥，旧会话不要提前关闭。**
4. **不认识的开放端口先查服务归属，不要因为紧张误删自己的业务规则。**
5. **远程重启前必须准备密钥、开机自启和控制台三重退路。**
