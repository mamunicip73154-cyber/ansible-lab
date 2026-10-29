# Zabbix 7.0 LTS 部署实操手册

> **目标机器**：Ubuntu 24.04.4 LTS（VMware 虚拟机，192.168.52.139）  
> **已有环境**：MySQL 8.0.46、Nginx 1.24.0（80 端口已反向代理给 Tomcat 8080）、时区 Asia/Shanghai  
> **版本选择**：Zabbix **7.0 LTS**（1:7.0.31-1+ubuntu24.04）——LTS 支持到 2029 年  
> **本手册的每条命令、每个预期输出，都是先在你这台机器上实测过的**

---

## 一、端口与组件规划（先看这张表，理解自己在干什么）

Zabbix 不是"一个软件"，是**四个组件拼起来的系统**：

| 组件                    | 作用                   | 端口     | 状态 |
| --------------------- | -------------------- | ------ | -- |
| `zabbix-agent`        | 装在被监控机上，采集 CPU/内存/磁盘 | 10050  | 待装 |
| `zabbix-server`       | 大脑，接收数据、判断阈值、触发告警    | 10051  | 待装 |
| `zabbix-frontend-php` | Web 界面（PHP 写的）       | 走 8082 | 待装 |
| `zabbix-sql-scripts`  | 数据库初始化脚本（不含程序）       | —      | 待装 |

配套组件：

|          | 谁              | 说明                            |
| -------- | -------------- | ----------------------------- |
| Web 服务器  | **复用现有 Nginx** | 新增一个 `listen 8082` 的 server 块 |
| PHP 运行环境 | **php8.3-fpm** | 需新装，Zabbix 前端是 PHP 应用         |
| 数据库      | **复用现有 MySQL** | 新建 `zabbix` 库，不动现有数据          |

**数据流向**：

```
zabbix-agent(:10050) ──采集数据──> zabbix-server(:10051) ──写入──> MySQL(:3306)
                                                                      ↑
浏览器 ──> Nginx(:8082) ──> php-fpm ──> Zabbix Web 前端 ──读取──────┘
```

### 为什么 Web 用 8082，不用 80？

你的 80 端口现在是这样一个配置：

```nginx
server {
    listen 80 default_server;
    location / {
        proxy_pass http://127.0.0.1:8080/;   # 所有请求都转给 Tomcat
    }
}
```

它是 `default_server`，**所有 IP 直访的请求都被它接走转给 Tomcat**。Zabbix 挤在 80 上要么抢不到，要么互相覆盖。所以给 Zabbix 单独一个端口 8082，互不干扰。

> 更"生产"的做法是用域名区分（`server_name zabbix.local`）。等你熟悉了虚拟主机再升级，见附录 C。



---

## 第 0 步：确认前置条件

```bash
# 1. 系统与服务
lsb_release -d
systemctl is-active mysql nginx

# 2. 时区必须是 Asia/Shanghai（Zabbix 对时间敏感，时区错会导致图表时间全乱）
timedatectl | grep "Time zone"

# 3. 确认端口是空闲的
ss -tln | grep -E ":8082|:10050|:10051"

# 4. 磁盘余量（Zabbix + 数据库约需 2G）
df -h / | tail -1

# 5. MySQL 怎么登录（你这台是 auth_socket，必须用 sudo mysql）
sudo mysql -e "SELECT VERSION();"
```

**预期**：

| 检查                  | 预期结果                         |
| ------------------- | ---------------------------- |
| 系统                  | Ubuntu 24.04.4 LTS           |
| mysql / nginx       | 都是 `active`                  |
| 时区                  | `Asia/Shanghai (CST, +0800)` |
| 端口 8082/10050/10051 | **没有任何输出**（表示空闲）             |
| MySQL               | `8.0.46-0ubuntu0.24.04.4`    |
| 磁盘                  | 可用 40G 以上                    |

> **注意**：`sudo mysql` 能进，`mysql -uroot` 会报 `ERROR 1698` —— 这是 MySQL 8 的 `auth_socket` 插件，root 只允许从本地 socket 以系统 root 身份登录。这不是故障，是安全设计。

---

## 第 1 步：添加 Zabbix 官方源

**为什么必须加源**：Ubuntu 24.04 官方仓库里**没有** Zabbix（`apt-cache search ^zabbix` 只剩一个 `python3-pyzabbix`），必须用 Zabbix 官方仓库。

```bash
cd /tmp

# 下载 release 包（只有 8KB）
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb

# 安装它（它只是个"源配置包"，不含 Zabbix 程序）
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb

# 刷新索引
sudo apt update
```

**预期输出（末几行）**：

```
Get:6 https://repo.zabbix.com/zabbix/7.0/ubuntu noble InRelease [3,964 B]
Get:11 https://repo.zabbix.com/zabbix/7.0/ubuntu noble/main amd64 Packages [108 kB]
Fetched 188 kB in 4s
Reading package lists... Done
```

**验证源生效**：

```bash
apt-cache policy zabbix-server-mysql
```

**预期**：

```
zabbix-server-mysql:
  Installed: (none)
  Candidate: 1:7.0.31-1+ubuntu24.04      <-- 看到这个就成功了
```

**看看它写了什么**（值得看一眼，理解 deb822 格式）：

```bash
cat /etc/apt/sources.list.d/zabbix.sources
```

新版的 Ubuntu 用 `.sources`（deb822 格式）替代老的 `.list`，内容是这样的：

```
Types: deb deb-src
URIs: https://repo.zabbix.com/zabbix/7.0/ubuntu
Suites: noble
Components: main
Signed-By: /usr/share/keyrings/zabbix.gpg
```

（同一文件里还有一段 6.5 的源，`Enabled: no` 表示禁用状态）

> **排障**：如果 `apt update` 报 `NO_PUBKEY`，执行  
> `sudo apt-key adv --keyserver keyserver.zabbix.com --recv-keys <缺失的KEY>`，  
> 但正常情况用不到——release 包已经把 GPG key 装到 `/usr/share/keyrings/zabbix.gpg` 了。

---

## 第 2 步：先装 PHP-FPM（顺序不能反！）

```bash
sudo apt install -y php8.3-fpm php8.3-mysql php8.3-mbstring php8.3-bcmath \
                    php8.3-xml php8.3-gd php8.3-ldap php8.3-curl
```

**为什么必须先装 php-fpm**：Zabbix 的 `zabbix-nginx-conf` 包在安装时（postinst 脚本）会做这件事：

```sh
for dir in $(realpath /etc/php/*/fpm/pool.d); do
    ln -sf /etc/zabbix/php-fpm.conf "$dir/zabbix-php-fpm.conf"
done
```

它要往 `/etc/php/8.3/fpm/pool.d/` 里放软链接。**先装 Zabbix 后装 php-fpm，这个软链接就不会被创建**，你得手动补。

**预期输出**：一堆 `Setting up php8.3-xxx ...`，最后：

```
Processing triggers for php8.3-fpm ...
```

**验证**：

```bash
php -v                              # 应显示 PHP 8.3.6
systemctl is-active php8.3-fpm      # 应显示 active
ls -l /run/php/                     # 应看到 php8.3-fpm.sock
```

**预期**：

```
PHP 8.3.6 (cli) (built: ...)
active
srw-rw---- 1 www-data www-data 0 ... /run/php/php8.3-fpm.sock
```

> **为什么是 8.3**：Ubuntu 24.04 自带 PHP 8.3.6，而 Zabbix 7.0.10+ 已官方支持 PHP 8.3（7.0.31 完全兼容）。不用去加 ppa:ondrej 装别的版本。

---

## 第 3 步：安装 Zabbix 组件

```bash
sudo apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf \
                    zabbix-sql-scripts zabbix-agent
```

各包体积（实测）：

| 包                     | 体积      | 装了什么                                                     |
| --------------------- | ------- | -------------------------------------------------------- |
| `zabbix-server-mysql` | 1.7 MB  | `/usr/sbin/zabbix_server` + systemd 服务                   |
| `zabbix-frontend-php` | 7.5 MB  | Web 界面，2318 个文件装到 `/usr/share/zabbix`                    |
| `zabbix-sql-scripts`  | 7.8 MB  | 建表脚本 `/usr/share/zabbix-sql-scripts/mysql/server.sql.gz` |
| `zabbix-nginx-conf`   | 8 KB    | nginx 与 php-fpm 的配置**模板**                                |
| `zabbix-agent`        | 0.28 MB | 采集端                                                      |

**预期输出末尾**：

```
Setting up zabbix-agent (1:7.0.31-1+ubuntu24.04) ...
Setting up zabbix-frontend-php (1:7.0.31-1+ubuntu24.04) ...
```

**装完立刻检查三个关键文件**：

```bash
# 1. nginx 模板是否被自动软链接到 nginx 配置目录
ls -l /etc/nginx/conf.d/zabbix.conf

# 2. php-fpm 池是否被自动软链接
ls -l /etc/php/8.3/fpm/pool.d/zabbix-php-fpm.conf

# 3. 官方模板本体
ls -l /etc/zabbix/nginx.conf /etc/zabbix/php-fpm.conf
```

**预期**：前两个都是软链接，指向 `/etc/zabbix/` 下的本体。

> **这里有个坑，一定要知道**：`zabbix-nginx-conf` 的安装脚本**会自动**建软链接  
> `ln -s /etc/zabbix/nginx.conf /etc/nginx/conf.d/zabbix.conf`。  
> 而 nginx 的主配置里有 `include /etc/nginx/conf.d/*.conf;`  
> ——**也就是说这个 server 块已经被 nginx 加载了**。  
> 更麻烦的是：官方模板里 `listen` 和 `server_name` 两行**是注释掉的**（见附录 A）。  
> nginx 遇到没有 `listen` 的 server 块，会**默认监听 80**。  
> 而你的 80 端口是 `default_server`（指向 Tomcat），  
> 结果是：**不报错、不冲突，但 Zabbix 页面就是进不去**——最阴的那种故障。

---

## 第 4 步：建库并导入 schema

### 4.1 创建数据库和账号

```bash
sudo mysql
```

进入 MySQL 后逐条执行：

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'zabbix_2026';

GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

**几个说明**：

| 项                       | 为什么                                          |
| ----------------------- | -------------------------------------------- |
| `CHARACTER SET utf8mb4` | Zabbix 6.0+ 强制要求 utf8mb4，否则中文主机名会乱码          |
| `COLLATE utf8mb4_bin`   | 官方要求区分大小写的排序规则                               |
| 用户名密码                   | 记住 `zabbix` / `zabbix_2026`，第 5 步和 Web 向导都要填 |
| 只授权 `zabbix.*`          | 最小权限原则，这个账号只能碰自己的库                           |

### 4.2 导入 schema（这一步最耗时）

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | \
  mysql --default-character-set=utf8mb4 -uzabbix -pzabbix_2026 zabbix
```

会提示 `mysql: [Warning] Using a password on the command line interface can be insecure.` —— **这是警告不是错误**，忽略。

**实测数据**：

| 项      | 值                          |
| ------ | -------------------------- |
| 压缩包    | 4.0 MB                     |
| 解压后    | **23.5 MB**                |
| SQL 行数 | **190,597 行**              |
| 预计耗时   | 30～90 秒（虚拟机磁盘慢，别急着 Ctrl+C） |

导入期间没有进度提示，屏幕是静的。**耐心等回到命令提示符。**

**验证导入成功**：

```bash
mysql -uzabbix -pzabbix_2026 zabbix -e "SELECT COUNT(*) AS 表数量 FROM information_schema.tables WHERE table_schema='zabbix';"
```

**预期**：**203** 张表。（此值经 7.0.31 实测确认：早期版本约 170 张，7.0.x 已增至 203 张）

```bash
mysql -uzabbix -pzabbix_2026 zabbix -e "SELECT * FROM dbversion;"
```

**预期**：

```
dbversionid  mandatory  optional
1            7000000    7000030
```

> **这个数字是最硬的证据**：`7000000` 表示 7.0.0 基线版本，`7000030` 表示已升到 7.0.30。
> 只要这两个值不空、不为 0，就说明 schema 导入完整。**查表数量可能会因版本略有出入，查 dbversion 永远不会错。**

> **关于 `log_bin_trust_function_creators`（2026-10-12 实测修正）**：  
> ⚠️ **你的环境实测必须先开这个开关**——7.0.31 的 schema 在第 2494 行附近就有 CREATE FUNCTION/TRIGGER，  
> 而 MySQL 开着 `log_bin` 且导入账号（zabbix）没有 SUPER 权限，直接导会报：  
> `ERROR 1418 (HY000) at line 2494: You do not have the SUPER privilege and binary logging is enabled`  
> 正确顺序（三步，别漏最后一步）：
>
> ```sql
> SET GLOBAL log_bin_trust_function_creators = 1;   -- 导入前（你的环境必做）
> -- ...导入...
> SET GLOBAL log_bin_trust_function_creators = 0;   -- 导入后必关（安全隐患，别留）
> ```
>
> **原理**：MySQL 开着二进制日志（log_bin）时，拒绝普通账号创建存储函数/触发器——  
> 因为这些函数在主从复制时可能行为不一致。开这个开关 = 声明"我信任这份 schema"。

---

## 第 5 步：配置 zabbix-server

打开主配置文件：

```bash
sudo vim /etc/zabbix/zabbix_server.conf
```

**只需改一处**——找到第 129 行附近的 `DBPassword`：

```ini
### Option: DBPassword
#       Database password.
#       Comment this line if no password is used.
#
# Mandatory: no
# Default:
# DBPassword=
```

改成：

```ini
DBPassword=zabbix_2026
```

（删掉行首的 `#` 和空格，填上密码）

**为什么只改这一处**：这个包的默认配置里，另外两个关键参数**已经是填好的**（我实测过行号）：

| 行号  | 内容                   | 状态                                   |
| --- | -------------------- | ------------------------------------ |
| 93  | `# DBHost=localhost` | 注释 = 默认值 localhost（走 unix socket，正好） |
| 105 | `DBName=zabbix`      | **已配置**，不用动                          |
| 121 | `DBUser=zabbix`      | **已配置**，不用动                          |
| 129 | `# DBPassword=`      | **需要你填**                             |

用 grep 快速确认改对了：

```bash
grep -nE "^DBName|^DBUser|^DBPassword|^DBHost" /etc/zabbix/zabbix_server.conf
```

**预期**：

```
105:DBName=zabbix
121:DBUser=zabbix
129:DBPassword=zabbix_2026
```

> **顺带说个安全项**：`DBHost=localhost` 表示走 Unix socket 连数据库，不经过网络。  
> 这意味着你可以放心地把 MySQL 的 `bind-address` 从 `0.0.0.0` 改成 `127.0.0.1`  
> （现在它是 `0.0.0.0`，任何能连到这台机器的人都能试 3306 端口）。见附录 D。

---

## 第 6 步：配置 Nginx（核心步骤，改动很小）

要改的文件是 `/etc/zabbix/nginx.conf`（记住它是被软链接到 `/etc/nginx/conf.d/zabbix.conf` 的，改哪个都一样）。

```bash
sudo vim /etc/zabbix/nginx.conf
```

**只改最前面两行**——把注释打开并换掉值：

```nginx
server {
        listen          8082;        # 原来是 # listen 8080;
        server_name     _;           # 原来是 # server_name example.com;

        root    /usr/share/zabbix;   # 下面这些都不用动
        index   index.php;
        ...
```

**改动前后对比**：

|             | 改之前                                           | 改之后                     |
| ----------- | --------------------------------------------- | ----------------------- |
| listen      | `#        listen          8080;`（注释状态）        | `listen          8082;` |
| server_name | `#        server_name     example.com;`（注释状态） | `server_name     _;`    |

**为什么 `server_name _`**：下划线是个约定俗成的"无域名"占位符，表示"没匹配到具体域名时用我"。因为你是直接用 IP 访问，不需要真实域名。

**为什么不用 8080**：官方模板默认写 8080，但你机器上 8080 是 Tomcat。

**验证并生效**：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**预期**：

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

**验证 8082 在监听了**：

```bash
sudo ss -tlnp | grep 8082
```

**预期**：`LISTEN 0 511 0.0.0.0:8082 ... nginx`

> **对照复习**：这个文件和你在 `/etc/nginx/sites-available/default` 里配的反向代理是**同级**的——  
> 都是被 `nginx.conf` 里 `include /etc/nginx/conf.d/*.conf;` 和  
> `include /etc/nginx/sites-enabled/*;` 两条指令加载进来的 server 块。  
> 一个监听 80（代理给 Tomcat），一个监听 8082（跑 PHP）。

---

## 第 7 步：检查 PHP-FPM 池

Zabbix 用的是**独立的 php-fpm 进程池**（不是默认的 www 池），这样它的 `memory_limit`、`max_execution_time` 可以单独调。

```bash
# 1. 池配置文件在不在（第 3 步自动软链接的）
ls -l /etc/php/8.3/fpm/pool.d/zabbix-php-fpm.conf

# 2. 看它监听的 socket 路径
grep -E "^\[|^listen|^user|^pm.max_children" /etc/zabbix/php-fpm.conf

# 3. 重启 php-fpm 让新池生效
sudo systemctl restart php8.3-fpm

# 4. 确认 socket 真的建出来了
ls -l /run/php/zabbix.sock
```


**预期**：

```
[zabbix]
user = www-data
listen = /var/run/php/zabbix.sock
pm.max_children = 50
```

```
srw-rw-rw- 1 www-data www-data 0 ... /run/php/zabbix.sock
```

**如果第 1 步找不到文件**（说明安装顺序反了），手动补：

```bash
sudo ln -sf /etc/zabbix/php-fpm.conf /etc/php/8.3/fpm/pool.d/zabbix-php-fpm.conf
sudo systemctl restart php8.3-fpm
```

**为什么 nginx 和 php-fpm 能配合**：它们通过 **Unix socket 文件**通信，不需要网络端口。nginx 配置里写的是：

```nginx
fastcgi_pass    unix:/var/run/php/zabbix.sock;
```

php-fpm 配置里写的是：

```ini
listen = /var/run/php/zabbix.sock
```

**两个路径必须完全一致**——这是 LNMP 排障的第一检查项。（`/var/run` 是 `/run` 的软链接，所以 `ls /run/php/zabbix.sock` 也能看到同一个文件）

---

## 第 8 步：启动全部服务并验证

```bash
sudo systemctl restart zabbix-server zabbix-agent php8.3-fpm nginx

# 设开机自启（生产必做）
sudo systemctl enable zabbix-server zabbix-agent php8.3-fpm
```

**逐个验证（这是运维的标准动作，别跳）**：

```bash
# 1. 四个服务的状态
systemctl is-active zabbix-server zabbix-agent php8.3-fpm nginx

# 2. 端口监听情况
sudo ss -tlnp | grep -E ":8082|:10050|:10051"

# 3. Web 能否本地访问
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1:8082/

# 4. server 日志有没有报错
sudo tail -20 /var/log/zabbix/zabbix_server.log
```

**预期**：

| 检查 | 预期 |
|------|------|
| 四个服务 | `active active active active` |
| 端口 | 8082（nginx）、10050（zabbix_agentd）、10051（zabbix_server） |
| curl | `HTTP 200`（或 302 跳转到 setup.php） |
| 日志 | 出现 `Zabbix Server 7.0.31 ... started` 和 `server #0 started` |

**常见启动失败**：

| 日志报错 | 原因 | 解法 |
|---|---|---|
| `cannot connect to database: Access denied` | 第 5 步密码填错 | 核对 `DBPassword` |
| `Cannot connect to the database. Exiting...` | MySQL 没启动 | `sudo systemctl start mysql` |
| `connection to database 'zabbix' failed: Unknown database` | 第 4 步建库没成功 | 回第 4 步 |
| `Address already in use` | 10051 被占 | `sudo ss -tlnp \| grep 10051` 找元凶 |

**关于进程身份**：Zabbix 官方包的 systemd 服务文件里**没有 `User=` 指令**，所以 `zabbix_server` 默认以 **root** 运行。这不理想（和你学过的 Tomcat 那课是一个道理），但官方包一直这么设计。生产加固可参考附录 E。

---

## 第 9 步：Web 初始化向导

**浏览器打开**（注意写全协议和端口）：

```
http://192.168.52.139:8082
```

**向导流程**：

1. **Welcome** → 选语言 `Chinese (zh_CN)` → `Next step`
2. **Check of pre-requisites** → 全部 `OK` 才继续 → `Next step`
3. **Configure DB connection** → 填：

   | 字段 | 值 |
   |---|---|
   | Database host | `localhost` |
   | Database port | `0`（用默认） |
   | Database name | `zabbix` |
   | User | `zabbix` |
   | Password | `zabbix_2026` |

   → `Next step`

4. **Zabbix server details** → 全默认（Host: `localhost`，Port: `10051`，Name: `Zabbix server`）→ `Next step`
5. **Summary** → `Next step`
6. **Install** → 完成

**默认登录**：

```
用户名：Admin
密码：  zabbix
```

> **首次登录立刻改密码**：右上角头像 → `User settings` → `Change password`。
> 默认密码是全网公开的，生产环境不改等于开门。

### 如果第 6 步报"无法写入配置文件"

向导结束时要在 `/usr/share/zabbix/conf/` 下生成 `zabbix.conf.php`，而这个目录属主是 root。表现为报错 `Cannot write configuration file` 或 `Unable to create configuration file`。

**解法（二选一）**：

**方案 1：先手动创建文件，交给 www-data**（推荐）

```bash
sudo cp /usr/share/zabbix/conf/zabbix.conf.php.example /usr/share/zabbix/conf/zabbix.conf.php
sudo chown www-data:www-data /usr/share/zabbix/conf/zabbix.conf.php
sudo chmod 640 /usr/share/zabbix/conf/zabbix.conf.php
```

然后重新走一遍向导，它会直接写进这个文件。

**方案 2：临时放开目录写权限，装完收回来**

```bash
sudo chown www-data:www-data /usr/share/zabbix/conf
# ...走完向导...
sudo chown root:root /usr/share/zabbix/conf
sudo chmod 755 /usr/share/zabbix/conf
sudo chown www-data:www-data /usr/share/zabbix/conf/zabbix.conf.php
```

**验证文件生成成功**：

```bash
sudo grep -E "DBName|DBUser|DBPassword|server" /usr/share/zabbix/conf/zabbix.conf.php
```

**预期**：能看到你填的 `zabbix` / `zabbix_2026`。

---

## 第 10 步：添加第一台被监控主机

Zabbix 装好是**空的**，一台主机都没有。你现在把它自己加上去。

### 10.1 Agent 端的配置（被监控机侧）

```bash
sudo vim /etc/zabbix/zabbix_agentd.conf
```

改两行：

```ini
Server=127.0.0.1              # 允许哪个 server 来取数据（本机监控填自己）
ServerActive=127.0.0.1        # 主动推送模式的目标
Hostname=ubuntu               # 主机名，必须和 Web 上填的完全一致
```

```bash
sudo systemctl restart zabbix-agent
grep -E "^Server|^ServerActive|^Hostname" /etc/zabbix/zabbix_agentd.conf
```

**验证 agent 能应答**（这是判断 agent 是否正常的关键命令）：

```bash
zabbix_get -s 127.0.0.1 -p 10050 -k "agent.version"
```

**预期**：`7.0.31`

如果报 `command not found`，装上工具包：`sudo apt install -y zabbix-get`

**再取一个真实指标**：

```bash
zabbix_get -s 127.0.0.1 -p 10050 -k "system.cpu.load[all,avg1]"
zabbix_get -s 127.0.0.1 -p 10050 -k "vm.memory.utilization"
```

**预期**：返回具体数字，比如 `0.04`、`21.36`。

### 10.2 Web 端添加主机

界面是中文的（你在向导里选了 zh_CN）：

1. 左侧 `数据采集` → `主机` → 右上角 `创建主机`
2. 填写：

   | 标签页 | 字段 | 值 |
   |---|---|---|
   | 主机 | 主机名称 | `ubuntu`（**必须和 agent 配置里的 Hostname 一模一样**） |
   | 主机 | 可见名称 | `Ubuntu测试机` |
   | 主机 | 主机群组 | 选 `Linux servers`（点 `选择` 按钮） |
   | 主机 | 接口 | 点 `添加` → `Agent` → IP 填 `127.0.0.1` → 端口 `10050` |
   | 模板 | 链接新模板 | 输入框里搜 `Linux by Zabbix agent` → 选中 → `添加` |

3. 点 `添加`

**预期**：主机列表里出现 `ubuntu`，**可用性**那一列的两个图标会变成绿色 `ZBX`。

**如果 ZBX 是红色的**：

| 现象 | 原因 | 排查 |
|---|---|---|
| 红色，鼠标悬停提示 `Connection refused` | agent 没跑 | `systemctl status zabbix-agent` |
| 红色，提示 `Get value from agent failed` | `Server=` 里没包含 server 的 IP | 检查 `/etc/zabbix/zabbix_agentd.conf` |
| 灰色，没反应 | 主机名不匹配 或 端口填错 | 核对 Hostname 与接口配置 |

**也可以让 agent 自己上报**（主动模式，不需要 Web 配置接口）：

`数据采集` → `自动注册` 里能看到 agent 主动注册的记录。这是几百台机器批量纳管的惯用方式，值得知道。

### 10.3 等 1-2 分钟看数据

左侧 `监测` → `最新数据` → 筛选主机 `ubuntu`。

**预期**：几十个监控项开始出现数值——CPU 负载、内存使用率、磁盘可用空间、网卡流量、进程数等。刚点进去可能有"没有数据"，**等 1-2 分钟**（采集间隔默认 1 分钟）。

再看 `监测` → `仪表盘` → `Zabbix server health`，能看到一堆图表。

---

## 第 11 步：配一个告警（可选，但强烈建议做）

这是 Zabbix 真正有价值的地方，也是面试会问的。

### 11.1 建一个"能收到"的媒介（先别急着发邮件）

**简单版：用"自定义告警脚本"演示完整链路**

其实最省事的验证方式是：先建一个必定触发的触发器，看 Zabbix 是否产生问题事件。

1. `数据采集` → `主机` → 点 `ubuntu` → `触发器` → `创建触发器`
2. 填写：

   | 字段 | 值 |
   |---|---|
   | 名称 | `测试告警-内存使用率过高` |
   | 严重性 | `警告` |
   | 表达式 | 点 `添加` → 选 `Zabbix agent` 里的 `Memory utilization` → 常量框里填 **`1`**（**只填数字 1，表示 1%**） |

   > ⚠️ **常量框里绝对不能写 `%` 号！**（2026-10-13 实测踩坑）
   > 写成 `{...}>="1%"` 会报 `Cannot evaluate expression: right operand "1%" is not a numeric value`，
   > 触发器状态变成「未知」，**永远不触发**。
   >
   > 原因：`Memory utilization` 这个监控项**本身单位就是 `%`**，返回值形如 `21.36`——
   > 表达式只需要和数字比较。**单位写在监控项的「单位」字段里，和表达式无关。**
   > 想表达"90%"就写 `>=90`。

3. `添加`

**预期**：1 分钟内，右上角铃铛图标出现红色数字，`监测` → `问题` 里出现这条告警。

**验证完把阈值改回 `>= 90`**，否则告警一直响。

> **排障提示**：触发器不触发时，第一件事是去 `数据采集 → 主机 → ubuntu → 触发器` 看有没有**黄色警告三角**。
> 鼠标悬停会显示具体错误（表达式非法、依赖的监控项没数据等）。
> 命令行等价查法（免 sudo）：
> ```bash
> mysql -uzabbix -pzabbix_2026 zabbix -e "SELECT triggerid,description,state,error FROM triggers WHERE error<>'';"
> ```
> `state=1` 表示「未知」（通常是表达式算不出来），正常应为 `state=0`。

### 11.2 告警链路的三层结构（要理解，不只是点界面）

```
触发器（什么时候算故障）              ← 11.1 已完成 ✔
    ↓ 产生 Event
动作（谁来处理这个事件 + 发给谁）      ← 还差 ✗
    ↓ 调用媒介
媒介（邮件 / 钉钉 / 企业微信 / 脚本）  ← 还差 ✗
```

**关键认知：触发器只负责"发现问题"，它自己发不出任何通知。**
触发器建好后，`监测` → `问题` 里会亮红，但**不会有任何人收到消息**——
2026-10-13 实测你的环境正是这个状态：

| 检查项（免 sudo 直读库） | 实测状态 |
|---|---|
| 动作 `Report problems to Zabbix administrators` | **status=1（禁用）** |
| Admin 用户绑定的媒介（`media` 表） | **空** |
| 媒介类型 Email | 内置但**未配 SMTP** |

```bash
# 一条命令看清"通知链"配没配
mysql -uzabbix -pzabbix_2026 zabbix -e "SELECT actionid,name,status FROM actions WHERE eventsource=0; SELECT u.username,m.mediatypeid,m.sendto,m.active FROM users u LEFT JOIN media m ON m.userid=u.userid;"
```

所以下面三小节，才是把告警**真正送到人手上**的部分。

### 11.3 建动作（Action）——决定"事件来了怎么办"

`告警` → `动作` → `触发器动作` → 找到 `Report problems to Zabbix administrators` → 点进去：

1. **先启用它**：右上角「已启用」开关（官方默认是**禁用**的，这是新手最容易漏的一步）
2. 切到「操作」标签页 → `添加`：
   - 发送至用户：`Admin`
   - 仅送到：你建的媒介（下面的 `练习脚本`）
3. 切到「恢复操作」标签页 → 同样 `添加` 一条 → **否则故障恢复后不会通知你**
4. `更新`

### 11.4 建媒介 + 绑定到用户（练习环境用"脚本"最省事）

**练习环境别去折腾真实 SMTP**——用自定义脚本，10 秒见效：

**① 建媒介类型**：`告警` → `媒介` → `创建媒介类型`

| 字段 | 值 |
|---|---|
| 名称 | `练习脚本` |
| 类型 | `脚本` |
| 脚本名称 | `alert_demo.sh` |
| 脚本参数 | `{ALERT.SENDTO}` 换行 `{ALERT.SUBJECT}` 换行 `{ALERT.MESSAGE}`（三行分别填） |

**② 在服务器上写脚本**（路径必须是 `AlertScriptsPath` 指向的目录）：

```bash
# 先确认路径
grep -n AlertScriptsPath /etc/zabbix/zabbix_server.conf

# 写脚本（默认目录 /usr/lib/zabbix/alertscripts）
sudo tee /usr/lib/zabbix/alertscripts/alert_demo.sh >/dev/null <<'EOF'
#!/bin/bash
echo "[$(date '+%F %T')] 收件人=$1 | 主题=$2 | 内容=$3" >> /tmp/zabbix_alert.log
EOF

sudo chmod +x /usr/lib/zabbix/alertscripts/alert_demo.sh
sudo chown zabbix:zabbix /usr/lib/zabbix/alertscripts/alert_demo.sh
```

**③ 把媒介绑到用户**：`管理` → `用户` → `Admin` → 「媒介」标签页 → `添加`

| 字段 | 值 |
|---|---|
| 类型 | `练习脚本` |
| 收件人 | `admin@local`（这是脚本的 `$1`，随便填） |
| 启用时间 | 默认 `1-7,00:00-24:00` |

→ `更新`

**④ 验证**：把 11.1 的阈值临时改回 `1`，等 1 分钟，然后在 VM 里：

```bash
cat /tmp/zabbix_alert.log
```

出现 `[时间] 收件人=admin@local | 主题=Problem: ...` → **整条链路打通**。

> **为什么用脚本而不是邮件？** 生产上的钉钉/企业微信告警，本质就是「Webhook 媒介 + HTTP 请求」，
> 和这里的脚本媒介是同一个套路：**动作 → 媒介 → 外部程序**。
> 先把脚本链路走通，换 Webhook 只是把 `echo 到文件` 改成 `curl 到机器人地址`。

### 11.5 `报表` → `动作日志`：告警没收到，第一个来看这里

| 动作日志里的状态 | 含义 |
|---|---|
| `已发送` | 动作执行了、媒介也调用了（对方收没收到是另一回事） |
| `失败` + 错误信息 | 媒介调用失败（脚本没执行权限、SMTP 连不上） |
| **什么都没有** | 动作根本没触发 → 回去查动作是否启用、条件是否匹配、用户有没有绑媒介 |

**三层排障顺序永远从后往前**：没收到 → 先看动作日志 → 再看媒介 → 最后才怀疑触发器。

生产环境常用的是 `Webhook`，对接钉钉/企业微信机器人，这个等你 Zabbix 玩熟了再上。

---

## 排障速查表

| 现象 | 可能原因 | 一条命令定位 |
|---|---|---|
| 浏览器打开 8082 空白/超时 | nginx 没监听 8082 | `sudo ss -tlnp \| grep 8082` |
| 打开 8082 出现**汤姆猫** | 你访问的是 80，或 nginx 模板还监听着 80 | `curl -I 127.0.0.1:8082` |
| 显示 PHP 源码而不是页面 | php-fpm 没跑 或 socket 路径不一致 | `ls /run/php/zabbix.sock` |
| `502 Bad Gateway` | php-fpm 挂了 | `systemctl status php8.3-fpm` |
| `404 File not found`（PHP 报的） | SCRIPT_FILENAME 路径不对 | `grep SCRIPT_FILENAME /etc/zabbix/nginx.conf` |
| Web 报 `Database error` | DB 密码/库名不对 | `sudo tail -30 /var/log/zabbix/zabbix_server.log` |
| 主机 ZBX 图标红/灰 | agent 配置或主机名不匹配 | `zabbix_get -s 127.0.0.1 -k agent.ping` |
| 图表没有数据 | 监控项还没采集 / server 没跑 | `systemctl is-active zabbix-server` |
| 页面时间不对 | 时区没设 | `timedatectl` |

**看日志的三个位置**：

```bash
sudo tail -f /var/log/zabbix/zabbix_server.log     # server 端（最常看这个）
sudo tail -f /var/log/zabbix/zabbix_agentd.log     # agent 端
sudo journalctl -u zabbix-server -n 30 --no-pager  # systemd 视角
sudo tail -f /var/log/nginx/error.log              # nginx/php 转发问题
```

---

## 附录 A：官方 nginx 模板原文（我拆包拿到的）

这是 `zabbix-nginx-conf` 包里的 `/etc/zabbix/nginx.conf` 原样内容，供你对照：

```nginx
server {
#        listen          8080;
#        server_name     example.com;

        root    /usr/share/zabbix;

        index   index.php;

        location = /favicon.ico {
                log_not_found   off;
        }

        location / {
                try_files       $uri $uri/ =404;
        }

        location /assets {
                access_log      off;
                expires         10d;
        }

        location ~ /\.ht {
                deny            all;
        }

        location ~ /(api\/|conf[^\.]|include|locale) {
                deny            all;
                return          404;
        }

        location /vendor {
                deny            all;
                return          404;
        }

        location ~ [^/]\.php(/|$) {
                fastcgi_pass    unix:/var/run/php/zabbix.sock;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_index   index.php;

                fastcgi_param   DOCUMENT_ROOT   /usr/share/zabbix;
                fastcgi_param   SCRIPT_FILENAME /usr/share/zabbix$fastcgi_script_name;
                fastcgi_param   PATH_TRANSLATED /usr/share/zabbix$fastcgi_script_name;

                include fastcgi_params;
                ...
        }
}
```

**值得注意的三处安全设计**（官方模板写得挺讲究）：

| 配置 | 作用 |
|---|---|
| `location ~ /(api\/\|conf[^\.]\|include\|locale) { deny all; }` | 禁止直接访问 `conf/`、`include/`、`locale/` 目录——防止别人下载你的 `zabbix.conf.php`（里面有数据库密码！） |
| `location /vendor { deny all; }` | 禁止访问 PHP 依赖库目录 |
| `location ~ /\.ht { deny all; }` | 拒绝 `.htaccess` 之类的隐藏文件 |

> 这几条就是"默认安全配置"的范例：**不是靠管理员记得去关，而是默认就关着**。

## 附录 B：官方 php-fpm 池模板原文

```ini
[zabbix]
user = www-data
group = www-data

listen = /var/run/php/zabbix.sock
listen.owner = www-data
listen.allowed_clients = 127.0.0.1

pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
pm.max_requests = 200

php_value[session.save_handler] = files
php_value[session.save_path]    = /var/lib/php/sessions/
php_value[max_execution_time] = 300
php_value[memory_limit] = 128M
php_value[post_max_size] = 16M
php_value[upload_max_filesize] = 2M
php_value[max_input_time] = 300
php_value[max_input_vars] = 10000
```

**两个参数值得理解**：

- `pm.max_children = 50` —— 最多 50 个 PHP 进程。每个约 30-50MB，理论上限 2.5GB。你这台 8GB 内存跑得动，但生产上要按内存倒推。
- `php_value[max_input_time] = 300` —— PHP 接收数据的最长等待时间。Zabbix 前端提交的数据可能很大（比如批量导入主机），所以比默认的 60 秒放宽了。

## 附录 C：进阶——给 Zabbix 配个域名

等你熟悉了虚拟主机（`server_name` 分流），可以把 8082 升级成域名访问。

**1. 在 nginx 上加一个 80 端口的 server 块**（跟 Tomcat 那个并存，靠域名区分）：

```bash
sudo tee /etc/nginx/conf.d/zabbix-domain.conf > /dev/null << 'EOF'
server {
    listen 80;
    server_name zabbix.local;

    root /usr/share/zabbix;
    index index.php;
    # ...其余配置从 /etc/zabbix/nginx.conf 复制...
}
EOF
```

**2. 在 Windows 宿主机加 hosts 映射**（推荐用 VSCode 打开改，管理员权限）

文件：`C:\Windows\System32\drivers\etc\hosts`

```
192.168.52.139  zabbix.local
```

**3. 浏览器访问** `http://zabbix.local`

**原理**：nginx 收到请求后看 HTTP 头里的 `Host:` 字段——`zabbix.local` 匹配到 Zabbix 的 server 块，IP 直访则落到 `default_server`（Tomcat）。**同一台机器、同一个 80 端口，靠域名分流到不同服务**，这就是"虚拟主机"。

## 附录 D：顺手处理的三个遗留安全项

> 📖 **原理 / 攻击场景 / 命令验证过程 / DBeaver 替代方案，见同目录《附录D-三个安全项详解.md》**
> 本附录只保留可直接执行的操作 + 验收。以下命令均经 **2026-10-13 干跑实测**，不是纸上推演。

**执行顺序有讲究**：**D.1 → D.2 → D.3**（先做影响最小的，最后做会切断 DBeaver 的那个）。
**铁律：一项一验，别攒着一起测**（跟你那个 `systemctl restart` 四连的教训同理）。

### D.1 nginx 关闭版本号暴露（先做，风险为零）

```bash
# ⚠️ 文件里那一行是 <TAB># server_tokens off;（行首是 TAB，不是 #）
#    所以正则必须用 [[:space:]]* 吃掉前导空白；写 ^# 会【静默失败】——不报错，但什么也没改
sudo sed -i 's/^[[:space:]]*#[[:space:]]*server_tokens off;/    server_tokens off;/' /etc/nginx/nginx.conf
sudo nginx -t && sudo systemctl reload nginx
```

**必须验证**（`sed -i` 匹配失败时退出码仍是 `0`，不能只看退出码）：

```bash
grep -n "server_tokens" /etc/nginx/nginx.conf          # 应看到不带 # 的那行
curl -sI http://127.0.0.1:8082/ | grep -i "^Server:"   # 应只剩 Server: nginx
```

**预期**：从 `Server: nginx/1.24.0 (Ubuntu)` 变成 `Server: nginx`。

### D.2 MySQL 只监听本机

> **前提已验证**（2026-10-13）：你的 Zabbix **走 unix socket**——
> `information_schema.processlist` 里 host 列全是 `localhost`（无端口号），且 `ss -tn | grep 3306` 为空。
> 所以改 `bind-address` 对 Zabbix **零影响**。

```bash
sudo sed -i 's/^bind-address\s*=\s*0.0.0.0/bind-address\t\t= 127.0.0.1/' /etc/mysql/mysql.conf.d/mysqld.cnf
sudo sed -i 's/^mysqlx-bind-address\s*=\s*0.0.0.0/mysqlx-bind-address\t= 127.0.0.1/' /etc/mysql/mysql.conf.d/mysqld.cnf
grep -n "bind-address" /etc/mysql/mysql.conf.d/mysqld.cnf
sudo systemctl restart mysql
ss -tln | grep 3306      # 预期：127.0.0.1:3306 和 127.0.0.1:33060
```

> **⚠️ 第二条不能漏**：`mysqlx-bind-address` 对应 **33060 端口**（MySQL 8 的 X Protocol）。
> 只改第一行，33060 照样对全网开着——很多人就是漏在这。

> **⚠️ 代价**：DBeaver 从 Windows 直连会断。正确解法是 **SSH 隧道 + 专用账号**（详解文档第三节），
> **不要**为了保住 DBeaver 而回退掉加固。

### D.3 删掉 `root@%` 账号（最后做）

**这个账号哪来的**：是你当初为了 DBeaver 远程连库，按 `DBeaver连接Ubuntu_MySQL指南.md` 第 19-20 行建的
（`CREATE USER 'root'@'%' ... GRANT ALL ... WITH GRANT OPTION`）。不是被入侵，是历史配置，现在该收了。

**① 先建替代账号**（不建就先删的话，DBeaver 立刻断）：

```bash
sudo mysql <<'SQL'
CREATE USER IF NOT EXISTS 'dba'@'localhost' IDENTIFIED BY 'Dba_2026#local';
GRANT ALL PRIVILEGES ON *.* TO 'dba'@'localhost' WITH GRANT OPTION;
CREATE USER IF NOT EXISTS 'dba'@'127.0.0.1' IDENTIFIED BY 'Dba_2026#local';
GRANT ALL PRIVILEGES ON *.* TO 'dba'@'127.0.0.1' WITH GRANT OPTION;
FLUSH PRIVILEGES;
SELECT user,host,plugin FROM mysql.user ORDER BY user,host;
SQL
```

**② 再删**：

```bash
sudo mysql -e "SELECT user,host,plugin FROM mysql.user WHERE user='root';"
sudo mysql -e "DROP USER IF EXISTS 'root'@'%';"
sudo mysql -e "FLUSH PRIVILEGES;"
```

**预期**：`root` 只剩 `localhost` 一行（plugin = `auth_socket`）。`sudo mysql` 不受影响。

> **⚠️ auth_socket 的坑（会咬人）**：`root@localhost` 用的是 `auth_socket` 插件，**它根本不认密码**。
> 所以 DBeaver 哪怕配好了 SSH 隧道，用 `root` + 密码照样被拒（`Access denied`）。
> **必须用上面新建的 `dba` 账号。** 详见详解文档 2.4 / 3.4 节。

### D.4 全量验收

```bash
ss -tln | grep 3306                                    # 只剩 127.0.0.1
curl -sI http://127.0.0.1:8082/ | grep -i "^Server:"   # Server: nginx
systemctl is-active zabbix-server zabbix-agent php8.3-fpm nginx mysql
curl -s -o /dev/null -w "Zabbix Web: HTTP %{http_code}\n" http://127.0.0.1:8082/
sudo mysql -e "SELECT COUNT(*) AS tables_cnt FROM information_schema.tables WHERE table_schema='zabbix';"
```

**预期**：`127.0.0.1:3306` / `Server: nginx` / `active × 5` / `HTTP 200` / `tables_cnt 203`。
最后那条 **203 是"加固没弄坏 Zabbix"的硬证据**。

### D.5 顺手学会列"暴露面清单"

```bash
ss -tln | grep -vE "127.0.0.1|::1"
```

**好习惯：每次装完一个新服务就立刻跑一遍**，看它多开了哪些门、该不该对外。
（本次侦察发现 10051 zabbix-server、8080 Tomcat 也在对外监听，属进阶话题，这次不动。）

## 附录 E：进阶——Zabbix Server 不用 root 跑

官方 systemd 服务文件里没有 `User=`，默认 root 运行。生产加固方式：

```bash
# 1. 建专用用户
sudo useradd -r -s /usr/sbin/nologin zabbix 2>/dev/null || echo "用户已存在"

# 2. 在 drop-in 目录里加覆盖配置（不改动官方文件，升级不丢）
sudo mkdir -p /etc/systemd/system/zabbix-server.service.d
sudo tee /etc/systemd/system/zabbix-server.service.d/user.conf > /dev/null << 'EOF'
[Service]
User=zabbix
Group=zabbix
EOF

# 3. 把需要的目录交给它
sudo chown -R zabbix:zabbix /var/log/zabbix /var/lib/zabbix /run/zabbix

# 4. 重载并重启
sudo systemctl daemon-reload
sudo systemctl restart zabbix-server
```

**`/etc/systemd/system/xxx.service.d/` 这个机制要记住**：它叫 drop-in 覆盖，
`daemon-reload` 时 systemd 会把 `/etc/systemd/system/` 下的同名覆盖片段**合并**到 `/usr/lib/systemd/system/` 的原文件上。
好处是：**软件包升级时你的自定义配置不会被覆盖掉**。这是生产环境改 systemd 服务的标准姿势。

---

## 最后：这套东西装完，你能在简历上写什么

```
监控告警体系：独立完成 Zabbix 7.0 LTS 部署（Server + Agent + Web）
    - Nginx + PHP-FPM(独立进程池) + MySQL 完整链路搭建
    - 自定义触发器与告警动作，实现主机资源监控与阈值告警
    - 处理过 PHP socket 通信、数据库 schema 导入、systemd drop-in 覆盖等问题
```

**更值钱的不是"会装"，而是能讲清楚**：

- 为什么 agent 和 server 要分开（采集与计算解耦、agent 轻量、可跨网络）
- 10050/10051 两个端口的区别（agent 被动/主动模式的差异）
- 触发器 → 动作 → 媒介 三层为什么这么设计（复用、解耦）
- 告警误报怎么处理（维护期、依赖关系、分层阈值）

这些是你驻场管几百台机器时天天在想的事，只是换了个工具表达。
