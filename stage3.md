以下是 阶段三：半商用系统（多节点 + 监控 + 自动故障转移） 的完整架构与部署方案。该阶段目标是构建一个 自用可商用化的系统雏形，提升可靠性、可视化监控、基础容灾能力，支持多节点部署与运营。

🎯 阶段三目标
实现 多节点部署，可应对 IP 被封 / 节点丢包 / 地域优化

实现 故障自动切换（failover）

接入 实时监控平台（Grafana + Prometheus）

用户侧透明自动使用可用节点（或节点测速排序）

🧱 架构总览图
```
                        [ 用户客户端 ]
                               │
                      订阅多节点信息（YAML/JSON）
                               ▼
+----------------+     +----------------+     +----------------+
| VPS 节点 A     |     | VPS 节点 B     | ... | VPS 节点 N     |
| - sing-box     |     | - sing-box     |     | - sing-box     |
| - 多协议支持   |     | - 多协议支持   |     | - 多协议支持   |
+--------▲-------+     +--------▲-------+     +--------▲-------+
         │                        │                        │
         ▼                        ▼                        ▼
   [ sing-box-manager 或 订阅聚合器（后端） ]
         │
         ▼
  [ Prometheus + Grafana + Alertmanager ]
```
🧩 技术栈清单
```
分类	技术
多节点部署	Ansible / Shell 脚本自动部署
节点配置同步	Git pull / API 同步 / 手动
节点管理	sing-box-manager 支持多配置生成
订阅聚合	同一 UUID 多节点订阅输出
健康检测	sing-box failover，或外部探测脚本
监控系统	Prometheus + Node Exporter + Grafana
告警系统	Alertmanager + Telegram/Email 推送
负载分流	客户端 failover 或节点测速排序使用
中央管理	可选：自建后台统一管理节点状态与用户
```

🛠️ 实施步骤
1. 多节点自动部署
方法一：Ansible 自动化部署多台 VPS
编写 Ansible role（或 Bash 脚本）部署：

安装 sing-box

设置防火墙

同步 sing-box JSON 配置

设置 Systemd 管理

```
yaml
# ansible/roles/sing-box/tasks/main.yml 示例
- name: Install sing-box
  shell: curl -fsSL https://sing-box.app/install.sh | bash

- name: Copy config.json
  copy:
    src: files/{{ inventory_hostname }}/config.json
    dest: /etc/sing-box/config.json

- name: Enable and start sing-box
  systemd:
    name: sing-box
    enabled: yes
    state: restarted
```
建议每台节点配置不同端口或域名，增加抗封能力。

2. 订阅多节点聚合（给客户端）
支持单用户多节点订阅的生成：

使用 sing-box-manager 生成多节点配置

或写一个简易聚合器，将多节点 YAML 合并成一个订阅

示例：

```
bash
# 脚本拼接多个 Clash 节点为一个订阅
cat node1.yaml node2.yaml node3.yaml > all-nodes.yaml
```
也可以用 Go / Python 写一个小型聚合订阅 API。

3. 客户端自动故障切换配置（Failover）
示例（Clash Meta 多节点 failover）：
```
yaml
proxies:
  - name: node-a
    type: hysteria2
    server: node-a.com
    ...
  - name: node-b
    type: hysteria2
    server: node-b.com
    ...
proxy-groups:
  - name: auto
    type: url-test
    proxies:
      - node-a
      - node-b
    url: http://www.gstatic.com/generate_204
    interval: 300
```
示例（sing-box 客户端配置 failover）：
```
jsonc
"outbounds": [
  {
    "tag": "proxy",
    "type": "selector",
    "outbounds": ["node-a", "node-b"],
    "strategy": "latency"
  },
  {
    "tag": "node-a",
    "type": "vless",
    ...
  },
  {
    "tag": "node-b",
    "type": "vless",
    ...
  }
]
```
4. 节点可用性监控与告警
安装 Prometheus + Grafana
每个节点安装 Node Exporter / sing-box-exporter

Prometheus 拉取每台节点指标

Grafana 显示带宽、连接数、CPU、延迟等

Prometheus 配置多节点示例：
```
yaml
scrape_configs:
  - job_name: 'singbox'
    static_configs:
      - targets: ['node-a:9100', 'node-b:9100']
```
Alertmanager 设置告警（如节点掉线）
```
yaml
receivers:
  - name: 'telegram'
    telegram_configs:
      - bot_token: 'your_bot_token'
        chat_id: 'your_chat_id'
```
5. 节点健康探测（主动检测）
方法一：shell 脚本 + systemd + cron
```
#!/bin/bash
ping -c 2 1.1.1.1 > /dev/null || systemctl restart sing-box
```
方法二：客户端主动测速 + url-test 策略自动选择
✅ 阶段三完成标准
至少两个 VPS 节点部署完毕，配置一致（UUID、端口可变）

订阅系统支持输出多节点配置

客户端具备自动测速、故障切换能力

节点运行状态实时监控，Grafana 可查看带宽/连接数

节点掉线或负载过高时可收到告警

📦 附加优化建议
```
项目	建议
配置中心	使用 Git 仓库存放所有节点配置
多地域优化	按目标用户部署亚洲/美洲/欧洲节点
NAT 探测	自动探测 NAT VPS 实 IP（用于 Reality）
客户端智能排序	每次订阅请求动态返回延迟最优节点
日志分析	接入 Loki / Vector 分析连接记录与行为
```

📌 下一步：进入阶段四（小规模商用：用户注册、订阅、支付）
用户注册登录系统

订阅过期与流量控制

接入支付（USDT / Stripe）

多协议订阅管理前端化

是否继续进入 阶段四：商用系统 MVP（注册+支付+限额） 的详细规划？我也可以提供订阅系统源码结构或数据库模型建议。