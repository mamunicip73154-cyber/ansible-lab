# ansible-lab — Linux 运维自动化实验环境

基于 Ansible 的多主机 Linux 运维自动化项目，覆盖 SSH 安全加固、监控体系部署、MySQL 全链路交付。

## 环境架构

| 主机 | IP | 角色 |
|---|---|---|
| ubuntu139 | 192.168.52.139 | 控制节点 + Zabbix Server + MySQL + Nginx + Uptime Kuma(Docker) |
| clone140 | 192.168.52.140 | 受管节点（SSH 加固：仅密钥 + 2222 端口 + fail2ban）+ MySQL + Zabbix Agent |
| clone79 | 笔记本 | 受管节点 + Zabbix Agent |

## 角色清单

| Role | 功能 | 亮点 |
|---|---|---|
| ssh_hardening | SSH 安全加固（禁 root/仅密钥/改端口） | 处理 Ubuntu 24.04 ssh.socket 激活机制兼容性 |
| fail2ban | 暴力破解防护 | 白名单/封禁策略变量化 |
| zabbix_agent | Zabbix Agent 部署与配置 | Jinja2 模板化配置，多机差异化（Server 白名单/Hostname） |
| installMysql | MySQL 全链路交付 | 装包/设密/建库/监控账号/凭据分发一条龙，Vault 管理敏感变量 |

## 特性

- **幂等交付**：所有角色二次执行 changed=0，任意时刻可安全重跑
- **变量分层**：defaults → host_vars → group_vars(Vault)，配置与逻辑解耦——主机 IP 迁移零代码改动
- **敏感信息**：Ansible Vault AES256 加密，密码文件与密文分离（.gitignore 强制隔离）
- **代码质量**：ansible-lint production profile + FQCN 全覆盖
- **分层监控**：Zabbix 采集业务指标，Uptime Kuma 独立心跳监控告警链路本身（Web 8082 + server 10051）

## 快速开始

```bash
# 全部角色幂等交付
ansible-playbook zabbix_playbook.yml -K
ansible-playbook ssh_hardening.yml -K
ansible-playbook installMysql.yml -K

# 主机体检（服务状态/磁盘阈值）
ansible-playbook health_check.yml -K
```

## 目录结构

```
├── ansible.cfg            # 项目级配置（inventory/vault 密码文件）
├── inventory.ini          # 主机清单
├── group_vars/all/        # 全局变量（vault.yml 加密，不入库）
├── host_vars/             # 主机差异化变量
├── roles/                 # 四个自研角色
│   ├── ssh_hardening/
│   ├── fail2ban/
│   ├── zabbix_agent/
│   └── installMysql/
├── health_check.yml       # 主机体检 playbook
├── docs/                  # 部署手册与事故复盘
└── README.md
```

## 文档

- 《Zabbix 7.0 部署实操手册》
- 《Ansible 入门实操手册》
- 《生产事故复盘：自动更新导致 MySQL 停机与启动死锁》
