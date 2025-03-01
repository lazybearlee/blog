---
title: Docker网络
date: 2024-09-16 13:15
tags:
  - Docker
  - Note
---

> [!note] Docker网络覆盖范围与网络查看
> - 从覆盖范围上，可以分为单个host上的容器网路以及跨多个host的网络
> - 使用`docker network ls` 查看已创建的网络

## 5.1 none网络

> [!summary] none网络就是什么都没有的网络。挂在这个网络下的容器除了lo，没有其他任何网卡。容器创建时，可以通过 `--network=none` 指定使用none网络
> 

```shell
$ docker run -it --network=none busybox
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
2fce1e0cdfc5: Pull complete 
Digest: sha256:c230832bd3b0be59a6c47ed64294f9ce71e91b327957920b6929a0caa8353140
Status: Downloaded newer image for busybox:latest
/ # ifconfig
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ # 
```

> [!note] 使用 none 则创建了一个封闭的网络，封闭意味着隔离，一些对安全性要求高并且不需要联网的应用可以使用none网络。
> 

## 5.2 host网络

> [!summary] 连接到host网络的容器共享Docker host的网络栈，容器的网络配置与host完全一样。可以通过 `--network=host` 指定使用host网络，而且我们还会注意到连hostname也和host一样。
> 

```shell
$ docker run -it --network=host busybox
/ # ifconfig
br-d71b316c20a8 Link encap:Ethernet  HWaddr 02:42:99:95:64:8D  
          inet addr:172.18.0.1  Bcast:172.18.255.255  Mask:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

docker0   Link encap:Ethernet  HWaddr 02:42:C8:A9:01:91  
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          inet6 addr: fe80::42:c8ff:fea9:191/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:5 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:526 (526.0 B)

eth0      Link encap:Ethernet  HWaddr 00:15:5D:E3:EC:55  
          inet addr:172.27.47.144  Bcast:172.27.47.255  Mask:255.255.240.0
          inet6 addr: fe80::215:5dff:fee3:ec55/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:257755 errors:0 dropped:0 overruns:0 frame:0
          TX packets:95729 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:377574429 (360.0 MiB)  TX bytes:7164748 (6.8 MiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:63636 errors:0 dropped:0 overruns:0 frame:0
          TX packets:63636 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:146451727 (139.6 MiB)  TX bytes:146451727 (139.6 MiB)

/ # hostname
lazybear
```

> [!note]
> - 直接使用Docker host的网络最大的好处就是**性能**，如果容器对网络传输效率有较高要求，则可以选择host网络。当然不便之处就是牺牲一些灵活性，比如要考虑**端口冲突问题**，Docker host上已经使用的端口就不能再用了。
> - Docker host的另一个用途是让**容器可以直接配置host网路**，比如某些跨host的网络解决方案，其本身也是以容器方式运行的，这些方案需要对网络进行配置，比如管理iptables

## 5.3 bridge网络

> [!summary] Docker安装时会创建一个命名为docker0的Linux bridge。如果不指定`--network`，创建的容器默认都会挂到docker0上
> 

```shell
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
77f059b22353   bridge    bridge    local
a18be4245d81   host      host      local
aa8dc8e0b0e4   none      null      local
```

```shell
$ brctl show
bridge name     bridge id               STP enabled     interfaces
br-d71b316c20a8 8000.02429995648d       no
docker0         8000.0242c8a90191       no
```

然后我们部署一个容器：

```shell
$ docker run -d docker.io/apitable/web-server
Unable to find image 'apitable/web-server:latest' locally
latest: Pulling from apitable/web-server
df9b9388f04a: Pull complete 
70c90f7de7cb: Pull complete 
f83937c3ce37: Pull complete 
98b78bba1d70: Pull complete 
87c1986a1762: Pull complete 
68181ccfd26d: Pull complete 
f0bca8e8a68a: Pull complete 
8bc12e1e76f0: Pull complete 
d1f91b85ede6: Pull complete 
f84ce17c9019: Pull complete 
6630b24e3a91: Pull complete 
a5cf5a18079c: Pull complete 
4f4fb700ef54: Pull complete 
Digest: sha256:6afa132b9ad610be862585f832b87e4608597c52c170f94fb9581dc372c2a596
Status: Downloaded newer image for apitable/web-server:latest
0e7d0b48abd77fdfd6caea743ddd29c23bfeb972be6fe566ee609761a73bd35d
lee@lazybear:~$ brctl show
bridge name     bridge id               STP enabled     interfaces
br-d71b316c20a8         8000.02429995648d       no
docker0         8000.0242c8a90191       no              veth3ce435d
```

我们会发现，docker0已经挂载了一个新的接口：veth3ce435d。这时候我们再去容器内查看其网卡：

```shell
lee@lazybear:~$ docker exec -it 0e7d0b48abd7 /bin/sh
/app/packages/datasheet $ ls
next.config.js  node_modules    package.json    public          server.js       src             web_build
/app/packages/datasheet $ ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02  
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:13 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1086 (1.0 KiB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/app/packages/datasheet $ 
```

我们会发现其网卡为 eth0，而不是veth3ce435d。实际上，它们是一对veth pair。veth pair是一种成对出现的特殊网络设备，可以把它们想象成由一根虚拟网线连接起来的一对网卡，网卡的一头（eth0）在容器中，另一头（而不是veth3ce435d）挂在网桥docker0上，其效果就是将eth0也挂在了docker0上。

```shell
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "77f059b2235372eedbc762b11c6796fcd7cd6a1d5fdefcf81e16bd4afbe21d23",
        "Created": "2024-09-16T11:36:54.537899041+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        ...
    }
]
```

而我们还会发现，bridge网络配置的subnet就是172.17.0.0/16，并且网关是172.17.0.1。

实际上的网络拓扑如下所示：

![](Blog/Docker/assets/Pasted%20image%2020250301180831.png)

## 5.4 user-defined网络

> [!summary] 除了none、host、bridge这三个自动创建的网络，用户也可以根据业务需要创建user-defined网络。Docker提供三种user-defined网络驱动：**bridge、overlay和macvlan**。overlay和macvlan用于创建跨主机的网络。
> 

```shell
docker network create --driver bridge net_name
```

此前我已经创建一个 es 的网络：

```shell
$ brctl show
bridge name     bridge id               STP enabled     interfaces
br-d71b316c20a8 8000.02429995648d       no
docker0         8000.0242c8a90191       no
```

br-d71b316c20a8 即是其网络的短id。我们可以用 `docker network inspect`查看其配置信息。

```shell
$ docker network inspect es-net
[
    {
        "Name": "es-net",
        "Id": "d71b316c20a874f9cfeb5f1ebc9c0a2605cb8da27134233e54d389b60c12a0f6",
        "Created": "2024-04-05T20:30:29.734391859+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

而 `172.18.0.0/16` 就是Docker自动分配的IP网段。

> [!tip] 我们也可以通过在创建网段时指定 `--subnet`和`--gateway`参数来指定IP网段。
> 

#### 容器怎么使用我们的自定义网络呢？

> [!note] 容器要使用新的网络，需要在启动时通过 `--network`指定，IP会由dockers自动从subnet中分配。不过，我们也可以通过`--ip`参数指定。
> 

> [!note] 同一网络中的容器、网关之间都是可以通信的。而在不同网桥中的网络是不能通信的。
> 

#### 如何让两个不同网桥中的容器通信？

> [!note] 为容器添加一块另一个网桥的网卡。这个可以通过`docker network connect`命令实现。
> 

## 5.5 容器间通信

> [!summary] 容器之间可通过IP, Docker DNS Server或joined容器三种方式通信。

### IP通信

> [!note] 两个容器要能通信，必须要有属于同一个网络的网卡。
> 

### Docker DNS Server

> [!note] Docker 自带一个 DNS Server，使容器可以直接通过容器名通信，只需要在启动容器时使用 `--name` 为容器命名即可。
> 

> [!attention] 使用docker DNS有个限制：只能在user-defined网络中使用。也就是说，默认的bridge网络是无法使用DNS的。
> 

### joined容器

> [!note] joined容器可以使两个或多个容器共享一个网络栈，共享网卡和配置信息，joined容器之间可以通过127.0.0.1直接通信。
> 

> [!example] 如何采用joined容器进行通信
> - 首先创建容器 `docer run -d --name=web1 xxx`
> - 创建另一个容器，并使用 `--network=container:web1` 来指定joined容器为web1.

> [!note] joined容器适合的场景
> （1）不同容器中的程序希望通过loopback高效快速地通信，比如Web Server与App Server。
> （2）希望监控其他容器的网络流量，比如运行在独立容器中的网络监控程序。

## 5.6 将容器与外部世界连接

### 容器如何访问外部世界

> [!note] docker实际是通过NAT来实现容器对外网的访问的
> （1）busybox发送ping包：172.17.0.2 \> www.bing.com
> （2）docker0收到包，发现是发送到外网的，交给NAT处理。
> （3）NAT将源地址换成enp0s3的IP:10.0.2.15 > www.bing.com
> （4）ping包从enp0s3发送出去，到达www.bing.com。

### 外部世界如何访问容器

> [!note] 外部世界是通过**端口映射**来访问到容器的
> docker可将容器对外提供服务的端口映射到host的某个端口，外网通过该端口访问容器。容器启动时通过`-p`参数映射端口。
> 而每一个映射的端口，host都会启动一个docker-proxy进程来处理访问容器的流量。
