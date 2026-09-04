# 2026-08-22 实战：SSH 加固排查

## 1. 密码登录"改了没生效"的坑

症状：`sshd_config` 和 `sshd_config.d/99-hardening.conf` 都写了 `PasswordAuthentication no`，但 `sshd -T` 仍显示 `yes`。

真凶：`/etc/ssh/sshd_config.d/60-cloudimg-settings.conf`（云镜像自动生成，默认 `PasswordAuthentication yes`）覆盖了 99 的 no。

教训：验证必须用 `sudo sshd -T`（实际生效配置），不能只看配置文件。

修复：把 60-cloudimg 也改成 no，然后 `systemctl restart ssh`，再用 `sshd -T` 验证。

## 2. Windows 密钥 CRLF 坑

症状：OpenSSH 私钥报 `invalid format`（`ssh -v` 显示 `identity file ... type -1`）。

原因：用记事本/复制粘贴保存密钥文件，换行符是 CRLF，OpenSSH 不认。

修复：`scp` 从服务器字节级复制最可靠，别手抄、别用记事本编辑密钥。

```bash
scp -i 本机密钥 -P 22022 root@SERVER:/服务器/密钥路径 本地路径
```

## 3. 双机加固流程速查

1. 封爆破 IP：`ufw deny from <IP>`（全封）
2. fail2ban：`apt install fail2ban && systemctl enable --now fail2ban`（默认自带 sshd jail）
3. 禁密码：`echo 'PasswordAuthentication no' > /etc/ssh/sshd_config.d/99-hardening.conf && systemctl restart ssh && sshd -T` 验证
4. 密钥验证：`ssh -v -i <key> -p <port> root@<ip>`（看 `Offering public key` 行）

## 4. 防火墙概念大白话

- 端口 = 门（每扇门通向不同服务）
- `Anywhere` = 谁都能进这扇门
- `DENY <IP>` = 拉黑特定 IP
- fail2ban = 自动门卫（反复敲错门自动拉黑）
- systemd = 给服务办永久居留证（开机自动上岗）
