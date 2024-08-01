+++
title = "一个sshd崩溃的修复记录"
date = "2024-08-02T00:51:34+08:00"
tags = []
slug = "a-record-of-repairs-for-an-sshd-crash"
+++
# 前情提要

有一台`amd 5800h`的迷你主机，放在实验室机房中，装了debian12和docker，docker中跑了一系列的http服务，包括今天的主人——portainer。

在大概两周前，发现我的服务器的ssh莫名其妙的出现问题，除了ssh其他服务一切正常，当时不知道原因，尝试了各种方法无果，只能让在学校的学长帮忙手动重启。当时觉得这是偶发事件，为了以防万一还装了code-server，即使ssh挂了还有web界面能输指令。

但是今天下午用code-server敲代码的时候，突然发现code-server并不好用了（可能是因为插件问题），于是重启，重启完发现code-server直接502了，于是ssh希望看看什么问题，然后发现ssh也无法使用了。

# 修复过程

## 现在的情况

所以，现在来总结一下我的局面：

- ssh完全不可用，不管是电脑用vpn在内网访问还是frp，都不可用，说明至少是sshd的问题

    ```shell
    ➜  ~ ssh -vvv beelink_local
    OpenSSH_9.6p1, LibreSSL 3.3.6
    debug1: Reading configuration data /Users/silentdragon/.ssh/config
    debug3: /Users/silentdragon/.ssh/config line 1: Including file /Users/silentdragon/.orbstack/ssh/config depth 0
    debug1: Reading configuration data /Users/silentdragon/.orbstack/ssh/config
    debug1: /Users/silentdragon/.ssh/config line 20: Applying options for beelink_local
    debug1: Reading configuration data /etc/ssh/ssh_config
    debug1: /etc/ssh/ssh_config line 21: include /etc/ssh/ssh_config.d/* matched no files
    debug1: /etc/ssh/ssh_config line 54: Applying options for *
    debug2: resolve_canonicalize: hostname 192.168.*.* is address
    debug3: expanded UserKnownHostsFile '~/.ssh/known_hosts' -> '/Users/silentdragon/.ssh/known_hosts'
    debug3: expanded UserKnownHostsFile '~/.ssh/known_hosts2' -> '/Users/silentdragon/.ssh/known_hosts2'
    debug1: Authenticator provider $SSH_SK_PROVIDER did not resolve; disabling
    debug3: channel_clear_timeouts: clearing
    debug3: ssh_connect_direct: entering
    debug1: Connecting to 192.168.*.* [192.168.*.*] port 22.
    debug3: set_sock_tos: set socket 3 IP_TOS 0x48
    debug1: connect to address 192.168.*.* port 22: Connection refused
    ssh: connect to host 192.168.*.* port 22: Connection refused
    ```

    

- http服务完全可用，（除了抽风的code-server），包括portainer，grafana，gogs...

## 解决过程

### 第一个尝试的方案

依赖这两个前置条件，在没有能力线下查看机器的前提下，我们能做的似乎只有尝试远程找出问题的原因。

首先，我们第一应该看的是ssh客户端日志：

```shell
~ ssh: connect to host 192.168.*.* port *: Connection refused
~ kex_exchange_identification: Connection closed by remote host
```

两个分别是本地和frp情况下尝试ssh的报错，其实并没有很多有用的信息，上网搜的很多原因，比如ssh连接数满等。

网上给出的大多数解决方案都需要修改ssh服务端，显然我们不具备这种条件。但是换种思路，如果真的是连接数满，那我们清除一些外部连接是否可以呢？

于是问题 变成了，我们如何清除外部与我们的连接。最方便的事，把frp服务端关闭，肯定可以减少很多外部连接。但是后来发现及时关闭了外部连接，依然无法使用ssh（可以确定的是外部连接确实关闭了，利用后文说的docker的方案来验证的，在此不赘述了）

### 第二种方案

第一个思路失效了，我们进行了一些尝试，但是依然无法恢复。但是其实并不是完全没有启发，在这个事件中，最影响我们的是什么呢？没错，宿主机运行良好，但是我们无法进入宿主机。如果能够进入宿主机，不管是看ssh日志，systemctl中查看ssh的状态，我们都有无数方法可以debug，甚至最极端的例子是可以直接reboot，重启后就至少可以ssh连接上了。所以说，我们希望，能够执行宿主机命令。而在我已有的条件中，唯一可能的就是利用portainer了，先给不了解portainer的同学介绍一下：portainer是一个docker的web管理平台，我们通过portainer可以在web界面中，查看你的容器，新建一个容器并进行各种配置，进入容器的交互环境，查看容器日志等...总的来说是一个相当强大的docker管理平台。

那么，我们现在的目标就是：使用docker，来执行宿主机命令。

乍一看，似乎属于docker逃逸等话题，由于我对安全方面了解不多，所以利用容器逃逸对于我来说有一定难度。但是我们其实并不一定需要这个，有一些更简单的工具供我们使用，比如nsenter。

nsenter是一个可以让你在别的命名空间中执行命令的工具，较为常见的用法是，容器的工具可能较少，我们使用宿主机的工具来对容器内部进行调试，而我们的目的刚好反过来——使用容器内部的工具，调试宿主机。

首先我们要解决的第一个问题：我们需要一个可以访问宿主机命名空间的镜像，如何获取呢？

我们肯定会想到利用portainer，但是portainer有一个很不爽的设定（至少是在这个场景下）：portainer无法直接用类似`docker run`的命令行方式来启动容器，只能用可视化界面，作者在github解释为了用户安全原因，而我们为什么需要命令行启动docker呢，因为访问宿主机命名空间，需要`--pid=host`，而portainer是没有提供这个选项的。所以我们的目标变成了：如何启动一个自定义的docker，利用portainer？

如果你有折腾Jenkins等ci/cd的工具，你一定会想到docker中的Jenkins也有类似的需求，我们是怎么解决的呢？没错，启动一个容器，挂载宿主机的`/var/run/docker.sock`和`/usr/bin/docker`，这样子在容器中就可以使用宿主机的docker bin。

依靠这种思想，我们用portainer创造了一个容器，镜像是ubuntu:22.04，没有特别的设置，只是把上面提到的两个路径挂载了出来，创建之后，我们利用portainer的可视化界面进入容器内部，`docker run --name recover--privileged --net=host --ipc=host --pid=host -it --rm docker.io/ubuntu:22.04`，启动了一个我们需要的新容器。

有了这个新容器，我们现在可以去portainer中，找到这个新容器，同样进入交互界面，输入`nsenter -a -t 1 sh -c "ps -ef"`其中`-t 1`是进入pid为1的空间，即宿主机的空间，查看输出，我们发现正是宿主机正在运行的进程，这就代表我们成功了。

接下来的事，就水到渠成了，我们输入`nsenter -a -t 1 sh -c "systemctl status sshd"`，查看sshd进程的状态。

```shell
root@beelink:/# nsenter -a -t 1 sh -c "systemctl status sshd"
× ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: failed (Result: exit-code) since Mon 2024-07-29 12:32:38 CST; 3 days ago
   Duration: 2w 5d 18h 54min 3.570s
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 714 ExecStart=/usr/sbin/sshd -D $SSHD_OPTS (code=exited, status=255/EXCEPTION)
   Main PID: 714 (code=exited, status=255/EXCEPTION)
        CPU: 1h 12min 57.169s

Jul 29 12:32:38 beelink systemd[1]: ssh.service: Unit process 2423720 (sshd) remains running after unit stopped.
Jul 29 12:32:38 beelink systemd[1]: ssh.service: Unit process 2423721 (sshd) remains running after unit stopped.
Jul 29 12:32:38 beelink systemd[1]: ssh.service: Unit process 2423723 (sshd) remains running after unit stopped.
Jul 29 12:32:38 beelink systemd[1]: ssh.service: Consumed 1h 12min 57.156s CPU time.
Jul 29 12:32:38 beelink sshd[2423723]: fatal: Missing privilege separation directory: /run/sshd
Jul 29 12:32:38 beelink sshd[2423720]: pam_unix(sshd:auth): check pass; user unknown
Jul 29 12:32:38 beelink sshd[2423720]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rho>
Jul 29 12:32:39 beelink sshd[2423718]: Connection closed by invalid user kcw 192.168.*.* port * [preauth]
Jul 29 12:32:39 beelink sshd[2423720]: Failed password for invalid user melony from 192.168.*.* port * ssh2
Jul 29 12:32:40 beelink sshd[2423720]: Connection closed by invalid user melony 192.168.*.* port * [preauth]
```

我们发现，sshd进程果然有问题，而也正有一个fatal报错：`fatal: Missing privilege separation directory: /run/sshd`。

我们直接谷歌这个报错，发现有非常简单的解决方案，直接mkdir一个报错中缺失的路径即可，

```shell
root@beelink:/# nsenter -a -t 1 sh -c "mkdir /run/sshd"
root@beelink:/# nsenter -a -t 1 sh -c "systemctl restart  sshd"
root@beelink:/# nsenter -a -t 1 sh -c "systemctl status sshd"
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Thu 2024-08-01 23:48:44 CST; 3s ago
```

现在，sshd服务已经恢复了正常，而我们再次尝试ssh连接，发现已经可以连接了。

总的来说，整体的思路其实就是思考如何在宿主机执行命令->利用docker资源->docker执行宿主机命令->使用nsenter工具->获得一个pid=host的docker->执行命令排查。

希望这个博客可以给正在看的你一些帮助和启发。
