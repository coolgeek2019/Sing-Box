以下是 阶段一：最小可用自用代理系统 的详细技术方案和部署建议。

🎯 阶段一目标
快速搭建一个支持多协议的代理服务

兼顾抗封锁能力、安全性与性能

自用为主，手动配置为主，轻量级，易维护

🧱 架构概览
```
[ 客户端：sing-box / v2rayN / Clash.meta ]
              │
              ▼
[ VPS: sing-box 单实例 ]
   ├─ 入站：VLESS + Reality (TCP)
   ├─ 入站：Hysteria2 (UDP)
   ├─ 入站：Shadowsocks 2022 (TCP/UDP)
   ├─ 出站：Direct / FREEDOM
   └─ DNS：内置 DoH / DoQ
```

🧩 技术栈组成
```
模块	技术 / 工具	说明
代理程序	sing-box（推荐 release 1.9+）	多协议单进程，极致性能
协议	VLESS+Reality、Hysteria2、Shadowsocks 2022	三种主流协议
操作系统	Ubuntu 22.04 / Debian 12	稳定，社区支持好
防火墙	ufw / iptables	限制非必要端口开放
安全防护	Fail2Ban、SSH Key、禁止 root 登录	防爆破、防止横向入侵
域名解析	Cloudflare DNS + API	支持 HTTPS DNS 与 SNI 防污染
DNS	sing-box 内置 DoH（如 https://dns.cloudflare.com/dns-query）	防污染，隐藏请求域名
```

🔧 实践部署流程
1. 安装 sing-box
下载 sing-box（替换为最新版本）
```
curl -fsSL https://sing-box.app/install.sh | bash
```
2. 创建配置文件（推荐手动写好 /etc/sing-box/config.json）
最小示例结构：
```
jsonc
{
  "log": {
    "level": "info",
    "output": "stdout"
  },
  "inbounds": [
    {
      "type": "vless",
      "listen": "::",
      "listen_port": 443,
      "users": [{ "uuid": "你的UUID", "flow": "xtls-rprx-vision" }],
      "tls": {
        "enabled": true,
        "reality": {
          "enabled": true,
          "handshake": {
            "server": "cloudflare.com",
            "server_port": 443
          },
          "private_key": "你的私钥",
          "short_id": ["你的shortID"]
        },
        "server_name": "cloudflare.com"
      }
    },
    {
      "type": "hysteria2",
      "listen": "::",
      "listen_port": 8443,
      "users": [{ "password": "你的密码" }]
    },
    {
      "type": "shadowsocks",
      "listen": "::",
      "listen_port": 2022,
      "method": "2022-blake3-aes-128-gcm",
      "password": "你的密码",
      "network": "tcp,udp"
    }
  ],
  "outbounds": [
    { "type": "direct" },
    { "type": "block", "tag": "blocked" }
  ],
  "dns": {
    "servers": [
      {
        "tag": "dns-remote",
        "address": "https://dns.cloudflare.com/dns-query",
        "detour": "direct"
      }
    ]
  }
}
```
3. 启动并验证
```
systemctl enable sing-box
systemctl restart sing-box
journalctl -u sing-box -f
```

🔐 安全加固建议
SSH：仅允许密钥登录，关闭密码登录

防火墙：
```
ufw allow 443/tcp
ufw allow 8443/udp
ufw allow 2022
ufw enable
```
Fail2Ban：
```
apt install fail2ban -y
systemctl enable --now fail2ban
```

🧪 客户端配置建议
```
客户端	协议	配置
v2rayN / Nekoray	VLESS+Reality	填写 UUID、shortID、私钥、公钥、SNI
sing-box	全部	支持全协议（推荐）
Clash.meta	Hysteria2 / Shadowsocks	需要专用配置转换工具
安卓推荐	Nekoray Android / v2rayNG+meta	支持 Reality 与 Hysteria2
```

✅ 阶段一完成标准
三协议均可正常连接（VLESS+Reality / Hysteria2 / SS2022）

sing-box 运行稳定，日志无异常

有完整 JSON 配置文件和客户端配置备份

安全规则有效（封非必要端口、防暴力破解）

如需我为你生成完整配置模版、私钥生成脚本、客户端配置 QR 链接，可继续补充你的域名/IP/偏好。是否现在继续进入阶段二：一键部署 + 多协议订阅管理？