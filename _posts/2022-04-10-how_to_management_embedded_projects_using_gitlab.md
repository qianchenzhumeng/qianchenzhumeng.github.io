---
layout: post
title:  "如何使用 Gitlab 进行嵌入式项目管理"
date:   2022-04-10 15:04:00 +0800
categories: [Embedded, Other]
tags: [Other]
excerpt: 本文以在 wsl2 上安装 Gitlab 为例，介绍如何自建系统，利用 Gitlab 进行嵌入式项目管理，包括源代码版本控制、制品管理以及故障追踪。
---

## 1. 背景

对于体量比较小的嵌入式项目来说，如果打算自建项目管理系统，那么 Gitlab 是个不错的选择，像源代码版本控制、制品管理、故障跟踪，甚至需求管理，都可以在 Gitlab 完成。

## 2. Gitlab 安装及启动

安装过程相对简单，以 Ubuntu 为例，安装社区版：

```bash
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo apt-get install gitlab-ce
```

这就安装好了。当然，如果是在服务器上安装，那还要进行更多的配置，按照官网上的操作即可，此处不再赘述。

首次运行前，先进行配置：

```bash
sudo gitlab-ctl reconfigure
```

默认的管理员账号及密码相关的信息会显示在屏幕上，例如：

```
Notes:
Default admin account has been configured with following details:
Username: root
Password: You didn't opt-in to print initial root password to STDOUT.
Password stored to /etc/gitlab/initial_root_password. This file will be cleaned up in first reconfigure run after 24 hours.
```

启动：

```bash
sudo gitlab-ctl start
```

然后检查一下运行状态：

```bash
sudo gitlab-ctl status
```

没有问题的话，就可以使用本地地址 `127.0.0.1` 进行访问了。使用管理员账号登录，添加一个账户，例如 `qianchenzhumeng`，添加好后记得为该账户修改密码。

## 3. 项目管理

### (1) 源代码版本控制

使用刚才创建的账户登录 Gitlab，新建代码仓库，例如 `arduino_test`，并将测试用的代码推送到该仓库中。

### (2) 制品管理

这里说的制品指的是编译出的二进制文件。制品管理，所面临的最主要的问题是，如何维护制品和源代码的对应关系，一般需要将每个版本的制品以及源代码进行统一归档。可以使用 Gitlab 的发布系统来完成该工作。

Gitlab 有 CI/CD，进行相关配置后，可以自动进行构建、发布。对于一些嵌入式项目来说，可能开发是在另外的系统上完成的，比如 Windows，要放在 Linux 上构建要么不太可能，要么得费一番周折，得不偿失，因此，自动构建、发布可能无法使用。不过这并不妨碍我们达成制品管理的目的，我们可以采取手动上传二进制文件，发布版本时，将其作为资源添加进去。

发布是基于 Git 标签进行的，可以通过 UI 完成，但是，上传文件没有 UI 的支持，需要调用 API 完成。下面使用具体的示例来介绍完整的发布过程。

假定要发布版本 `v0.0.1`，那么创建对应的标签（假定标签名就是 `v0.0.1`），并将标签推送到 Gitlab 上。检出对应的源代码，编译出要发布的二进制文件（比如，`Blink_v_0.0.1.hex`）。

```bash
git tag v0.0.1
git push origin v0.0.1
```

上传文件的 API 是这样的：

```
POST /projects/:id/uploads
```

其中 id 是指项目 id，进入项目页面后可以看到。

curl 示例如下：

```bash
curl --request POST --header "PRIVATE-TOKEN: <your_access_token>" --form "file=@dk.png" "https://gitlab.example.com/api/v4/projects/5/uploads"
```

如实例命令所示，需要一个私有令牌。进入项目页面，点击左侧菜单中的 `设置` -> `访问令牌` 可以为项目创建访问令牌，创建好后需要记下来，后续无法查看，忘记的话只能重新创建。

有了令牌（例如：`hymhsbyqCjZF_6MyyhX1`）就可以上传文件了：

```bash
curl --request POST --header "PRIVATE-TOKEN: hymhsbyqCjZF_6MyyhX1" --form "file=@Blink_v_0.0.1.hex" "http://127.0.0.1/api/v4/projects/2/uploads"
```

返回信息：

```json
{
  "alt":"Blink_v_0.0.1.hex",
  "url":"/uploads/a52ab44fad015bb96df8085f6e9cf1a4/Blink_v_0.0.1.hex",
  "full_path":"/qianchenzhumeng/arduino_test/uploads/a52ab44fad015bb96df8085f6e9cf1a4/Blink_v_0.0.1.hex",
  "markdown":"[Blink_v_0.0.1.hex](/uploads/a52ab44fad015bb96df8085f6e9cf1a4/Blink_v_0.0.1.hex)"
}
```

返回信息中的 `url` 是文件相对于项目路径的 url，因此，假定项目地址是 `http://127.0.0.1/qianchenzhumeng/arduino_test`，使用如下地址即可访问刚上传的文件：

```
http://127.0.0.1/qianchenzhumeng/arduino_test/uploads/a52ab44fad015bb96df8085f6e9cf1a4/Blink_v_0.0.1.hex
```

接下来就可以发布了。在左侧菜单中选择 `部署` -> `发布`，然后点击页面内的 `新建发布` 按钮，选择标签，输入发布说明，在 `链接` 中添加刚才上传的文件的链接，然后点击创建发布。发布成功后，就可以在 `部署` -> `发布` 页面看到了。

![release](/assets/img/2022-04-10-how_to_management_embedded_projects_using_gitlab.assets/release.png)

### (3) 需求管理&故障跟踪

左侧菜单中有 `议题`，下面有 `列表`、`看板` 等，可以利用这个来进行需求管理以及故障跟踪。

## 4. 其他

如果局域网内其他机器要访问安装在 wsl2 中的 Gitlab，还需要完成如下三件事：

- 修改 Gitlab 的 `external_url`
- 在 Windows 上配置端口转发规则
- 在 Windows 的防火墙上添加 tcp 80 端口入站规则

修改 Gitlab 配置：

```bash
sudo vim /etc/gitlab/gitlab.rb
```

将 `external_url` 修改为 `http://localhost`。修改配置后需要重新配置并启动 Gitlab。

然后以管理权限运行 PowerShell，添加端口转发规则（假定 wsl2 的 ip 是 172.21.11.89，使用 ifconfig 查看）：

```powershell
netsh interface portproxy add v4tov4 listenport=80 listenaddress=0.0.0.0 connectport=80 connectaddress=172.21.11.89 protocol=tcp
```

然后在 Windows 防火墙中添加 tcp 80 入站规则，放开 80 端口访问。

通过如下命令查询已配置的端口转发规则：

```powershell
netsh interface portproxy show all
```

配置好之后，就可以在其他机器上进行访问了。

## 参考

[1] [https://packages.gitlab.com/gitlab/gitlab-ce/install#bash-deb](https://packages.gitlab.com/gitlab/gitlab-ce/install#bash-deb)

[2] [https://docs.gitlab.com/ee/api/projects.html#upload-a-file](https://docs.gitlab.com/ee/api/projects.html#upload-a-file)

[3] [https://blog.csdn.net/cf313995/article/details/108871531](https://blog.csdn.net/cf313995/article/details/108871531)
