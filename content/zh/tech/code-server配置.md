+++
title = "Code Server配置"
date = "2023-06-04T14:28:16+08:00"
tags = []
slug = ""
+++

# code-server 简介

目前 *Coder Technologies Inc, an Austin TX company* 公司开源了一个基于服务器端的 VScode -- code-server，只要服务器端配置好code-server，就可以在任何浏览器上使用VScode 。在code-server上可以自由的使用vscode中的插件，也可以快速的配置好一套云开发环境。

下面来讲解一下code-server在linux上的下载与配置。

```shell
主机环境：（linux系统均可）
debian11
```

# 下载

## 1. 下载code-server的二进制文件

**[code-server官方github地址](https://github.com/coder/code-server)**

在release的asset中找到最新版本，此时是4.13.0

![image-20230604213416888|400](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20230604213416888.png)

根据自己的linux服务器架构选择对应的版本，可以用`arch` 命令查看，例如我就是x86架构，应该选择amd64。

```shell
root@vultr:~# arch
x86_64
```

使用wget来从github获取到压缩包

```shell
# wget https://github.com/coder/code-server/releases/download/v4.13.0/code-server-4.13.0-linux-amd64.tar.gz
# ls
code-server-4.13.0-linux-amd64.tar.gz
//解压
# tar -zxvf code-server-4.13.0-linux-amd64.tar.gz
//此时就可以删除压缩包
# rm code-server-4.13.0-linux-amd64.tar.gz 
//给文件夹改名
# mv code-server-4.13.0-linux-amd64/ code-server

```

## 运行可执行文件

```shell
//进入code-server的bin目录
# cd code-server/bin
# ls
code-server  //只有一个code-server的可执行文件
//执行code-server
# ./code-server
[2023-06-04T13:47:08.159Z] info  Wrote default config file to ~/.config/code-server/config.yaml
[2023-06-04T13:47:08.744Z] info  code-server 4.13.0 2798322b03e7f446f59c5142215c11711ed7a427
[2023-06-04T13:47:08.747Z] info  Using user-data-dir ~/.local/share/code-server
[2023-06-04T13:47:08.763Z] info  Using config file ~/.config/code-server/config.yaml
[2023-06-04T13:47:08.764Z] info  HTTP server listening on http://127.0.0.1:8080/
[2023-06-04T13:47:08.764Z] info    - Authentication is enabled
[2023-06-04T13:47:08.765Z] info      - Using password from ~/.config/code-server/config.yaml
[2023-06-04T13:47:08.766Z] info    - Not serving HTTPS
```

这时候还不能直接访问，因为开放连接的ip是`127.0.0.1`，除了本机外无法访问。

## 更改配置文件

可以去配置文件中修改，配置文件的路径是上面`[2023-06-04T13:47:08.763Z] info  Using config file ~/.config/code-server/config.yaml`中的`~/.config/code-server/config.yaml`，vim修改。

```shell
# vim ~/.config/code-server/config.yaml
bind-addr: 127.0.0.1:8080
auth: password
password: 25d2c03cc74d3e3f6c56499a
cert: false
```

bind-addr是你允许的访问的ip以及在那个端口开放，例如，如果想允许所有IP访问并开放8077端口，应该写`0.0.0.0:8077`。同时需要开放服务器的8077端口，如果云服务器有防火墙还需要去云服务器控制台开放8077端口。

password是指你的访问密码，最好设置的复杂一点。

然后保存文件并退出。

## 再次运行

再次`./code-server`

```shell
root@vultr:~/code-server/bin# ./code-server
[2023-06-04T14:24:04.718Z] info  code-server 4.13.0 2798322b03e7f446f59c5142215c11711ed7a427
[2023-06-04T14:24:04.723Z] info  Using user-data-dir ~/.local/share/code-server
[2023-06-04T14:24:04.740Z] info  Using config file ~/.config/code-server/config.yaml
[2023-06-04T14:24:04.740Z] info  HTTP server listening on http://0.0.0.0:8077/
[2023-06-04T14:24:04.741Z] info    - Authentication is enabled
[2023-06-04T14:24:04.742Z] info      - Using password from ~/.config/code-server/config.yaml
[2023-06-04T14:24:04.742Z] info    - Not serving HTTPS
```

这时候code-server就启动成功了，可以去浏览器输入<你的ip>:8077访问code-server，例如：108.78.89.88:8077

![image-20230604222509505](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20230604222509505.png)

出现这个界面就成功了，输入刚刚的密码，进入。

![image-20230604222828825](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20230604222828825.png)

这和我们本地的vscode几乎一样的。

## 安装插件

可以去插件市场搜索chinese插件，更换语言，并且下载一些自己需要的插件。有一些插件无法下载。我们可以从vscode扩展商店的网站上下载`.vsix`文件来手动安装。

[微软插件市场](https://marketplace.visualstudio.com/VSCode)

![image-20230604223222729](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20230604223222729.png)

可以在这里下载，然后通过sftp上传到服务器，在code-server的插件界面可以选择`.vsix`文件安装拓展。

![image-20230604223350938](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20230604223350938.png)

##  在screen中启动

目前这个code-server只存在于当前这个连接中，连接一旦断开就没有了。我们可以用screen，nohup或者tmux来让他在后台一直运行。

例如screen：

首先安装screen：

```shell
# apt install screen
```

开启一个新的screen(命名为"code-server")：

```shell
# screen -S "code-server" 
```

现在即使断开连接，code-server在后台也会一直运行。

ps. 一些常见的screen命令

```shell
screen -ls 显示进程列表
screen -r sid 恢复某个进程
screen -X -S sid quit 终止某个进程
screen -S my_screen_name 修改会话名称
ctrl+a d 离开当前进程
ctrl+a k 终止当前进程
```