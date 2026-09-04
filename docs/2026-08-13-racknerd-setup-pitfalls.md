# RackNerd 新机部署实战：踩坑记录（2026-08-13）

给新 VPS（RackNerd 洛杉矶 DC03，Ubuntu 22.04）做初始部署时踩的坑。

## 坑1：耗时命令超时

apt install / npm install 等长命令在 MCP 客户端会被取消（超时），看起来像"卡死/失败"。

解法：nohup 后台跑 + 重定向日志 + 分次轮询，别在前台等。

```bash
nohup apt-get install -y <包名> > /tmp/apt.log 2>&1 &
# 轮询：
tail -f /tmp/apt.log
```

## 坑2：unattended-upgrades 锁 dpkg

新机器首次开机后自动更新会锁住 dpkg，apt 装啥都卡（报 dpkg lock）。

解法：彻底禁用 unattended-upgrades，再清 stale 锁。

```bash
systemctl mask unattended-upgrades
pkill -f unattended-upgrades
rm -f /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/cache/apt/archives/lock
apt-get update
apt-get install -f
```

## 坑3：Ubuntu 22.04 自带 node 太老（12.x）

系统自带 node 是 12，跑现代项目直接报错。用 nodesource 装 node 20：

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs
node -v  # 20.x
```

## 坑6：SSH 加固三件套

新机器到手先做：改端口 + 密钥登录 + 禁密码。

```bash
# 1. 改端口（sshd_config 追加）
echo 'Port 22022' >> /etc/ssh/sshd_config
systemctl restart sshd

# 2. 密钥登录
ssh-keygen -t ed25519 -f ~/.ssh/<name>_key -N ""
cat ~/.ssh/<name>_key.pub >> ~/.ssh/authorized_keys

# 3. 禁密码
sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd
```

注意：HOME 未设置时 `~` 不会展开，密钥会生成到字面 `~` 目录——务必先确认 HOME 变量。

（坑4/坑5 为网络相关项，公开仓库合规脱敏，不入库）
