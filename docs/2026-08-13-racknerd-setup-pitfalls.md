# 新 VPS 部署踩坑记录（2026-08-13）

## 长命令被远程工具超时

`apt install`、`npm install` 等操作可能超过远程管理工具的超时时间。使用 systemd 临时单元或 `nohup` 后台执行，并通过日志轮询结果：

```bash
sudo systemd-run --unit=oneoff-install --collect \
  /bin/sh -c 'apt-get install -y <PACKAGE>'
journalctl -u oneoff-install -f
```

不要因为客户端超时就重复启动多个安装进程。

## dpkg 锁被自动更新占用

先确认持锁进程是否仍在正常工作：

```bash
sudo systemctl status apt-daily.service apt-daily-upgrade.service --no-pager
ps aux | grep -E 'apt|dpkg|unattended'
sudo lsof /var/lib/dpkg/lock-frontend
```

通常应等待自动更新完成。不要在 apt/dpkg 仍运行时强杀进程或删除锁文件，否则可能破坏包管理数据库。

若确认进程已经异常退出，再执行恢复：

```bash
sudo dpkg --configure -a
sudo apt-get -f install
sudo apt-get update
```

不建议为了避开一次锁冲突永久禁用 `unattended-upgrades`，它承担安全更新职责。

## Node.js 版本

先检查项目声明的版本，再选择系统包、NodeSource 或版本管理器，并固定主版本：

```bash
node --version
cat package.json | grep -A3 engines
```

不要默认所有项目都必须使用同一个最新 Node.js 版本。

## SSH 加固顺序

私钥应在可信的本地设备生成，服务器只接收公钥：

```bash
# 在本地执行
ssh-keygen -t ed25519 -f <LOCAL_KEY_PATH>
ssh-copy-id -i <LOCAL_KEY_PATH>.pub -p <SSH_PORT> <ADMIN_USER>@<SERVER_HOST>
```

随后保留旧会话，用第二个终端验证密钥登录。验证成功后再禁用密码认证：

```bash
sudo sshd -t
sudo systemctl reload ssh
sudo sshd -T | grep -Ei 'passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication'
```

不要在服务器上生成私钥再把私钥复制到客户端，也不要把私钥放进项目目录或 Git 仓库。

