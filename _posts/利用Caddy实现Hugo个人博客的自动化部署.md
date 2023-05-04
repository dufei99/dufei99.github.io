---
title: 利用Caddy实现Hugo个人博客的自动化部署
updated: 2023-05-04 15:54:11Z
created: 2023-05-04 13:43:23Z
source: https://blog.wangjunfeng.com/post/caddy-hugo/
tags:
  - caddy
  - hugo
---

# 利用Caddy实现Hugo个人博客的自动化部署

- [关于Caddy](#%e5%85%b3%e4%ba%8ecaddy)
- [安装Caddy](#%e5%ae%89%e8%a3%85caddy)
- [注册Caddy服务](#%e6%b3%a8%e5%86%8ccaddy%e6%9c%8d%e5%8a%a1)
- [配置 Github 仓库的WebHook](#%e9%85%8d%e7%bd%ae-github-%e4%bb%93%e5%ba%93%e7%9a%84webhook)
- [配置Caddy](#%e9%85%8d%e7%bd%aecaddy)
- [最后](#%e6%9c%80%e5%90%8e)

之前我的博客的自动化部署，是使用golang写了一个webserver，来接收coding或者github的webhook，然后拉取最新的博客数据，使用hugo生成静态页，来实现自动化。同时证书使用Let’s Encrypt，因为证书只有3个月有效期，所以使用了Acme.sh来实现证书的自动续期。最近看到了Caddy，这两个功能都能通过简单配置即可自动化，所以就使用Caddy来替换nginx和之前的自动化部署方案。

## 关于Caddy

![Caddy](../../_resources/caddy_4f5b2433830a491bb8357e0901451389.png)

Caddy是一个使用Golang编写的开源的Web服务器，支持HTTP/2，默认启用HTTPS，且是第一个无需额外配置即可提供HTTPS特性的Web服务器。同时也可以作为反向代理和负载均衡器。Caddy的大部分功能都实现为中间件，并通过Caddyfile配置文件中的指令来进行控制。支持以下特性：

- HTTP/1.1 (原始的HTTP) and HTTP/2 (HTTPS的推荐连接方案)
- HTTPS， 同时接受自动签发和手动管理
- 虚拟主机 (多个站点工作在单个端口上)
- 原生 IPv4 和 IPv6 支持
- 静态文件分发
- 平滑重启/重载
- 反向代理 ( HTTP 或 WebSockets)
- 负载均衡和健康性检查
- FastCGI 支持
- 配置文件模板
- Markdown 渲染
- CGI 通过 WebSockets
- Gzip 压缩
- 简单服务器鉴权
- URL 重写
- 重定向
- 文件浏览服务
- 访问日志
- 实验性 QUIC 支持

Caddy官方网站：https://caddyserver.com/

Github: https://github.com/mholt/caddy

## 安装Caddy

首先是下载Caddy： https://caddyserver.com/download

左侧需要选择自己的环境、插件、是否开启获取你的caddy运行状态，以及选择个人还是商业证书。

由于本文主要讲如何实现Hugo的自动化部署，所以这里就只选择相关的插件：

- http.git
- http.hugo

比如我这里选择后下方显示两种下载方式：

直接下载地址： https://caddyserver.com/download/linux/amd64?plugins=http.git,http.hugo&license=personal&telemetry=off

一键安装脚本：

```
curl https://getcaddy.com | bash -s personal http.git,http.hugo 
```

当然为了方便肯定选择一键安装脚本，Caddy二进制文件会被自动放在`/usr/local/bin/caddy`下面。

至此就算安装完毕了，但是它目前还不会开机自启，接着我们还需要把Caddy注册为服务。

## 注册Caddy服务

这里我们可以使用官方提供的脚本 [caddy.service](https://github.com/mholt/caddy/blob/master/dist/init/linux-systemd/caddy.service) ，其他系统也可以在 [这里](https://github.com/mholt/caddy/tree/master/dist/init) 找到相应的脚本。

为了安全，我们不能使用root直接运行caddy，所以我们假设

- 我们使用用户`www-data`和用户组`www-data`来运行Caddy，UID和GID都是33
- 使用一个非root账户来运行，并且这个账户可以使用sudo来以root运行命令

首先，我们需要给caddy设置适当的权限：

```
sudo chown root:root /usr/local/bin/caddy
sudo chmod 755 /usr/local/bin/caddy 
```

给Caddy二进制文件以非root账户身份绑定到80和443端口的能力：

```
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/caddy 
```

这只需要的用户、用户组和目录：

```
sudo groupadd -g 33 www-data
sudo useradd \
	-g www-data --no-user-group \
	--home-dir /var/www --no-create-home \
	--shell /usr/sbin/nologin \
	--system --uid 33 www-data  
sudo mkdir /etc/caddy
sudo chown -R root:root /etc/caddy
sudo mkdir /etc/ssl/caddy
sudo chown -R root:www-data /etc/ssl/caddy sudo chmod 0770 /etc/ssl/caddy 
```

将您的Caddy配置文件(比如`Caddyfile`)放在适当的目录中，并设置适当的权限：

```
sudo cp /path/to/Caddyfile /etc/caddy/ sudo chown root:root /etc/caddy/Caddyfile sudo chmod 644 /etc/caddy/Caddyfile 
```

创建用户的主目录，并赋予适当的权限：

```
sudo mkdir /var/www
sudo chown www-data:www-data /var/www
sudo chmod 555 /var/www 
```

安装systemd服务配置文件，并重新加载systemd守护进程，然后启动caddy。

```
wget https://raw.githubusercontent.com/mholt/caddy/master/dist/init/linux-systemd/caddy.service sudo cp caddy.service /etc/systemd/system/
sudo chown root:root /etc/systemd/system/caddy.service
sudo chmod 644 /etc/systemd/system/caddy.service
sudo systemctl daemon-reload 
```

然后还需要让我们的caddy随系统自动启动：

```
sudo systemctl enable caddy.service 
```

## 配置 Github 仓库的WebHook

假设您的博客域名为 `blog.wangjunfeng.com`。

打开github仓库，点击Setting，选择Webhooks，点击Add webhook。

<img width="720" height="342" src="../../_resources/github-webhook_2075e35c4ce6407b9a04bd5c5d034feb.jpg"/>

- Payload URL设置为您的博客根目录加`/webhook`，例如这里就是`https://blog.wangjunfeng.com/webhook`
- Content type 设置为 `application/json`
- Secret这个自己随便设置，比如`NtaZj251UH8xqB9WrRYo3Xbpvwnm6Jhs`
- 事件可以只选择push event
- 点击Add webhook按钮即可。

## 配置Caddy

创建我们的博客目录`/var/www/blog.wangjunfeng.com`。

创建日志目录`/var/www/logs`。

然后编辑我们的caddy的配置文件`/etc/caddy/Caddyfile`如下：

```
blog.wangjunfeng.com {
	tls admin@wangjunfeng.com
	gzip
	root /var/www/blog.wangjunfeng.com/public
	log /var/www/logs/blog.wangjunfeng.com.access.log
	git {
		repo git@github.com:JefferyWang/blog.git
		branch master
		path /var/www/blog.wangjunfeng.com
		then hugo --destination=/var/www/blog.wangjunfeng.com/public
		hook /webhook NtaZj251UH8xqB9WrRYo3Xbpvwnm6Jhs
		hook_type github
	}
	errors /var/www/logs/blog.wangjunfeng.com.error.log {
		404 404.html 
	}
} 
```

- `tls`后面的邮箱是我们申请证书的邮箱，填写自己的邮箱即可
- `gzip`指开启gzip压缩
- `root`是我们站点的目录
- `log`用来记录我们博客的访问日志
- `repo`用来配置我们的git仓库
- `branch`用来配置我们要拉取的分支
- `path`是指我们的拉取代码的目标地址
- `then`用来配置拉取代码后，我们重新编译hugo的静态文件
- `hook`是我们对github开放的webhook地址和密钥
- `hook_type`是我们的webhook类型

当然上面是针对仓库是public开放状态，如果你的仓库是private状态，那么还需要配置ssh公钥，密钥。具体如何生成以及github如何添加ssh公钥，这个需要你自行搜索配置。这里只说如何在caddy内配置ssh免密拉取代码，我们只需要在git下增加下面一行即可：

```
key /path/to/.ssh/id_rsa 
```

然后启动我们的服务就可以了：

```
sudo systemctl start caddy.service 
```

## 最后

虽然Caddy给我们带来了很大便利，但是由于是Golang编写，可能跟nginx相比，性能会有差异。对于我们博客这种小的网站，这些性能差异是无所谓的。但是如果是实际工作中的业务，建议还是使用nginx这种方案更好。