# 2026-08-16 安全巡检 + 表情包 MCP 部署实战

> 实战记录：一次深夜巡检抓出 3.4 万次爆破 + 把 AI 发表情包的服务从零部署上线。

## 一、SSH 暴力破解的发现与处置

巡检 `auth.log` 发现 **34913 次 Failed password**（僵尸网络，单 IP 打了 1.4 万次）。处置：

```bash
# 1. 封禁恶意 IP（top 5）
ufw deny from 恶意IP

# 2. 关闭密码登录（先验证密钥能登录！）
ssh -i 你的私钥 -p 端口 root@127.0.0.1 "echo KEY-LOGIN-OK"
sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart ssh

# 3. 装 fail2ban（自动封禁暴力尝试）
apt-get install -y fail2ban
systemctl enable --now fail2ban
```

> ⚠️ 禁密码前**必须先用密钥验证能登录**，否则把自己锁外面。

## 二、定期巡检机制（承诺靠不住，机制才靠得住）

巡检脚本 `/opt/security-audit.sh`，检查：

- 监听端口基线（新增端口告警）
- crontab md5（被篡改告警）
- authorized_keys（被塞后门公钥告警）
- 高 CPU 陌生进程
- /tmp /var/tmp 新增文件
- SSH 爆破次数（>20 告警）

异常推 Bark 到手机，基线存 `/opt/security-audit-base/`。crontab 每周一 9:00 跑：

```bash
# ⚠️ VPS 时区是 UTC！北京 9:00 = UTC 1:00
0 1 * * 1 /opt/security-audit.sh
```

## 三、废弃端口清理

服务迁走/停用后，UFW 规则要同步删（减少暴露面）：

```bash
ufw delete allow 端口号/tcp
```

## 四、sticker-mcp 部署（AI 发表情包）

[asashiki/sticker-mcp](https://github.com/asashiki/sticker-mcp)：MCP 服务器，让 AI 在聊天里发贴纸/表情包，自带网页管理后台。

```bash
git clone https://github.com/asashiki/sticker-mcp.git
cd sticker-mcp
npm install
npm run build
# systemd 托管 + nginx https 反代 + 防火墙放行
```

工具：`send_sticker`（按情绪发）/ `list_available_stickers` / `add_sticker` / `create_sticker_upload`。

## 五、UFW 假象大坑（重点！）

**现象**：服务部署完，VPS 本机 curl 公网地址返回 200，但外部机器（手机/另一台 VPS）连不上，超时。

**真相**：配置 nginx 时某行报错，`&&` 命令链中断，`ufw allow` 那步**根本没执行**；而 VPS 本机 curl 自己的公网 IP 走本地流量**绕过 UFW**，200 是假象！

**教训**：新端口必须**从外部机器**测试公网连通，不能只在本机自测。

## 六、验证

用另一台 VPS（外部网络）测试：

```bash
curl -X POST https://域名:端口/mcp/sticker -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}'
# → 200 = 通了
```

## 今日金句

- **承诺靠不住，机制才靠得住**（例行检查全部装进 crontab）
- **本机自测 200 不算数，外部访问才是真的**
