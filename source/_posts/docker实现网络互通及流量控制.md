---
title: docker实现网络互通及流量控制
date: 2025-10-31 21:44:23
categories: docker
tags: 网络
cover: /imgs/default-cover.jpeg
---

## 创建docker虚拟网络：

```sh
docker network create -d bridge [网络名]
```

比如，创建一个网络名叫test-bridge的虚拟网络，命令如下：

```sh
docker network create -d bridge test-bridge
```

创建好后，可以通过以下命令查看新建的网络信息：

```sh
docker network ls
```

输出如下：

```text
NETWORK ID     NAME          DRIVER    SCOPE
66517e12ce54   test-bridge   bridge    local
9d5286df7a7f   bridge        bridge    local
77820f87e156   host          host      local
574c413719ba   none          null      local
```

如果想删除创建的网络test-bridge，输入以下命令：

```sh
docker network rm test-brigde
```

## 创建docker时候指定虚拟网络

比如，我想创建两个容器，分别为alice和bob，我想让它们能够相互通信，则在创建的时候可以指定一个虚拟网络（如test-bridge），使它们在同一虚拟网络下，这样可以实现alice和bob的相互通信，方便后续进行网络控制等。命令如下：

```sh
docker run -idt --name=alice --net=test-bridge --cap-add=NET_ADMIN ubuntu:latest bash
docker run -idt --name=bob --net=test-bridge --cap-add=NET_ADMIN ubuntu:latest bash
```

分别进入两个容器，查看是否在同一网段内。

alice:
```sh
# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.2  netmask 255.255.0.0  broadcast 172.18.255.255
        ether 02:42:ac:12:00:04  txqueuelen 0  (Ethernet)
        RX packets 26017  bytes 34691727 (34.6 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 19947  bytes 1322869 (1.3 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

bob:
```sh
# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.3  netmask 255.255.0.0  broadcast 172.18.255.255
        ether 02:42:ac:12:00:04  txqueuelen 0  (Ethernet)
        RX packets 26017  bytes 34691727 (34.6 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 19947  bytes 1322869 (1.3 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

可以看到alice和bob在同一网段内。

注：若提示ifconfig命令未找到，安装net-tools即可：

```sh
apt install net-tools
```

## 安装流量控制工具：

```sh
apt install iproute2
```

## 流量控制

首先查看是否有已经配置好的流量控制：

```bash
tc qdisc show dev eth0 root 
```

若有，通过以下命令删除：

```bash
tc qdisc del dev eth0 root 
```

进行流量控制：20ms 延迟，100Mbit 带宽

```bash
tc qdisc add dev eth0 root handle 1: netem delay 20ms rate 100mbit
```