---
layout: post
title:  "创建 jekyll Docker 镜像"
date:   2025-07-24 22:03:00 +0800
categories: [Toolbox]
tags: [jekyll Docker]
excerpt: 这篇文章描述了如何基于 ubuntu 24.04 镜像创建带 nginx 的 jekyll 镜像。
---

## 1. 安装 ubuntu 24.04 Docker 镜像

将当前用户添加到 docker 组后，可以不使用 sudo 来操作 docker（加入后不行的话重启系统）：

```
sudo usermod -aG docker $USER
```

使用合适的 Docker 镜像源拉取 ubuntu 24.04 的基础镜像：

```
sudo docker pull docker.1ms.run/ubuntu:24.04
```

## 2. 基于 ubuntu 24.04 镜像创建新镜像

新建 Dockerfile 目录，创建 Dockerfile 文件：

```
~$ mkdir Dockerfile
~$ cd Dockerfile
~$ vim Dockerfile
```

输入以下内容：

```
FROM docker.1ms.run/ubuntu:24.04

# 需要挂载的目录（这里也可以不用指定）
VOLUME ["/home", "/etc/nginx/conf.d", "/var/www/html"]

# 安装必要软件
RUN apt-get update && apt-get install -y sudo vim nginx ruby-full git build-essential zlib1g-dev openssh-server
# ubuntu 24.04 默认有 ubuntu 用户，无需创建
# RUN useradd -m -s /bin/bash ubuntu
# RUN usermod -a -G sudo ubuntu

# 清除缓存
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# 为 sshd 创建特权分离目录
RUN mkdir /var/run/sshd

# 将 ubuntu 用户的密码修改为 password
RUN echo 'ubuntu:password' | chpasswd

# 这里不安装 jekyll，因为安装了没有用，构建博客时，还需要安装博客依赖的 gems，要安装到挂载的目录中，避免容器重启后丢失。

EXPOSE 80
EXPOSE 22
# 让 nginx 在前台运行，不加这一句，容器启动后 nginx 不会自动启动
ENTRYPOINT ["nginx", "-g", "daemon off;"]
```

编译镜像：

```
~/Dockerfile$ docker build -t ubuntu_jekyll:24.04 .
```

查看镜像

```
~/Dockerfile$ docker images
REPOSITORY              TAG       IMAGE ID       CREATED          SIZE
ubuntu_jekyll           24.04     52413e910f8e   23 minutes ago   657MB
docker.1ms.run/ubuntu   24.04     65ae7a6f3544   9 days ago       78.1MB
ubuntu                  latest    65ae7a6f3544   9 days ago       78.1MB
docker.1ms.run/ubuntu   20.04     b7bab04fd9aa   3 months ago     72.8MB
docker.1ms.run/ubuntu   20.10     e508bd6d694e   3 years ago      79.4M
```

导出带标检的包：

```
~/Dockerfile$ docker save -o ubuntu_jekyll.tar ubuntu_jekyll:24.04
```

查看导出的包：

```
~/Dockerfile$ ls
Dockerfile ubuntu_jekyll.tar
```

## 3. 使用镜像

将容器的如下路径映射到宿主机合适的位置上：
- /etc/nginx/conf.d
- /home

将 80 端口映射到合适的端口上。

测试新的镜像：

```
~/Dockerfile$ docker run --name ubuntu_jekyll -p 80:80 -p 2222:22 -itd ubuntu_jekyll:24.04
```

或：

```
~/Dockerfile$ docker run --name ubuntu_jekyll -p 80:80 -p 2222:22 -itd -v .:/home -v .:/etc/nginx/conf.d ubuntu_jekyll:24.04
```

查看：

```
~/Dockerfile$ docker ps -a
CONTAINER ID   IMAGE                 COMMAND                  CREATED          STATUS          PORTS                                                                      NAMES
6a4beee9cd39   ubuntu_jekyll:24.04   "nginx -g 'daemon of…"   26 minutes ago   Up 26 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:2222->22/tcp, :::2222->22/tcp   ubuntu_jekyll
```

使用 curl 测试，一切顺利的话，容器会返回 nginx 的欢迎信息：

```
~/Dockerfile$ curl localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

进入容器：

```
~/Dockerfile$ docker exec -it ubuntu_jekyll /bin/bash
```

查看 nginx 的状态，可以看到，nginx 已经在运行了：

```
root@6a4beee9cd39:/# service nginx status
 * nginx is running
```

本例中，sshd 不会自动启动，使用时需要进入容器，手动启动 sshd：

````
root@6a4beee9cd39:/# /usr/sbin/sshd
````

启动后就可以通过 ssh 登录容器了，例如：

```
ssh ubuntu@localhost -p 2222
```

也可以通过 SFTP 访问。

## 4. 安装 jekyll

在容器中切换到 ubuntu 用户，将 gems 安装到该用户的 home 目录下，由于 /home 目录是挂载到容器上去的，因此安装完毕后，即便容器重启也不会丢失。

指定 gems 目录在 ubuntu 用户的 home 目录下：

```
ubuntu@6a4beee9cd39:~$ echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
ubuntu@6a4beee9cd39:~$ echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
ubuntu@6a4beee9cd39:~$ echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
ubuntu@6a4beee9cd39:~$ source ~/.bashrc
```

进入博客目录，安装需要的 gems：

```
ubuntu@6a4beee9cd39:~/blog$ bundle install
```

之后就可以构建博客了。


## 说明

为了让 nginx 能够在容器启动时运行，需要通过 Dockerfile 传输 "nginx -g daemon off;" 命令，即让 nginx 在前台运行，也就是说需要通过 Dockerfile 编译镜像。通过运行容器、在容器中安装并启动 nginx（包括 systemctl enable nginx），然后 commit 的方式创建出来的新镜像，创建容器后 nginx 不会随容器的启动运行。
