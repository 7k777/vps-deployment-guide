# 2026-08-22：SSH 配置覆盖与密钥格式排查

## 配置写了却没有生效

`sshd_config` 和 drop-in 文件中即使出现 `PasswordAuthentication no`，也必须检查最终生效值：

```bash
sudo sshd -t
sudo sshd -T | grep -Ei 'passwordauthentication|pubkeyauthentication|kbdinteractiveauthentication|permitrootlogin'
```

云镜像可能通过 `/etc/ssh/sshd_config.d/*.conf` 提供额外设置。不要只看某一个文件，也不要简单假设“文件名数字越大就一定覆盖”；OpenSSH 对部分关键字采用首次获得的值，应结合主配置中的 Include 顺序与 `sshd -T` 判断。

修改后优先平滑重载，并保留旧会话，直到第二个终端验证成功：

```bash
sudo systemctl reload ssh
```

## Windows 私钥格式问题

私钥被编辑器改写换行或编码后，OpenSSH 可能报告 `invalid format`。私钥应在本地生成并保持原始字节，不要通过聊天、剪贴板或仓库传递。

```powershell
ssh-keygen -t ed25519 -f C:\Users\<USER>\.ssh\<KEY_NAME>
ssh -i C:\Users\<USER>\.ssh\<KEY_NAME> -p <SSH_PORT> <ADMIN_USER>@<SERVER_HOST>
```

服务器只保存公钥内容到 `~/.ssh/authorized_keys`。如果私钥已经通过不安全渠道传输，应重新生成密钥对并撤销旧公钥。

## 加固检查表

1. 本地生成密钥，只上传公钥；
2. 第二个终端验证密钥登录；
3. `sshd -t` 检查语法；
4. `sshd -T` 检查最终生效值；
5. 禁用密码与键盘交互认证；
6. 启用并验证 Fail2Ban；
7. 保留主机商控制台作为应急入口。

