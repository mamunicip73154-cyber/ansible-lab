# Ansible 入门实操手册（双机靶场版）

> 2026-10-17 定稿。目标环境：控制节点 = ubuntu(.139)，被管节点 = ubuntu(.139 自身) + 主机140(.140，你的克隆机)。  
> 学习路线：装环境 → 通密钥 → 写清单 → Ad-hoc 找感觉 → 第一个 Playbook → 最终项目（自动部署 Zabbix agent）。



---

## 第 0 步：角色划分（先建立心智模型）

```
控制节点（Control Node）= .139   ← 只在这台装 ansible，往别人身上推操作
被管节点（Managed Node） = .139 自己 + .140
通信方式                  = SSH（Ansible 没有客户端，被管机什么都不用装！）
```

**Ansible 与 Zabbix agent 的本质区别**：Zabbix 是"装客户端、被动上报"；Ansible 是"SSH 过去、推完就走"。  
这就是为什么它是运维自动化的第一选择——**机器上不需要预装任何东西**。

---

## 第 1 步：控制节点安装 Ansible（在 .139 上）

```bash
sudo apt update
sudo apt install -y ansible
ansible --version        # 记住这个输出版本， noble 仓库一般是 9.x / core 2.17
```

**坑点**：

- **不要 `pip install ansible`**——Ubuntu 24.04 按 PEP 668 禁止系统级 pip 安装，会报  
  `externally-managed-environment`。系统工具用 apt 装，这是 Debian 系的规矩
- apt 装完命令是 `ansible`、`ansible-playbook`，配置文件在 `/etc/ansible/`，但**我们不用它**——  
  用项目目录里的私有配置（见第 3 步），这是生产习惯：配置跟着项目走

---

## 第 2 步：打通 SSH 免密（Ansible 的生命线）

**现状**：`.139 → .140` 免密不通（Permission denied）。原因：`.139` 的 ubuntu 用户从未生成过密钥对。  
（你 Windows 的密钥能登两台机，是因为克隆机复制了 `.139` 上装好的 authorized_keys——那是给 Windows 用的，跟 .139 自己无关。）

### 2.1 在 .139 生成密钥对（已有就跳过，问它会覆盖）

```bash
ls -l ~/.ssh/id_ed25519.pub 2>/dev/null && echo "已有密钥，跳过" || ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
```

> `-N ""` = 私钥不设密码。练习环境求方便；生产上私钥必须设密码，这是面试必问的安全点。

### 2.2 把公钥推到两台被管机（要输密码）

```bash
# 推给 .140（密码 = 克隆机 ubuntu 密码，和 .139 相同）
ssh-copy-id ubuntu@192.168.52.140

# 推给自己（.139），让控制节点也能 SSH 管理自身
ssh-copy-id ubuntu@192.168.52.139
```

### 2.3 验证（两条都免密成功才算过）

```bash
ssh ubuntu@192.168.52.140 "hostname"
ssh ubuntu@192.168.52.139 "hostname"
```

**坑点**：

- `ssh-copy-id` 到 `192.168.52.139`（自己的公网 IP）是绕一圈 SSH 服务器的真连接，  
  不是 localhost 捷径——所以能真实验证"被管节点链路"
- 第一次连接会问 `Are you sure you want to continue connecting`，输 yes（这是主机指纹确认，  
  Ansible 里对应 `host_key_checking`，后面有开关）

---

## 第 3 步：项目目录 + Inventory（主机清单）

Ansible 的习惯：**一个项目一个目录**，配置、清单、剧本都放里面。

```bash
mkdir -p ~/ansible-lab && cd ~/ansible-lab
```

创建 `inventory.ini`：

```bash
cat > ~/ansible-lab/inventory.ini <<'EOF'
[zabbix_agents]
ubuntu139 ansible_host=192.168.52.139
clone140  ansible_host=192.168.52.140

[zabbix_agents:vars]
ansible_user=ubuntu
EOF
```

**逐行解读**：

- `[zabbix_agents]` = 组名。把机器分组，操作时可以点名单台、也可以整组轰
- `ubuntu139` / `clone140` = **别名**（你随便起），后面所有命令都用别名指代机器
- `ansible_host=` 真实 IP；`ansible_user=` 用哪个账号连

**坑点**：

- Inventory 文件**千万别命名为 `.yaml` 后缀但写 ini 语法**，也别放在会被 ansible 找错的地方。  
  每条 ad-hoc/playbook 命令都显式加 `-i inventory.ini`，不依赖默认路径（默认会找 `/etc/ansible/hosts`）

---

## 第 4 步：Ad-hoc 三连（先用起来，再理解）

### 4.1 ping（Ansible 的"你好世界"，它测的不是 ICMP，是完整 SSH+Python 链路）

```bash
cd ~/ansible-lab
ansible zabbix_agents -i inventory.ini -m ping
```

**预期**：两台机器各返回一条绿色的 `"changed": false, "ping": "pong"`。

**必踩坑**：第一次跑大概率报 `"msg": "Failed to connect to the host via ssh"` 或者卡在  
host key 确认——因为 Ansible 非交互模式没法输 yes。练习环境一键放行（**生产禁用此开关**）：

```bash
export ANSIBLE_HOST_KEY_CHECKING=False
```

（写进 `~/.bashrc` 永久生效。生产上的正解是提前统一分发 known_hosts。）

### 4.2 shell 模块（把你会的 Linux 命令变成"批量"）

```bash
ansible zabbix_agents -i inventory.ini -m shell -a "uptime && free -h | head -2"
ansible clone140   -i inventory.ini -m shell -a "ss -Htan state established | wc -l"
```

**感受一下**：第二条命令你今天在 .140 手工跑过——现在一条命令可以让 200 台机器同时跑。  
**这就是你驻场数百台机器的经验 + Ansible 的价值：手工批量 → 代码批量。**

### 4.3 setup 模块（机器的"体检报告"）

```bash
ansible ubuntu139 -i inventory.ini -m setup | head -30
```

返回 JSON：操作系统、内存、网卡、磁盘、Python 版本……这些叫 **facts（事实变量）**，  
Playbook 里可以直接引用（比如 `ansible_memtotal_mb`）。先用 `grep` 找几个你感兴趣的变量名。

---

## 第 5 步：第一个 Playbook（从"命令"升级到"代码"）

Ad-hoc 是"一次性命令"，Playbook 是**可重复执行的脚本**——自动化的本体。

创建 `~/ansible-lab/first_playbook.yml`：

```yaml
---
- name: 双机基础配置练习
  hosts: zabbix_agents
  become: true                 # 用 sudo 执行（被管机的 ubuntu 有 sudo 密码）

  tasks:
    - name: 确认 htop 已安装
      ansible.builtin.apt:
        name: htop
        state: present

    - name: 写一个标识文件
      ansible.builtin.copy:
        content: "managed by ansible on {{ inventory_hostname }}\n"
        dest: /etc/ansible-managed
        owner: root
        group: root
        mode: "0644"
```

执行（`-K` = 询问 sudo 密码，两台机的密码一样）：

```bash
cd ~/ansible-lab
ansible-playbook first_playbook.yml -i inventory.ini -K
```

**预期输出逐段看**：

- `GATHERING FACTS` ✔ —— 每次跑剧本 Ansible 自动先采集 facts
- 每条 task 显示 `changed` 或 `ok` —— **`ok` = 已经是目标状态、没动手；`changed` = 这次真的改了**
- 再跑一遍，全部变 `ok` —— **这就是幂等性（idempotency）**，Ansible 最重要的设计：  
  同一个剧本跑一百遍，结果都收敛到同一状态，不会越跑越乱

**验证成果**：

```bash
ansible zabbix_agents -i inventory.ini -m shell -a "cat /etc/ansible-managed"
```

两台机各自返回 `managed by ansible on ubuntu139/clone140`——注意 `{{ inventory_hostname }}`  
变量在两台机上渲染出了不同的值，这就是模板化。

**坑点**：

- YAML 对缩进零容忍：**两级空格**，绝不用 Tab。报 `syntax error` 九成是缩进
- `-K` 大写是 ask-become-pass；`-k` 小写是 ask-ssh-pass。免密打通后 `-k` 用不上，`-K` 还要用  
  （除非以后给 ubuntu 配 NOPASSWD sudo）

---

## 第 6 步：常见坑速查表

| 症状                                        | 原因              | 处理                                               |
| ----------------------------------------- | --------------- | ------------------------------------------------ |
| `Failed to connect ... Permission denied` | 免密没打通 / user 不对 | 重做第 2 步；检查 inventory 的 ansible_user              |
| 卡住不动或 host key 报错                         | 首次连接指纹确认        | `ANSIBLE_HOST_KEY_CHECKING=False`（仅练习）           |
| `externally-managed-environment`          | pip 装系统包被拒      | 用 apt 装 ansible，别碰 pip                           |
| YAML syntax error                         | 缩进/Tab          | 两级空格，编辑器开"显示空白"                                  |
| task 显示 `ok` 但你以为会改                       | 幂等：已符合目标状态      | 正常，这是特性                                          |
| sudo 要密码但没加 -K                            | become 密码没给     | 加 `-K`，或在 inventory 配 `ansible_become_pass`（练习用） |

---

## 第 7 步：阶段项目预告（本手册的终点）

`deploy_zabbix_agent.yml` —— 一条命令给任意新机器自动完成：

1. apt 安装 zabbix-agent（源配置 + 包）
2. 写入 `/etc/zabbix/zabbix_agentd.d/custom_tcp.conf`（UserParameter）
3. 设置 `Server=192.168.52.139`、`Hostname=机器名`
4. `systemctl enable --now zabbix-agent`（handler 触发重启）
5. Web 端自动纳管（调用 Zabbix API）

涉及新概念：`handler`（变更后触发重启）、`template`（Jinja2 模板替换配置）、`register/when`（条件判断）。  
这些等 Ad-hoc 和第一个 Playbook 玩熟了再上，还是那句话——**每一步都上机，别云学习**。
