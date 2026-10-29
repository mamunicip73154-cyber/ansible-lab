# 生产事故复盘案例：自动更新引发的 MySQL 停机与 systemd 启动死锁

> 事故日期：2026-10-23 | 环境：Ubuntu 24.04 LTS 虚拟机（MySQL 8 + Zabbix Server）  
> 等级模拟：生产环境按 P2 处置（核心数据库不可用，下游监控失效）

---

## 一、事故时间线（排查后还原）

| 时间          | 事件                                                                                                         |
| ----------- | ---------------------------------------------------------------------------------------------------------- |
| 14:24       | VM 重启，MySQL 随开机正常启动（enabled）                                                                               |
| 15:17:33~39 | unattended-upgrades 自动安全更新：libnss3、libcares2、**zlib1g**                                                    |
| 15:17:40    | apt 联动重启依赖 zlib1g 的服务：zabbix-agent 重启成功                                                                    |
| 15:17:42    | mysql.service 被停止，**但未跟随重启**（restart 顺序缺陷）                                                                 |
| 15:17:42 起  | zabbix-server 连不上数据库 → 启动失败 → Restart=on-failure **无限重启循环（3 小时）**                                          |
| 18:05       | **Zabbix Web 前端报「数据库错误：没有这样的文件或目录」才发现故障**（前端连 MySQL 走 localhost socket，socket 文件随 mysqld 停止而消失，PHP 报 2002） |
| 18:10       | `systemctl start mysql` 卡住 → 发现任务被 systemd 事务阻塞（WAITING）                                                   |
| 18:16       | cancel 清队列 → stop zabbix-server → start mysql → 全部恢复                                                       |

## 二、生产环境最佳处置流程（恢复优先，但保留现场）

### 第 0 步：确认影响面（1 分钟内）

- 谁挂了：`systemctl is-active mysql zabbix-server`（受影响服务清单）
- 波及谁：下游依赖（监控、应用、备份任务）
- **先通知，再动手**：生产上发变更群/打电话，明确"我在处置，预计 X 分钟"

### 第 1 步：快速止血（按破坏性从小到大尝试）

| 手段        | 本案例对应                                           | 适用场景                    |
| --------- | ----------------------------------------------- | ----------------------- |
| 重启故障服务    | `systemctl start mysql`                         | 大多数单服务故障                |
| 清理阻塞任务后重试 | `systemctl list-jobs` → `systemctl cancel`      | **本案例**：start 卡 WAITING |
| 杀进程强制终结循环 | `systemctl kill -s SIGKILL xx` + `reset-failed` | 重启风暴（Restart 循环）        |
| 按依赖顺序逐个拉起 | 先 DB 后 Zabbix（谁依赖谁最后起谁）                         | 多服务联动故障                 |
| 兜底重启机器    | `reboot`                                        | 以上全失败 + 持久化状态正确         |

**铁律：每一步操作前留痕（终端复制/截图），每一步后验证（is-active + 端口 + 实际功能）。**

### 第 2 步：保留现场

- `journalctl -u 服务名 -b` 本次开机完整日志
- `/var/log/apt/history.log`、`/var/log/syslog`、`/var/log/auth.log`
- 这些是复盘和甩锅（划掉）定责的证据

### 第 3 步：根因分析（RCA）

本次用到的证据链：

```
journalctl -u mysql -b          → 15:17:42 被"干净地"停止（Stopping = 有人主动停）
tail /var/log/apt/history.log   → 15:17:33 unattended-upgrade 升级 zlib1g（时间吻合）
systemctl list-jobs             → zabbix-server restart running 3h + mysql start WAITING
```

**时间线对齐法**：把"服务日志"和"系统事件"按时间排在一起，因果自然浮现。

## 三、排查方法论（可复用的五连）

| # | 动作     | 命令                                          | 本案例产出                     |
| - | ------ | ------------------------------------------- | ------------------------- |
| 1 | 服务活着吗  | `systemctl is-active xx`                    | mysql inactive            |
| 2 | 它经历了什么 | `journalctl -u xx -b`                       | 15:17 被停，无崩溃记录            |
| 3 | 谁干的    | `apt history` + `journalctl --since` 时间窗对齐  | unattended-upgrades       |
| 4 | 为什么起不来 | `systemctl list-jobs`                       | start 被 WAITING，前面有重启风暴   |
| 5 | 依赖关系   | `systemctl show xx -p After,Requires,Wants` | 确认 zabbix-server 依赖 mysql |

**核心思想：先看状态，再看日志，再对时间线，最后才动手。跳过前四步直接重启 = 赌博。**

## 四、预防手段（分四层）

### 1. 管住自动更新（本次根因）

```ini
# /etc/apt/apt.conf.d/50unattended-upgrades 关键配置
Unattended-Upgrade::Package-Blacklist {
    "zlib1g";        # 核心库黑名单（会引发大范围服务重启）
    "mysql";
    "openssl";
};
Unattended-Upgrade::Automatic-Reboot "false";
```

- 生产 DB/中间件机器：**关闭无人值守更新**，改为人工维护窗口
- 或至少：黑名单核心库 + 自动重启禁用 + 邮件通知开启

### 2. 服务单元加固（防重启风暴）

创建方式：`sudo systemctl edit zabbix-server`（自动生成 drop-in 文件，勿直接改原始单元文件）

```ini
# /etc/systemd/system/zabbix-server.service.d/override.conf
# 注意：systemd 单元文件不支持行尾注释，注释必须独立成行；
# StartLimit 两键在 systemd v230+ 归 [Unit] 段，放 [Service] 会被忽略

[Unit]
StartLimitIntervalSec=300
StartLimitBurst=5

[Service]
RestartSec=30
```

- **无限重启循环 = 告警风暴 + 日志洪水 + 事务阻塞**，必须设上限
- `systemctl edit` 保存即自动 `daemon-reload`
- 验证：`systemctl show zabbix-server -p StartLimitIntervalUSec,StartLimitBurst,RestartSec`
- 本案例若有此配置，zabbix-server 循环 5 次就放弃，mysql 也不会被堵 3 小时

### 3. 监控告警（让故障自己开口）

- 服务存活：Zabbix `systemd.unit[mysql.service]` 类监控项
- 端口探活：3306 每 30s 探测，不通立刻告警（而不是等用户/Navicat 发现）
- 日志关键字：`journalctl` 出现 "Stopping mysql" 且非维护时段 → 告警
- 依赖拓扑告警：DB 挂了要同时告警 DB 和它的下游

### 3.5 谁来监控监控自己？（本案例暴露的盲区）

- 本案例最讽刺的一点：**监控系统（Zabbix）自己就是受害者**，它挂了以后全网没有任何告警 ——  
  故障是用户打开监控页面时人肉发现的
- 生产做法：
  - **外部心跳**：用一台独立机器定时探测 Zabbix Web 健康页（curl HTTP 200），失败即告警
  - 双活/主备监控：Zabbix HA（6.0+ 自带）或第二套轻量监控（如 Uptime Kuma）互为备份
  - 自监控项：Zabbix Server 自身的 `zabbix[process]` 内部监控 + 服务器上跑的 agent 上报  
    （本案例 .139 上有 agent，其实完全能监控到本机 mysql 停止 —— 但 server 挂了没人收告警）

### 4. 流程与文档

- **Runbook**：每个核心服务一张"挂了怎么办"卡片（本复盘的处置流程就是雏形）
- **变更管理**：生产上任何"我 start 一下"都属于变更，先报备
- **依赖清单**：server 上跑的每个服务"依赖谁、被谁依赖"画成图（本次若已知 zabbix-server←mysql，第一时间就会先看 DB）

## 五、面试表述模板

> "我处理过一次自动安全更新引发的连锁故障：unattended-upgrades 升级 zlib1g 后重启依赖服务，MySQL 被停止且未自动恢复，下游 Zabbix Server 陷入无限重启循环，还导致后续 systemctl start 的事务阻塞。我先通过 journalctl 和 apt history 对齐时间线定位根因，用 systemctl cancel 清理阻塞事务后按依赖顺序恢复服务，最后通过 StartLimit 限制重启次数 + 黑名单核心库做了预防。这个案例让我理解了变更管理、依赖拓扑和重启风暴防控在生产中的意义。"

---

*整理：老查 2026-10-23 | 配套实验环境：ansible-lab（Ubuntu 24.04 × 3）*
