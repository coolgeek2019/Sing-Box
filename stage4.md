以下是 阶段四：商用系统 MVP（注册 + 支付 + 限额控制） 的完整技术架构与部署建议。该阶段的目标是构建一个 小规模收费代理服务（10～100 用户），实现用户注册、订阅、支付、限额控制等基本商业功能。

🎯 阶段四目标
提供用户可自助注册 / 登录 / 修改信息的面板

用户可查看订阅链接（多协议 / 多节点）

支持订阅有效期和流量限额管理

实现在线支付（法币 / USDT）+ 自动激活

后台支持用户管理 / 续费 / 限速等运营操作

🧱 架构总览图
```
                  ┌────────────────────────┐
                  │     用户 Web 面板       │◄─────[ 注册 / 登录 ]
                  │                        │
                  │ 订阅链接、状态、续费、充值 │
                  └───────┬──────┬─────────┘
                          │      │
       ┌──────────────────┘      └────────────┐
       ▼                                       ▼
[ sing-box-manager 多节点生成订阅 ]       [ 支付系统（Stripe / USDT） ]
       │                                       │
       ▼                                       ▼
[ sing-box 实例组（Reality / Hysteria2 / SS2022） ]
```
🧩 技术栈建议
```
模块	技术
前端	React + Tailwind CSS / Next.js
后端	Go（推荐继续基于 sing-box-manager） / Node.js
数据库	SQLite（早期）/ PostgreSQL（可拓展）
管理面板	自研或魔改 sing-box-manager
支付接口	Stripe（信用卡）或 TRC20 USDT（+回调处理）
订阅格式	Clash URI / sing-box JSON / v2rayN 链接
状态同步	后端定时生成配置 / 实时热更新 JSON
服务运行	Systemd / Docker Compose
安全	JWT 登录 / HTTPS / IP 黑名单 / Google reCAPTCHA
```

🛠️ 实施功能模块
1. 用户系统
注册 / 登录 / 修改资料

创建唯一 UUID（支持自选 alias）

每个用户绑定多个节点配置（共用 UUID）

数据库设计建议：

```
sql
users (
  id UUID PRIMARY KEY,
  email TEXT UNIQUE,
  password_hash TEXT,
  register_time TIMESTAMP,
  expire_time TIMESTAMP,
  data_limit_bytes BIGINT,
  used_bytes BIGINT
)
```
2. 订阅系统
生成 Clash / JSON / sing-box 多协议配置

支持订阅链接带 token（防止泄漏）

每个用户唯一订阅地址，如：

```
pgsql
https://sub.example.com/clash/uuid.yaml
https://sub.example.com/sbox/uuid.json
```
定时更新：订阅地址内容反映实时流量剩余、节点状态、是否过期

3. 配额控制（限流 / 限期）
后端根据用户 expire_time 和 data_limit_bytes 判断是否允许连接

每个节点 sing-box 配置包含如下限制：
```
json
{
  "type": "vless",
  "users": [
    {
      "uuid": "user-uuid",
      "flow": "xtls-rprx-vision",
      "limit": {
        "bytes": 10737418240,  // 10 GB
        "expire": "2025-08-01T00:00:00Z"
      }
    }
  ]
}
```
可选：使用 Redis 实现实时流量记录累加（或定时更新 used_bytes）

4. 支付系统（可扩展模块）
Stripe（推荐）
支持信用卡 / Apple Pay / Google Pay

提供付款页面（如 Stripe Checkout）

支付成功回调 webhook → 激活用户流量/时间

USDT 手动支付
用户后台生成收款地址（手动或借助 Tron API）

用户付款后上传截图或填入 TXID

管理员确认后分配流量/时间

可逐步接入链上验证（TronScan API）

5. 后台管理系统
功能建议：
```
功能	描述
用户列表	查看用户状态、用量、过期时间
手动续费	为某用户添加流量或延长时间
用户限速	设置带宽上限
查看连接数	sing-box 实时状态 API 接入
节点状态监控	调用 API 检查节点可达性
```

✅ 阶段四完成标准
可通过网页注册账号，获得专属订阅地址

支持 Clash/sing-box/v2rayN 自动订阅

支持到期自动禁用 / 超额限制连接

实现至少一种支付方式（Stripe / USDT）

后台管理员可查看与管理所有用户

节点运行正常，配置可热更新

📦 示例页面结构建议
/login：登录页

/register：注册页（带验证码）

/dashboard：订阅信息 + 节点说明

/payment：充值页

/admin：后台（需权限）

/sub/:uuid.yaml：Clash 订阅地址

/sub/:uuid.json：sing-box 订阅地址

🚀 后续阶段（阶段五预告）
```
功能	技术建议
高可用（负载均衡）	HAProxy + DNS failover
配置中心	etcd / Consul / GitOps
审计与日志	Loki + Grafana / ClickHouse 日志分析
客户端测速报告	客户端上报延迟、抖动用于推荐节点
API 网关	接入 Kong / NGINX / Zero-Trust 验证
中继部署	增加 relay 中转点，隐藏真实节点
```

如果你希望我提供：

阶段四完整前后端项目目录结构

支付回调处理样例（Stripe / Tron）

用户订阅配置生成模板

欢迎继续指定，我可以帮你生成原型代码或数据库 schema。是否继续进入阶段五的高可用与运营自动化体系？