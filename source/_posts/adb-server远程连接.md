---
title: adb-server 远程连接
date: 2020-09-03 09:38:34
tags:
- android
---


# 0x01 背景

最近的项目里遇到个奇葩需求。

有一堆手机们用 USB 线连接到了一台开发机上，现在有另外一些机器，需要通过网络远程连接到这些手机上，并且能直接连上 adb 执行命令。

我第一反应是`adb connect ip:port`，但仔细一想好像不太对，这种方式需要手机开了网络调试，而且绕过开发机直接连手机，也不符合目前需求。

网上搜了下，大多数 adb 远程连接指的都是通过`adb connect`的这种连接方式，而如何连接到远程机器上的 USB 设备却没有太多的资料。

<!-- more -->

# 0x02 adb 架构及工作原理

为了解决这个需求，首先要了解下 adb 的架构和工作原理。

## adb 架构

adb 从架构上来看大致长这样：

<img src="https://i.loli.net/2020/09/05/C27Egj3YvyrNsXH.png" alt="image-20200905220532680" style="zoom:67%;" />

- adbd ：运行在手机上的后台常驻进程，由 `init` 负责启动，等待 adb-server 连接；
- adb-server：运行在开发机上的后台常驻进程，默认监听本地 5037 端口，等待 adb-client 连接，负责管理所有连接的设备；
- adb-client：运行在开发机上，可通过 shell 调用，对 adb-server 连接的设备进行操作。


## adb 工作原理

adb-server 在开发机上，后台常驻进程，默认监听本地 *localhost:5037* 端口。用户每次从 shell 执行 adb 命令，将会在本地启动一个 adb-client 进程，adb-client 负责建立一对儿 socket 连接，与本地的 adb-server 进行通信，使用基于 tcp 的 adb 私有协议。

![image-20200903091922452](https://i.loli.net/2020/09/03/mnWYSi9zIw23QVA.png)

adb-client 启动时会去找本地的 adb-server 实例，如果没有，会试图启动一个。**实际上，adb-server 和 adb-client 是同一个可执行文件**(*adb*)。

adb-server 和 adbd 之间通讯可以通过 USB 口或网络进行连接，同样使用基于 TCP 的 adb 私有协议进行通讯，每个设备中的 adbd 只能连接一个 adb-server。

![image-20200905220433628](https://i.loli.net/2020/09/05/RjIvts1k6Fy893O.png)

adb-server 和 adb-client 之间的连接，绝大多数情况下都是跑在同一台机器上，但我们目前的场景下，adb-server 运行在内网的一台开发机上，adb-client 需要在某个 docker 实例中，**这种 adb-server 和 adb-client 不在一台机器上的情况，翻了好久文档也没找到连接方法。**

# 0x03 解决方案

仔细看 *adb --help* 的输出：

![image-20200903092807405](https://i.loli.net/2020/09/03/OtEe25KGY9ga3UN.png)

*-a -H* 和 *-P* 这三个选项没用过，但看上去和我们的情况沾边儿。

但是文档中并没有这几个选项的任何说明，也没有给出如何以 server / client 方式启动 adb，而 *--help* 又输出了这样的选项，那么我们翻一翻 adb 源码，看下怎么整。

https://android.googlesource.com/platform/system/core/+/master/adb/client/commandline.cpp

`adb_commandline()` 这个方法中，找一找 -a、-H、-P 参数后面的逻辑。

```c
int adb_commandline(int argc, const char** argv) {
    bool no_daemon = false;
    bool is_daemon = false;
    bool is_server = false;
   //...
    while (argc > 0) {
        if (!strcmp(argv[0], "server")) {
            is_server = true;
        } else if (!strcmp(argv[0], "nodaemon")) {
            no_daemon = true;
        } else if (!strcmp(argv[0], "fork-server")) {
            /* this is a special flag used only when the ADB client launches the ADB Server */
            is_daemon = true;
        } 
        //...
        } else if (!strcmp(argv[0], "-a")) {
            gListenAll = 1;
        } else if (!strncmp(argv[0], "-H", 2)) {
            if (argv[0][2] == '\0') {
                if (argc < 2) error_exit("-H requires an argument");
                server_host_str = argv[1];
                argc--;
                argv++;
            } else {
                server_host_str = argv[0] + 2;
            }
        } else if (!strncmp(argv[0], "-P", 2)) {
            if (argv[0][2] == '\0') {
                if (argc < 2) error_exit("-P requires an argument");
                server_port_str = argv[1];
                argc--;
                argv++;
            } else {
                server_port_str = argv[0] + 2;
            }
        } 
    //...
    }

```

发现了`server` `nodaemon` 这两个参数，继续往后看代码，发现以这两个参数启动 adb ，adb 将会以 adb-server 模式启动，并从标准输入接收数据，符合我们在插手机的开发机上的需求。

而需要远程连接的机器上，指定 *-P port* 和 *-H host* 为开发机上的 adb-server 即可。

- 最终启动命令

> 插手机的开发机上：`adb -a -P {port} nodaemon server`

> 远程 adb-client 中：`adb -H {device_proxy_ip} -P {port} {其他 adb 命令}`

- 测试一下

插手机的开发机上：

![image-20200903093548536](https://i.loli.net/2020/09/03/gpaljD6xfzAS5Cy.png)

线没插好，忽视报错，重新插拔 USB 线即可。

远程 adb-client 上：

![image-20200903093711504](https://i.loli.net/2020/09/03/ulP2HWOZyKE7RBb.png)

连接正常。

至此解决 adb-server 和 adb-client 之间通过网络远程连接。

实际上，https://android.googlesource.com/platform/system/core/+/master/adb/SOCKET-ACTIVATION.txt  这里有给出这种方式启动 adb 的例子，捉迷藏一样，interesting.