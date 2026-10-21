# django部署到云服务器：从裸机到 HTTPS 上线的完整流程，含生产 settings 和踩坑清单

Django 部署到云服务器，本质上是四件事：把代码放到一台 Linux 机器上，用 Gunicorn 把 Django 跑起来，用 Nginx 挡在前面处理 HTTPS 和静态文件，再用 systemd 保证进程挂了能自己爬起来。

`manage.py runserver` 不在这个列表里。它是单线程的，改个文件就自动重启，Django 官方文档写得很明白：这玩意没经过安全审计，是开发工具，不是服务器。

下面按「买机器 → 配环境 → 改 settings → 起服务 → 上 HTTPS → 上线检查」的顺序走。中间会重点标出几个几乎一定会踩的坑，其中有些坑的报错信息，一个字都不会提到真正的原因。

> **一句话选型**：个人项目、博客、刚上手，腾讯云轻量应用服务器 2 核 2G 起步就够；正式业务、需要随时升降配，再看 CVM 云服务器。数据库能上云数据库就别跟应用挤一台机器。云服务器价格几乎每周都跟着活动变，本文不写死数字，你点进去看到的才是当天价。
>
> 👉 查看腾讯云当前云服务器活动价

## 先搞清楚：Django 站点到底需要什么样的服务器

一台 1–2 核、2GB 内存的 Linux 实例，就足够跑一个中小型 Django 站点。真正吃资源的是数据库和并发，不是 Django 本身。

内存是最先撞墙的地方。Gunicorn 每开一个 worker，就是一份独立的 Django 进程，跑起来 150–200MB 是常态。2 个 worker 加上 Nginx 和系统本身，2GB 内存是舒服的起点；1GB 也能跑，但要开 swap，而且一旦碰上慢查询或者大文件导出，很容易被 OOM killer 干掉，日志里只留一行 `Killed`。

带宽方面，静态文件交给 Nginx 直接读磁盘，不消耗 Python 进程，但 API 响应还是要走带宽。国内云厂商入门套餐一般给 3–5M 固定带宽，小站够用。

数据库这块，Django 官方文档的说法是生产环境推荐 PostgreSQL，同时也正式支持 MariaDB、MySQL、SQLite 和 Oracle。SQLite 的问题在于写操作会锁住整个库，单用户玩具项目无所谓，一旦有并发写入就会开始看到 `database is locked`。

用户上传的文件不要存本地磁盘。实例重建、换机器、或者以后想横向扩到两台，本地文件就是第一个出问题的地方。

腾讯云这边跟 Django 部署直接相关的产品大概是这样：

| 产品 | 适合什么场景 | 大致规格 | 计费方式 | 当前活动 |
| --- | --- | --- | --- | --- |
| 轻量应用服务器 | 个人项目、博客、演示站 | 固定档位，CPU/内存/带宽打包，含月流量包 | 包年包月 | [ 查看轻量服务器活动价](https://bit.ly/yunhuodong) |
| CVM 云服务器 | 正式业务，需要弹性升降配 | CPU、内存、带宽、系统盘分开选 | 包年包月 / 按量计费 | [ 查看 CVM 活动价](https://bit.ly/yunhuodong) |
| 云数据库 PostgreSQL | 不想自己装、自己备份 | 单节点 / 高可用双节点 | 包年包月 | [ 查看云数据库活动价](https://bit.ly/yunhuodong) |
| 对象存储 COS | 用户上传的图片、附件 | 按存储量 + 请求数 + 流量 | 按量计费 | [ 查看 COS 计费说明](https://bit.ly/yunhuodong) |
| 负载均衡 CLB | 多台后端、要做零停机 | 按实例 + 带宽 | 包年包月 / 按量 | [ 查看 CLB 计费](https://bit.ly/yunhuodong) |

轻量应用服务器和 CVM 的区别不在「能不能跑 Django」——两者都能装 Ubuntu 22.04/24.04、装 Python 3.12、装 Nginx，跑起来没有任何区别。区别在计费和弹性：轻量是把 CPU、内存、带宽、流量打包成固定档位卖，便宜、控制台简单，代价是不能单独调某一项；CVM 的 CPU、内存、带宽、系统盘是分开配置的，随时能升配，能挂多块云盘，适合业务会长大的场景。

个人项目选轻量，省下的钱够买两年域名。要是你已经在盘算「以后可能要加机器」，那就别从轻量起步，迁移那天的麻烦比省下的钱贵。

👉 [看看腾讯云当前有哪些服务器在做活动](https://bit.ly/yunhuodong)

## 服务器到手后的第一小时

先建一个只能跑代码、不能登录的系统账户，再装 Python 环境和 Nginx。全程用 root 操作，是后面一堆说不清来源的权限问题的源头。

bash
sudo apt update
sudo apt install -y python3-venv git nginx
sudo adduser --system --group --home /srv/myproject django
sudo -u django git clone <你的仓库地址> /srv/myproject/app
ls -ld /srv/myproject


最后那行 `ls -ld` 的输出，认真看一眼。

Ubuntu 的 `/etc/adduser.conf` 里默认 `DIR_MODE=0750`，意思是 `/srv/myproject` 这个目录，除了属主和同组，其他账户进不去。而 Nginx 的 worker 进程跑在 `www-data` 账户下——它读不到你放在里面的静态文件。

这个坑的恶心之处在于，它不会报「权限不足」，只会在浏览器里显示静态文件 404。然后你去翻 Django 的日志，翻半天，什么也找不到。

安全组（防火墙）只开三个口：22、80、443。22 最好限制成你自己的 IP。数据库端口 5432 或 3306 永远不要对 `0.0.0.0/0` 开放。如果数据库和应用在同一台机器上，直接让它绑 `127.0.0.1`，连安全组规则都省了。

Python 环境用 venv 还是 uv 都行。uv 快，`uv sync --frozen` 装出来的环境可复现；venv 是标准库自带，不用额外装东西。有一件事别做：`sudo pip install`。Ubuntu 24.04 会直接拒绝，报 `externally-managed-environment`，因为系统 Python 归 apt 管，apt 升级时会把你的包冲掉，而且不告诉你。

Gunicorn 写进 `requirements.txt`，跟 Django 放一起，别事后手动 `pip install`。不然换台机器重建环境，Gunicorn 就漏了，而且要到 systemd 启动失败那一刻才会发现。

## 生产环境的 settings，这几项不改一定出问题

`DEBUG` 关掉、`SECRET_KEY` 从环境变量读、`ALLOWED_HOSTS` 和 `CSRF_TRUSTED_ORIGINS` 填上域名。这三件事不做，站点不是报 400 就是带着安全隐患上线。

python
import os

DEBUG = os.environ.get("DJANGO_DEBUG", "0") == "1"
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
ALLOWED_HOSTS = [h for h in os.environ.get("DJANGO_ALLOWED_HOSTS", "").split(",") if h]
CSRF_TRUSTED_ORIGINS = ["https://" + h for h in ALLOWED_HOSTS]

SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")

STATIC_URL = "/static/"
STATIC_ROOT = "/srv/myproject/static"
STORAGES = {
    "default": {"BACKEND": "django.core.files.storage.FileSystemStorage"},
    "staticfiles": {"BACKEND": "django.contrib.staticfiles.storage.ManifestStaticFilesStorage"},
}


`SECRET_KEY` 用 `os.environ[...]` 而不是 `os.environ.get(...)`，是故意的。缺了这个变量，进程在 import 阶段就抛 `KeyError`，起不来——你能立刻发现。要是用了 `.get()` 配一个默认值，进程会带着假 key 正常启动，等哪天默认值一变，所有 session cookie 一起失效，线上用户全部被登出，而且你根本不会把这两件事联系起来。

`CSRF_TRUSTED_ORIGINS` 必须带 `https://`，从 Django 4.0 起就是这样。症状很好认：HTTP 下表单提交正常，一换成 HTTPS 就返回 CSRF 失败页。

`STORAGES` 是 Django 4.2 引入的，取代了 `STATICFILES_STORAGE`，后者在 5.1 已经被移除。这里有个细节很多人栽过：`STORAGES` 是整体替换默认字典，不是往里合并。只写 `staticfiles` 不写 `default`，文件上传的存储后端就没了，`FileField` 会在运行时报错，而 `check` 命令未必拦得住。

`SECURE_PROXY_SSL_HEADER` 是在告诉 Django：Nginx 已经处理过 TLS 了，原始请求是 https。不设这个，`request.is_secure()` 永远是 False，Django 生成的重定向会把你从 https 打回 http，然后浏览器再报混合内容。

环境变量放 `/etc/myproject/env`，权限 640，属主 root。systemd 是以 root 身份读这个文件的（在切到 django 账户之前），所以密钥不需要让 web 进程可读。

## Gunicorn 和 systemd：让 Django 变成「服务」

Gunicorn 负责跑 Django，systemd 负责它崩了自动重启、开机自动启动。两个配好，你的站点才不依赖于「那个 SSH 窗口一直开着」。

Gunicorn 绑 Unix socket，不要绑 TCP 端口：

bash
/srv/myproject/venv/bin/gunicorn myproject.wsgi:application \
  --bind unix:/run/myproject/gunicorn.sock \
  --workers 3 \
  --timeout 60


socket 文件没有端口，本机之外访问不到，Nginx 通过文件路径连它，等于少暴露一个端口。

worker 数量有个常见起点：`2 × CPU 核数 + 1`。2 核机器就是 5 个 worker，但每个吃 150MB 以上内存，2G 的机器扛不住。实际按内存倒推更靠谱，2G 内存开 2–3 个。

还有个容易忽略的问题：Gunicorn 默认的同步 worker 会把整个请求读完才交给视图。一个每秒发一个字节的客户端，就能一直占着这个 worker。Gunicorn 自己的文档说得很直接：用默认同步 worker 时，必须靠前面的代理缓冲慢请求，否则就是一个现成的拒绝服务入口。Nginx 的 `proxy_buffering` 默认是开的，能挡掉大半；但如果你前面没有 Nginx、Gunicorn 直连公网，这事就得单独处理。

systemd unit 里几个关键项：

ini
[Service]
User=django
Group=django
WorkingDirectory=/srv/myproject/app
EnvironmentFile=/etc/myproject/env
ExecStart=/srv/myproject/venv/bin/gunicorn myproject.wsgi:application --bind unix:/run/myproject/gunicorn.sock --workers 3
Restart=always
RuntimeDirectory=myproject


`RuntimeDirectory=myproject` 这一行别省。它让 systemd 自动创建 `/run/myproject` 并把属主设成 `django`，Gunicorn 才写得进 socket 文件。手动 `mkdir` 的目录，重启机器后就没了，然后你会看到一个毫无头绪的 502。

改完配置：`sudo systemctl daemon-reload && sudo systemctl restart myproject`，看日志用 `journalctl -u myproject -f`。

## Nginx：TLS、静态文件和慢请求缓冲

Nginx 干四件 Gunicorn 干不了的事：终止 TLS、缓冲慢请求、直接从磁盘读静态文件、独占 80 和 443 端口。

静态文件那段配置大概长这样：

nginx
location /static/ {
    alias /srv/myproject/static/;
    expires 30d;
}


请求 `/static/` 下的东西时，Nginx 直接读磁盘返回，完全不碰 Python 进程。这也是为什么 `collectstatic` 必须跑、`STATIC_ROOT` 必须指对——文件得真的在那。

反代那段：

nginx
location / {
    proxy_pass http://unix:/run/myproject/gunicorn.sock;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    client_max_body_size 20m;
}


`X-Forwarded-Proto` 和 Django 的 `SECURE_PROXY_SSL_HEADER` 是一对，少一边就不生效。

`client_max_body_size` 默认只有 1MB。用户传一张手机拍的照片就超过了，然后拿到一个 413，错误信息跟 Django 一点关系都没有。做图片上传的项目，这行必加。

HTTPS 用 certbot，一条命令签发证书 + 自动续期，它还会帮你改写 Nginx 配置。装完之后用 `sudo certbot renew --dry-run` 验证一下续期流程能不能走通，别等三个月后证书过期才想起来。

## 数据库、迁移和文件存储

生产环境别用 SQLite，用云数据库，并把访问来源限制成应用服务器的安全组，而不是某个固定 IP。

迁移在服务器上跑：

bash
python manage.py migrate


这里有个比代码回滚严重得多的问题：代码可以回滚，已经执行的迁移回不去。一个 `DROP COLUMN` 发出去，旧版本代码就算切回来，字段也没了。

稳妥的做法是把破坏性迁移拆成两次发布。第一次只加新字段、写双份数据，代码兼容新旧两种结构；确认线上稳定之后，第二次发布再删旧字段。麻烦，但比凌晨三点恢复数据库强。

云数据库的安全组规则，来源填「应用服务器的安全组」而不是具体 IP。这样服务器换 IP、加机器，规则都不用动；同时 PostgreSQL 的 5432 端口从头到尾没有对公网开放过。

用户上传的文件放对象存储 COS，本地磁盘只留代码和静态文件。多台后端的时候，本地文件方案会第一个崩掉，而且崩得很安静——只有用户告诉你图片打不开。

## 上线前照着过一遍

1. `DEBUG = False`
2. `ALLOWED_HOSTS` 里有你的域名
3. `CSRF_TRUSTED_ORIGINS` 带 `https://`
4. `python manage.py check --deploy` 没有 warning
5. 静态文件在浏览器里能直接打开，状态码 200
6. `kill` 掉 Gunicorn 进程，5 秒内能自己起来
7. 重启整台服务器，站点能自动恢复
8. 数据库端口没有对公网开放
9. 有备份，并且真的试过一次恢复
10. `journalctl -u myproject` 能看到日志和报错

第 4 条那个命令值得单独说。`check --deploy` 会把所有生产环境相关的配置项过一遍，包括 `SECURE_HSTS_SECONDS`、`SESSION_COOKIE_SECURE`、`CSRF_COOKIE_SECURE` 这些容易被忘掉的项。它不检查业务逻辑，但配置层面的疏漏基本都能扫出来。跑出来一堆 warning 不丢人，上线之后被扫出来才丢人。

## 上线当天最可能遇到的几个问题

**502 Bad Gateway**——Nginx 连不上 Gunicorn。先 `systemctl status myproject`，看进程在不在；进程在的话，八成是 socket 路径或权限问题。`/run/myproject` 目录属主不对是最常见的。

**静态文件 404，Nginx 日志里写 Permission denied**——就是前面提过的 `/srv/myproject` 目录 0750。改成 755，或者把 `STATIC_ROOT` 挪到别的路径。

**CSRF verification failed**——`CSRF_TRUSTED_ORIGINS` 没带 scheme，或者 Nginx 的 `X-Forwarded-Proto` 没传。

**上传文件返回 413**——`client_max_body_size` 没改。

**Gunicorn 明明在跑，还是 502**——worker 超时。默认 `--timeout 30`，导一个报表、跑一个慢查询就超了。调到 60 或 120，同时想想是不是该把重活丢给异步任务队列。

**时间和日志对不上**——`TIME_ZONE` 设成 `Asia/Shanghai`，`USE_TZ = True`，数据库那边也确认一下时区。

## 怎么选，看你属于哪种情况

只是学习、交课程作业：一台最低配的轻量应用服务器就够，甚至可以先在本地虚拟机上把整套流程跑一遍再上云。重点是走通 Gunicorn + Nginx + systemd 这条链路，配置错几次比看十篇教程管用。

个人项目、博客、内部小工具：轻量应用服务器 2 核 2G，数据库先跟应用挤一台，但务必配快照和自动备份。这个阶段单独买云数据库属于花钱买省心，值不值看你自己的时间成本。

正式业务：CVM + 云数据库 + 对象存储，应用和数据库分机器，至少两台后端加负载均衡，才谈得上零停机。单机也能做蓝绿部署——两份代码、两个端口、Nginx 切换 upstream——但那解决的是发布瞬间不中断，解决不了机器本身挂掉。

下单前确认三件事：带宽是固定还是按流量计费（按流量的超出部分不便宜）、续费价和首单价差多少、快照和备份有没有额外额度。云服务器的活动价变得比天气还快，本文里的规格描述只保证方向没错，具体数字以你打开页面时看到的为准。

👉 [现在去腾讯云看看当前的服务器和云数据库报价](https://bit.ly/yunhuodong)
