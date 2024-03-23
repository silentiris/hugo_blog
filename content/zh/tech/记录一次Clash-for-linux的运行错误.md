+++
title = "记录一次Clash for Linux的运行错误"
date = "2023-10-05T14:33:55+08:00"
tags = []
slug = "Log-a-Clash-for-Linux-run-error"
+++

项目地址：[clash-for-linux](https://github.com/wanhebin/clash-for-linux)

1. 首先是`sh start.sh`可能会启动不成功。在有些os中`/bin/sh`被更改为了dash。
    可以用`bash start.sh`

2. 在我运行时候提示运行成功，但是`lsof -i:7890`发现没有开启服务，`curl google.com`也无法连接。
    ![e376cfc8677bb11b8c795cb2c650c471.png|550](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/e376cfc8677bb11b8c795cb2c650c471.png)



3. 还试了chmod -x ，但是也没有解决，还尝试了很多方法，都没有作用，唯独忽略了去看日志，这是一个非常不好的习惯和错误。遇到了问题应该第一时间看日志的。

​	后来去./logs找到了日志，cat之后
![723c9b92aba4dd429cd88bd9eb3fd811.png|600](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/723c9b92aba4dd429cd88bd9eb3fd811.png)

发现
![93864714548aad5c3f04ed41b34fb1fc.png](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/93864714548aad5c3f04ed41b34fb1fc.png)


提示找不到这个bin....

后来cd到了bin目录，发现名字真的有问题....晕
![4.png](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/4.png)

`mv clash-linux-amd64-v1.13.0 clash-linux-amd64`后，再启动就好了。

第二个问题困扰了我很长时间，没想到最后用这种方式解决了。。。这给我两个启示

1. 出现问题一定要记得看日志！！！
2. 在自己编写接口和服务时一定要注意给予正确的反馈，不能像这种明明都没有启动还提示启动成功的。