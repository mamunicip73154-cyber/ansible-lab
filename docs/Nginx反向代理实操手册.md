# Nginx location 与反向代理实操手册

> 基于你 `192.168.52.139` 的真实配置编写
> 当前 server 块（`/etc/nginx/sites-available/default`）有效内容只有 10 行：

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;
    server_name _;
    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

## 开工前：先备份（习惯问题）

```bash
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.bak
```

**理由**：这一课要反复改同一个 server 块，改崩了一秒回滚：

```bash
sudo cp /etc/nginx/sites-available/default.bak /etc/nginx/sites-available/default
sudo systemctl reload nginx
```

> 生产环境改任何配置前都先备份。这是不用动脑就该有的动作。

---

# 第一部分：location 匹配规则

## 先搞清楚：五种写法长什么样

| 写法 | 名称 | 例子 |
|------|------|------|
| `location /abc` | 前缀匹配 | `/api` `/api/v1` `/apixxx` 都能匹配 `/api` |
| `location = /abc` | **精确匹配** | 只有 `/abc`，多一个字符都不行 |
| `location ~ pattern` | 正则（区分大小写） | `~ \.jpg$` 匹配 .jpg 结尾 |
| `location ~* pattern` | 正则（不区分大小写） | `~* \.JPG$` 也匹配 .jpg |
| `location ^~ /abc` | **优先前缀** | 匹配后**不再检查正则** |

## 优先级：不是一条链，是一套流程

网上流传的「= > ^~ > ~ > 普通前缀」并不准确——`^~` 不是优先级更高，
它的真实作用是**命中后跳过正则**。nginx 实际按 4 步走：

```
① 先查 = 精确匹配，命中立即结束
② 在所有前缀 location（^~ 和普通前缀算同一类）里找「最长」的那个
③ 若最长前缀带 ^~ → 直接用它，不查正则
④ 否则按出现顺序查正则，第一个命中的赢；全不中 → 退回第②步那个前缀
```

## 实验 1：亲眼验证匹配流程

先记一个坑：`location ^~ /test` 和 `location /test` 是**同一个 location**
（`^~` 是修饰符，不是路径的一部分），写在一起 nginx 直接报
`duplicate location "/test"`——很多人第一次就栽在这。

```bash
sudo tee /etc/nginx/conf.d/lab.conf > /dev/null << 'EOF'
server {
    listen 8081;
    server_name _;

    location = /test        { return 200 "1.精确匹配 =\n"; }
    location ^~ /static     { return 200 "2.优先前缀 ^~\n"; }
    location ~ \.png$       { return 200 "3.正则 ~\n"; }
    location /test/abc      { return 200 "4.长前缀\n"; }
    location /test          { return 200 "5.短前缀\n"; }
}
EOF

sudo nginx -t && sudo systemctl reload nginx
```

> `return 200 "内容"` 是调试神器：不依赖任何真实文件直接返回文字。
> `> /dev/null` 是让 tee 别把内容往屏幕上回显一遍。

逐个验证（**先自己猜，再跑**）：

```bash
curl localhost:8081/test
curl localhost:8081/test/abc
curl localhost:8081/test/abc/a.png
curl localhost:8081/static/a.png
curl localhost:8081/testxyz
```

答案：

| 请求 | 命中 | 为什么 |
|------|------|--------|
| `/test` | **1.精确** | `=` 第一步就结束 |
| `/test/abc` | **4.长前缀** | 最长前缀是 4；正则 `\.png$` 不匹配 → 退回前缀 |
| `/test/abc/a.png` | **3.正则** | 前缀 4 不带 `^~` → 继续查正则 → `.png` 赢 |
| `/static/a.png` | **2.优先前缀** | 前缀 2 带 `^~` → **跳过正则**，这就是 `^~` 唯一的作用 |
| `/testxyz` | **5.短前缀** | 前缀 `/test` 照样匹配 testxyz |

> 重点看中间两行：同样是 png 请求，在 `/test/abc` 下被正则抢走，
> 在 `/static` 下被拦住。**`^~` 不是优先级高，而是有"拒绝正则"的能力。**

## 实验 2：正则的顺序陷阱

把两个正则调换顺序，结果会变：

```nginx
location ~ \.php$        { return 200 "A\n"; }
location ~ \.(php|jsp)$  { return 200 "B\n"; }
```

请求 `/a.php` → **A**（谁写在前面谁赢）。

这正是 YAML/配置类文件的共性：**正则按出现顺序匹配，第一个命中就停**。生产环境里规则写错顺序，会导致本该走 A 的请求被 B 抢走，且不报错。

---

# 第二部分：root vs alias（面试高频陷阱）

这两个指令长得像，行为完全不同。

## 先看 root

把下面加到 lab.conf：

```nginx
location /images/ {
    root /data;
}
```

请求 `/images/logo.png` → nginx 拼成 **`root + 完整 URI`**：

```
/data + /images/logo.png = /data/images/logo.png
```

## 再看 alias

```nginx
location /images/ {
    alias /data/;
}
```

请求 `/images/logo.png` → nginx **用 alias 替换掉 location 匹配的那段**：

```
/data/ + logo.png = /data/logo.png
```

## 一句话区别

```
root  = 路径前缀，URI 原样追加在后面
alias = 路径替换，location 那段被 alias 顶掉
```

## 动手验证（注意两个不同的 URL 最后指向同一个文件）

```bash
# 准备素材
sudo mkdir -p /data/images /data
echo "THIS-IS-ROOT"  | sudo tee /data/images/a.txt
echo "THIS-IS-ALIAS" | sudo tee /data/a.txt

cat | sudo tee /etc/nginx/conf.d/lab.conf << 'EOF'
server {
    listen 8081;
    server_name _;

    location /r/images/ { root /data; }
    location /a/images/ { alias /data/images/; }
}
EOF
sudo nginx -t && sudo systemctl reload nginx
```

```bash
curl localhost:8081/r/images/a.txt    # → THIS-IS-ROOT   (/data/images/a.txt)
curl localhost:8081/a/images/a.txt    # → THIS-IS-ALIAS  (/data/images/a.txt)
```

看明白了吗：两个 URL 不同，但**最终都落到同一个文件 `/data/images/a.txt`**——root 是拼接，alias 是替换。

## 那个必踩的坑：alias 末尾的斜杠

```nginx
location /a/ {
    alias /data;      # ← 少了斜杠
}
```

请求 `/a/b.txt` → nginx 拼成 `/datab.txt` → **404**。

**规则**：`location` 带斜杠时，`alias` 也必须以斜杠结尾；两边要么都有，要么都没有。

（顺带联想：`proxy_pass` 末尾斜杠的坑跟这个异曲同工，就在下一部分。）

---

# 第三部分：反向代理 proxy_pass（主角）

先确认一件事：**这台机器的 nginx 自带 proxy 模块**（它是 nginx 默认编译进去的，只有加 `--without-http_proxy_module` 才会排除，所以 `nginx -V` 里看不到 `--with-http_proxy` 很正常）。

## 实验 3：最简代理

目标：让访问 `http://192.168.52.139/tomcat/` 显示 Tomcat 的页面。

把 lab.conf 换成：

```bash
sudo tee /etc/nginx/conf.d/lab.conf << 'EOF'
server {
    listen 8081;
    server_name _;

    location /tomcat/ {
        proxy_pass http://127.0.0.1:8080/;
    }
}
EOF
sudo nginx -t && sudo systemctl reload nginx
curl localhost:8081/tomcat/
```

应该看到 Tomcat 欢迎页的 HTML。**这就是反向代理：用户以为在访问 nginx，实际内容来自 Tomcat。**

## ⚠️ 那个决定成败的斜杠

`proxy_pass` 末尾**有没有 URI**，行为天差地别：

| 写法 | 请求 `/tomcat/index.jsp` | 转发给 Tomcat 的 URL |
|------|--------------------------|----------------------|
| `proxy_pass http://127.0.0.1:8080;`（无斜杠） | 保留原 URI | `http://127.0.0.1:8080/tomcat/index.jsp` |
| `proxy_pass http://127.0.0.1:8080/;`（有斜杠） | 替换 location 段 | `http://127.0.0.1:8080/index.jsp` |

**口诀**：

> **带 URI（哪怕只是一个斜杠）= 替换；不带 = 拼上去。**

这就是最常见的「代理配好了却是 404」的原因——Tomcat 收到了它根本没有的 `/tomcat/index.jsp` 路径。

亲自验证：

```bash
# 先试不带斜杠的版本
sudo sed -i 's|proxy_pass http://127.0.0.1:8080/;|proxy_pass http://127.0.0.1:8080;|' /etc/nginx/conf.d/lab.conf
sudo nginx -t && sudo systemctl reload nginx
curl -I localhost:8081/tomcat/       # 观察状态码是不是 404
```

## 实验 4：加上转发头（生产必配）

Tomcat 现在看到的来访 IP 是 `127.0.0.1`，等于**丢失了真实客户端 IP**——排查问题、做访问统计、封 IP 全都失效。

完整写法：

```nginx
location /tomcat/ {
    proxy_pass http://127.0.0.1:8080/;

    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_redirect http://127.0.0.1:8080/ /;
    proxy_connect_timeout 5s;
    proxy_read_timeout    60s;
}
```

| 指令 | 作用 |
|------|------|
| `Host $host` | 把用户请求的域名传过去，不然后端生成链接时会用自己的 IP |
| `X-Real-IP` | 真实客户端 IP |
| `X-Forwarded-For` | 经过的每一跳 IP 追加成链（`$proxy_add_x_forwarded_for` 是"已有值 + 当前"） |
| `X-Forwarded-Proto` | 用户用的是 http 还是 https，后端生成 URL 时需要 |
| `proxy_redirect` | **改掉后端重定向里的内网地址**，否则浏览器会被导向 `127.0.0.1:8080` |
| `proxy_connect_timeout` | 连不上后端最多等多久，超时返回 502/504 |
| `proxy_read_timeout` | 后端处理太久就放弃（防慢请求堆积） |

验证头部有没有传过去——在 Tomcat 端看访问日志，IP 应该变成 `192.168.52.1`（网关）：

```bash
# 先在 Windows 浏览器里访问一次 http://192.168.52.139:8081/tomcat/
tail -3 /opt/tomcat9/logs/localhost_access_log.$(date +%Y-%m-%d).txt
```

## 实验 5：动静分离（生产的真实形态）

打游戏以外的正经配置长这样：静态文件 nginx 自己处理（快），动态请求甩给 Tomcat（慢）。

```nginx
server {
    listen 8081;
    server_name _;

    # 静态：nginx 直接从磁盘读，不惊动 Tomcat
    location /static/ {
        root /var/www;
        expires 30d;
    }

    # 动态：全丢给 Tomcat
    location / {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_redirect http://127.0.0.1:8080/ /;
    }
}
```

```bash
sudo mkdir -p /var/www/static
echo "STATIC-FROM-NGINX" | sudo tee /var/www/static/test.txt

curl localhost:8081/static/test.txt    # → STATIC-FROM-NGINX  （nginx 处理）
curl localhost:8081/                   # → Tomcat 页面        （Tomcat 处理）
```

**这就是绝大多数生产站点的结构**，理解它就理解了 nginx 的核心价值。

---

## 最终实战：正式上线到 80 端口

前面都在 8081 做实验，练熟了就把成果搬到主站点。编辑 `/etc/nginx/sites-available/default`：

```bash
# 删掉旧的 location，改成:
sudo sed -i 's|try_files \$uri \$uri/ =404;|proxy_pass http://127.0.0.1:8080/;|' /etc/nginx/sites-available/default
```

或者干脆用 vim 把 `location / { ... }` 整块改成：

```nginx
location / {
    proxy_pass http://127.0.0.1:8080/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_redirect http://127.0.0.1:8080/ /;
}
```

标准三步：

```bash
sudo nginx -t                      # 1. 验证语法
sudo systemctl reload nginx        # 2. 平滑重载（不断连接）
curl -I localhost                  # 3. 确认返回的是 Tomcat
```

然后浏览器访问 **`http://192.168.52.139`**（不带端口了）——应该直接看到汤姆猫，而不是 nginx 欢迎页。

**至此服务栈闭环**：用户只需要访问 80 端口，nginx 在背后把请求交给 Tomcat。

> 注意：原来的静态页还在 `/var/www/html/`，只是被代理规则挡住了。想留静态能力就用实验 5 的动静分离写法。

---

## 排障速查表

| 现象 | 原因 | 怎么查 |
|------|------|--------|
| **502 Bad Gateway** | Tomcat 没启动 / 端口不对 / 连不上 | `systemctl is-active tomcat`；`curl 127.0.0.1:8080` |
| **504 Gateway Timeout** | Tomcat 在跑但处理太慢 | 看 `proxy_read_timeout`；查 Tomcat 的 catalina.out |
| **代理后 404** | **proxy_pass 末尾斜杠写错** | 对照上面的替换/拼接表 |
| 页面出来了，样式全丢 | 后端生成的是绝对路径 | 加 `proxy_set_header Host $host` 或补 location 规则 |
| 点了链接跳到 127.0.0.1 | 后端重定向带内网地址 | 加 `proxy_redirect` |
| 后端日志里全是 127.0.0.1 | 没传 X-Forwarded-For | 补上三个 `proxy_set_header` |
| `nginx -t` 报 unknown directive | 配置文件写错 / 模块没编入 | 一般不会出现在 proxy 上（默认内置） |

**万能排查顺序**：

```bash
sudo nginx -t                                  # 语法
sudo tail -20 /var/log/nginx/error.log         # nginx 侧错误
sudo tail -20 /opt/tomcat9/logs/catalina.out   # Tomcat 侧错误
curl -I localhost                              # 从内到外验证
```

> 记住：**502 是 nginx 找不到后面的服务，504 是后面服务太慢**。这两句话在客户现场能省你十分钟。

---

## 收尾：删掉实验文件

练完之后把临时配置清掉，免得留坑：

```bash
sudo rm /etc/nginx/conf.d/lab.conf
sudo rm -rf /data /var/www/static
sudo nginx -t && sudo systemctl reload nginx
```

（如果还想恢复 nginx 原来的欢迎页，用开工前备份的 `default.bak` 覆盖回来。）

---

## 验收清单

- [ ] 能用 `curl` 说出五种 location 的命中结果
- [ ] 说得出 `root` 和 `alias` 的区别
- [ ] 知道 `alias` 末尾斜漏了会怎样
- [ ] 能讲清 `proxy_pass` 带/不带斜杠的差异
- [ ] 配出的代理能返回 Tomcat 页面
- [ ] 后端日志里能看到真实客户端 IP（不是 127.0.0.1）
- [ ] 知道 502 和 504 的区别
- [ ] 改配置一定会先 `nginx -t`
- [ ] 浏览器访问 `http://192.168.52.139` 直接看到 Tomcat

---

## 再往后

反向代理跑通之后，你的服务栈就有了完整故事：

```
用户 → Nginx :80 → proxy_pass → Tomcat :8080 → MySQL :3306
             ↘ 静态文件本地返回
```

接下来两个方向任选：

**① 深入 Nginx**：HTTPS（Let's Encrypt 证书）、负载均衡（upstream 多台 Tomcat）、缓存。负载均衡配上就是真正的"集群"雏形，简历上分量一下子不一样。

**② 转 Ansible**（你的突破口）：现在你有了三个要管理的服务，正好练"用 playbook 一键部署 Nginx + Tomcat + MySQL"。把手工敲过的命令变成代码，这是从"会操作"到"会运维"的分界线。
