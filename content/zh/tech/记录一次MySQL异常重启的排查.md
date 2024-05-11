+++
title = "记录一次MySQL异常重启的排查"
date = "2024-05-11T12:17:33+08:00"
tags = []
slug = "Record-the-troubleshooting-of-an-abnormal-mysql-restart"
+++
发现公司的开发机上跑的web服务这两天总是异常重启，排查了之后发现是因为mysql挂了，所以web也自动挂了，所以问题就变成了为什么mysql会经常异常重启。

首先想到的第一个思路是看mysql的运行日志，mysql时跑在docker中的，所以直接docker logs。
![image-20240511114754011](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/image-20240511114754011.png)

发现只有只有一些重启的记录，但是没有mysql本身crash或者发出异常signal的记录，这说明并不是mysql本身的问题。

跳出了mysql本身oom的问题后，mysql会down掉只可能是因为被系统杀死了，我们可以进入到系统日志中查找，我们可以使用`dmesg`命令来诊断，`dmesg`的作用是显示*nix系统中内核环行形缓冲区的信息，包含了系统启动和运行期间内核活动的信息。

```shell
[root@localhost ~]# dmesg | grep mysql
[1129388.678972] [ 1750]   999  1750  5186616   874771    2262        0             0 mysqld
[1129388.679131] Out of memory: Kill process 1750 (mysqld) score 215 or sacrifice child
[1129388.679213] Killed process 1750 (mysqld) total-vm:20746464kB, anon-rss:3499084kB, file-rss:0kB, shmem-rss:0kB
[1152469.128610] [12657]   999 12657  2213712   179707     607        0             0 mysqld
[1211224.991271] [12657]   999 12657  5130055   846548    2077        0             0 mysqld
[1211224.991519] Out of memory: Kill process 12657 (mysqld) score 208 or sacrifice child
[1211224.991589] Killed process 12657 (mysqld) total-vm:20520220kB, anon-rss:3386192kB, file-rss:0kB, shmem-rss:0kB
[1219478.239851] [ 7749]   999  7749  3681289   612374    1478        0             0 mysqld
[1219478.240185] Out of memory: Kill process 7749 (mysqld) score 150 or sacrifice child
[1219478.240297] Killed process 7749 (mysqld) total-vm:14725156kB, anon-rss:2449496kB, file-rss:0kB, shmem-rss:0kB
[1225332.187678] [ 8807]   999  8807  2830571   366038     993        0             0 mysqld
[1225702.625051] [ 8807]   999  8807  2836460   483886    1220        0             0 mysqld
[1225702.625414] Out of memory: Kill process 8807 (mysqld) score 119 or sacrifice child
[1225702.625459] Killed process 8807 (mysqld) total-vm:11345840kB, anon-rss:1934972kB, file-rss:572kB, shmem-rss:0kB
[1230146.008989] [24278]   999 24278  2424195   224530     719        0             0 mysqld
[1233232.486908] [24278]   999 24278  2432709   322806     942        0             0 mysqld
[1242588.070982] [24278]   999 24278  2467363   395678    1219        0             0 mysqld
[1242588.071280] Out of memory: Kill process 24278 (mysqld) score 97 or sacrifice child
[1242588.071338] Killed process 24278 (mysqld) total-vm:9869452kB, anon-rss:1582712kB, file-rss:0kB, shmem-rss:0kB
```

`dmesg`后发现有很多关于`mysqld`的异常报错，日志最前的时间戳表示的是自开机以来的时间戳，`who -b`得到上次开机时间为`系统引导 2024-04-26 11:57`，换算一下时间刚好为昨天崩溃的时间，所以可以判断出崩溃的真实原因是系统的内存太小了，开了过多的服务导致系统oom，系统oom后会根据进程的内存占用计算出一个score（内存占用是一方面，具体的score计算还受其他因素影响），score越高，被杀死的优先级越高。而发现mysql给了10g的内存，所以几乎每次oom都会杀掉mysql，这就是问题产生的原因。而为什么会突然系统oom呢，推测的原因是因为在跑的服务中有airflow，有时候并行执行的任务过多可能导致内存占用突然飙高，于是导致了系统的oom，可以通过限制airflow的并发任务数来解决这个问题。

一些补充：

- 还可以使用`cat /var/log/messages | grep -i memory`来看上面的日志

    ```shell
    [root@localhost ~]# cat /var/log/messages | grep -i memory
    May  9 13:28:21 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May  9 13:28:21 localhost kernel: Out of memory: Kill process 1750 (mysqld) score 215 or sacrifice child
    May  9 19:53:01 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May  9 19:53:01 localhost kernel: Out of memory: Kill process 21438 (python) score 319 or sacrifice child
    May 10 12:12:18 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May 10 12:12:18 localhost kernel: Out of memory: Kill process 12657 (mysqld) score 208 or sacrifice child
    May 10 12:12:18 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May 10 12:12:18 localhost kernel: Out of memory: Kill process 12865 (ib_log_flush) score 208 or sacrifice child
    May 10 14:29:51 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May 10 14:29:51 localhost kernel: Out of memory: Kill process 7749 (mysqld) score 150 or sacrifice child
    May 10 16:07:24 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May 10 16:07:24 localhost kernel: Out of memory: Kill process 15275 (airflow task ru) score 125 or sacrifice child
    May 10 16:13:35 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May 10 16:13:35 localhost kernel: Out of memory: Kill process 8807 (mysqld) score 119 or sacrifice child
    May 10 16:13:35 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May 10 16:13:35 localhost kernel: Out of memory: Kill process 27647 ([ready] gunicor) score 100 or sacrifice child
    May 10 17:27:38 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May 10 17:27:38 localhost kernel: Out of memory: Kill process 13053 (airflow task ru) score 211 or sacrifice child
    May 10 18:19:05 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May 10 18:19:05 localhost kernel: Out of memory: Kill process 28586 (airflow task ru) score 125 or sacrifice child
    May 10 20:55:01 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May 10 20:55:01 localhost kernel: Out of memory: Kill process 24278 (mysqld) score 97 or sacrifice child
    May 10 20:55:01 localhost kernel: [<ffffffffab1baab6>] out_of_memory+0x4b6/0x4f0
    May 10 20:55:01 localhost kernel: Out of memory: Kill process 7116 (airflow task ru) score 52 or sacrifice child
    ```

    
