---
title: Docker容器
date: 2024-09-16 09:48
tags:
  - Docker
  - Note
---
> [!summary] Docker容器
> 本文档详细介绍了Docker容器的运行、管理和资源限制等方面的内容。首先，解释了如何使用`docker run`命令启动容器，并通过`docker ps`或`docker container ls`查看容器状态。接着，讨论了如何让容器长期运行，包括后台运行容器的方法和如何通过长ID、短ID和NAMES指定容器。此外，还介绍了如何进入容器，以及`docker attach`和`docker exec`命令的区别。文档还涵盖了容器的停止、启动、重启，以及如何实现容器的自动重启。此外，还探讨了容器的暂停和删除，以及容器状态机的概念。最后，文档深入讨论了如何限制容器的内存、CPU、Block IO带宽等资源，以及Docker容器实现所依赖的底层技术，包括cgroup和namespace的工作原理。

## 4.1 运行容器

> [!note] 容器运行与状态查看
> - docker run 运行容器
> - dockers ps / dockers container ls 查看容器

### 怎么让容器长期运行？

> [!note] 因为容器的生命周期依赖于启动时执行的命令，只要该命令不结束，容器也就不会退出。可以通过执行一个长期运行的命令来保持容器的运行状态。

> [!note] 以后台方式启动容器
> 使用 `docker run -d` ，容器启动后会回到docker host 的终端。docker会返回一串字符，即容器的ID。以这种方式启动时不会占据一个终端。

### 怎么指定一个容器？（长ID、短ID与NAMES）

> [!note] 长ID
> 启动容器时返回的字符串

> [!note] 短ID
> 长ID的前12个字符

> [!note] NAMES
> 容器的名字，可以通过 `--name` 参数指定，如果没有指定，则docker会自动为容器分配名字。

### 怎么进入容器？

> [!note] docker attach
> 通过docker attach可以attach到容器启动命令的终端。
> 如果需要退出，可以使用 `ctrl+p` 与 `ctrl+q` 组合键退出终端。

> [!note] dockers exec
> - 以这种方式进入容器时会执行 `exec` 后指定的命令。
> - `-it` 以交互模式打开 `pseudo-TTY`
> - 执行 `exit` 退出容器

#### attach 与 exec 有什么区别？

> [!note] 区别
> （1）attach直接进入容器启动命令的终端，不会启动新的进程。
> （2）exec则是在容器中打开新的终端，并且可以启动新的进程。
> （3）如果想直接在终端中查看启动命令的输出，用attach；其他情况使用exec。

> [!attention] 如果只是为了查看启动命令的输出，可以使用docker logs命令

## 4.2 stop/start/restart容器

> [!note]
> - 通过`docker stop`可以停止运行的容器，而容器实际上在docker host中是一个进程，`stop` 实质上是向该进程发送一个 `SIGTERM` 信号。而如果向快速停止容器，可用 `kill` 向容器进程发送 `SIGKILL` 信号。
> - 对于处于停止状态的容器，可以通过`docker start`重新启动，其会保留容器的第一次启动时的所有参数。
> - `docker restart`可以重启容器，其作用就是依次执行`docker stop`和`dockerstart`。

> [!note] SIGTERM与SIGKILL
> - **SIGTERM（终止信号）**： 
> 	- 此信号较为友好，它请求目标进程优雅地终止。当进程接收到 SIGTERM 信号时，它可以进行一些清理操作，如关闭文件描述符、保存数据等，然后再退出。 
> 	- 进程可以捕获这个信号并进行相应的处理，这给了进程一个机会来有序地结束。 
> - **SIGKILL（强制终止信号）**： 
> 	- 这个信号是强制终止进程，无法被捕获或忽略。 
> 	- 当发送 SIGKILL 信号给一个进程时，进程会立即被终止，没有机会进行任何清理操作。这可能会导致数据丢失或其他不良后果。

### 怎么让容器实现自动重启

> [!note] 在 Docker 中可以通过以下几种方式让容器实现自动重启：
>  - 使用 `restart` 策略启动容器时设置自动重启参数。例如，在使用 `docker run` 命令启动容器时，可以添加 `--restart=always` 参数。这样，无论容器因为何种原因退出，Docker 守护进程都会自动尝试重新启动容器。 
>  - 在 Docker Compose 文件中，可以在服务配置中设置 `restart` 属性为 `always`，以实现容器的自动重启。例如： 
> 	 ```yaml 
> 	 services: 
> 		 your_service: 
> 			 restart: always 
> 	```
> - 而对于`restart`的可选属性，还包括 `on-failure:3`——如果进程退出代码非0，则重启容器，最多重启三次。

## 4.3 pause / unpause容器

> [!note] 有时我们只是希望让容器暂停工作一段时间，比如要对容器的文件系统打个快照，或者dcoker host需要使用CPU，这时可以执行`docker pause`

## 4.4 删除容器

使用一段时间后，host上可能出现大量已经退出的容器：

```shell
CONTAINER ID   IMAGE                         COMMAND                  CREATED        STATUS                      PORTS                                                                     NAMES
b4217c261e6a   xuxueli/xxl-job-admin:2.3.0   "sh -c 'java -jar $J…"   5 months ago   Exited (255) 5 months ago   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp                                 xxl-job-admin
9c1d3251573d   zookeeper                     "/docker-entrypoint.…"   5 months ago   Exited (255) 5 months ago   2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, :::2181->2181/tcp, 8080/tcp   zookeeper
69567c3f8baf   openzipkin/zipkin             "start-zipkin"           5 months ago   Exited (255) 5 months ago   9410/tcp, 0.0.0.0:9411->9411/tcp, :::9411->9411/tcp                       zipkin
56580b78f6f5   consul                        "docker-entrypoint.s…"   5 months ago   Exited (1) 5 months ago
                                                                   consul
b39ff00c6620   kibana:8.6.0                  "/bin/tini -- /usr/l…"   5 months ago   Exited (137) 5 months ago
                                                                   kibana
5fd595390764   elasticsearch:8.6.0           "/bin/tini -- /usr/l…"   5 months ago   Exited (143) 5 months ago
                                                                   es
72902ed215b1   wurstmeister/kafka            "start-kafka.sh"         5 months ago   Exited (255) 5 months ago   0.0.0.0:9092->9092/tcp, :::9092->9092/tcp                                 kafka
63bb49f02abe   nacos/nacos-server            "bin/docker-startup.…"   5 months ago   Exited (143) 5 months ago
                                                                   nacos
fb8e5f38e401   mysql:8                       "docker-entrypoint.s…"   5 months ago   Exited (255) 12 days ago    0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp                      mysql
6bc0f4671333   redis                         "docker-entrypoint.s…"   5 months ago   Exited (0) 2 months ago
                                   
```

这时，我们可以通过 `docker rm` 删除容器。

### 如何批量删除已经退出的容器

> [!note] 批量删除
> ```shell
> docker rm -v $(docker ps -aq -f status=exited)
> ```
> - `docker ps`：这是 Docker 的命令，用于列出容器。
> - `-a` 或 `--all`：这个选项告诉 `docker ps` 命令列出所有容器，而不仅仅是当前正在运行的容器。
>- `-q`：这个选项表示“静默模式”，它会只输出容器的 ID，而不输出其他详细信息。
>- `-f` 或 `--filter`：这个选项用于对列出的容器进行过滤。你可以指定不同的条件来过滤容器。
>- `status=exited`：这是一个过滤条件，表示只列出状态为“已退出”的容器。

> [!tip] 批量删除由某个镜像创建的容器
> ```shell
> docker ps -a -q -f ancestor=busybox | xargs docker rm
>```
## 4.5 State Machine

![](Blog/Docker/assets/Pasted%20image%2020250301180821.png)

> [!tip] 一些结论
> - 启动进程**正常退出或发生OOM**，此时Docker会根据 `--restart`的策略判断是否需要重启容器。但如果容器是因为执行`docker stop`或`docker kill`退出，则不会自动重启。

## 4.6 资源限制

### 如何限制容器可用的内存？

> [!note] 容器可使用的内存
> - 物理内存
> - swap

> [!note] 控制容器内存的使用量的参数
> - `-m`或` --memory`：设置内存的使用限额，例如100MB,2GB。
> - `--memory-swap`：设置内存+swap的使用限额。
> - 默认情况下，上面两组参数为 -1，即对容器内存和swap的使用没有限制。
> - 如果在启动容器时只指定 -m而不指定 --memory-swap，那么 --memory-swap默认为 -m的两倍

> [!example] 使用progrium/stress镜像来学习如何为容器分配内存
> 执行如下命令：
> ```shell
> docker run -it -m 200M --memory-swap=300M progrium/stress --vm 1 --vm-bytes 280M
>```
>- `--vm 1` 启动1个内存工作线程
>- `-- vm-bytes 280M` 每个线程分配280MB内存

### 如何限制容器的CPU资源？

> [!note] 控制容器CPU的资源分配的参数
> - 默认设置下，所有容器可以平等地使用host CPU资源并且没有限制。
> - Docker可以通过 `-c` 或 `--cpu-shares` 设置容器**使用CPU的权重**。如果不指定，默认值为1024。
> - 需要特别注意的是，这种按权重分配CPU只会发生在**CPU资源紧张**的情况下。如果containerA处于空闲状态，这时，为了充分利用CPU资源，containerB也可以分配到全部可用的CPU。
> - `--cpu`用来设置工作线程的数量。如果host有多颗CPU，则需要相应增加 `--cpu` 的数量。

> [!tip] 使用`top`命令查看进程状态，包括CPU使用情况、内存使用情况等

### 如何限制容器的Block IO带宽？

> [!note] Block IO是另一种可以限制容器使用的资源。Block IO指的是磁盘的读写，docker可通过设置权重、限制bps和iops的方式控制容器读写磁盘的带宽

#### Block IO 权重

> [!note] 控制容器IO权重
> 默认情况下，所有容器能平等地读写磁盘，可以通过设置 `--blkio-weight` 参数来改变容器block IO的优先级。参数默认值为500.

#### bps与iops限制

> [!note] bps与iops
> - bps是byte per second，每秒读写的数据量。
> - iops是io per second，每秒IO的次数。

> [!note] 控制bps与iops的参数
> - `--device-read-bps`：限制读某个设备的bps。
> - `--device-write-bps`：限制写某个设备的bps。
> - `--device-read-iops`：限制读某个设备的iops。
> - `--device-write-iops`：限制写某个设备的iops。

> [!example] 限制容器写 /dev/sda 的速率
> ```shell
> docker run -it --device-write-bps /dev/sda:30MB ubuntu
> # in container
> time dd if=/dev/zero of=test.out bs=1M count=800 oflag=direct # 测试写磁盘速度
>```
>- `time`：这是一个命令，用于测量另一个命令的执行时间。它会输出命令的实时、用户时间和系统时间。
> - `dd`：这是一个用于转换和复制文件的 Unix 命令，常用于测试磁盘性能。
> - `if=/dev/zero`：`if` 表示输入文件，`/dev/zero` 是一个特殊文件，读取它会不断返回零值。
> - `of=test.out`：`of` 表示输出文件，这里是 `test.out`。
> - `bs=1M`：`bs` 表示块大小，这里是 1MB。
> - `count=800`：表示复制 800 个块。
> - `oflag=direct`：这个选项告诉 `dd` 直接写入文件，绕过文件系统的缓冲区。这样可以更准确地测试磁盘的写入速度，因为它避免了缓存的影响。（采用 direct IO，`--device-write-bps` 才能生效。）

## 4.7 实现容器的底层技术

### 什么是cgroup？

> [!summary] cgroup全称Control Group。Linux操作系统通过cgroup可以设置进程使用CPU、内存和IO资源的限额。

#### cgroup存放在哪里？长什么样？

```shell
lee@lazybear:/sys/fs/cgroup$ ls
blkio  cpu,cpuacct  cpuset   freezer  memory  net_cls           net_prio    pids  systemd
cpu    cpuacct      devices  hugetlb  misc    net_cls,net_prio  perf_event  rdma  unified
lee@lazybear:/sys/fs/cgroup$ pwd
/sys/fs/cgroup
```

> [!note] 我们可以在 `/sys/fs/cgroup` 目录下找到cgroup的内容。
而当我们进入具体的文件夹，可以发现，Linux为每个容器都创建了一个cgroup目录，这个目录则是以容器的长ID命名的。

```shell
lee@lazybear:/sys/fs/cgroup/cpu/docker$ ls
2fa1e460ac8a4c62676a6be9b5b838e2c6e24e3859ec6dd4c8c1e9e00f4a6afb  cpu.cfs_period_us  cpu.rt_runtime_us  tasks
cgroup.clone_children                                             cpu.cfs_quota_us   cpu.shares
cgroup.procs                                                      cpu.idle           cpu.stat
cpu.cfs_burst_us                                                  cpu.rt_period_us   notify_on_release
lee@lazybear:/sys/fs/cgroup/cpu/docker$ cd 2fa1e460ac8a4c62676a6be9b5b838e2c6e24e3859ec6dd4c8c1e9e00f4a6afb
lee@lazybear:/sys/fs/cgroup/cpu/docker/2fa1e460ac8a4c62676a6be9b5b838e2c6e24e3859ec6dd4c8c1e9e00f4a6afb$ ls
cgroup.clone_children  cpu.cfs_burst_us   cpu.cfs_quota_us  cpu.rt_period_us   cpu.shares  notify_on_release
cgroup.procs           cpu.cfs_period_us  cpu.idle          cpu.rt_runtime_us  cpu.stat    tasks
```

> [!note] 在上面的目录中，包含了所有与cpu相关的cgroup配置，例如 `cpu.shares` 保存的就是对应参数 `--cpu-shares` 的配置。

```shell
lee@lazybear:/sys/fs/cgroup/cpu/docker/2fa1e460ac8a4c62676a6be9b5b838e2c6e24e3859ec6dd4c8c1e9e00f4a6afb$ cat cpu.shares
1024
```

> [!note] 类似的，memory 和 blkio 目录下保存的就是内存和IO的配置信息。

### 什么是namespace？

> [!summary] namespace管理着host中全局唯一的资源，并可以让每个容器都觉得只有自己在使用它。换句话说，**namespace实现了容器间资源的隔离**。Linux使用了6种namespace，分别对应6种资源：Mount、UTS、IPC、PID、Network和User

#### Mount namespace

> [!note] Mount namespace让容器看上去拥有整个文件系统。
>容器有自己的/目录，可以执行mount和umount命令。当然我们知道这些操作只在当前容器中生效，不会影响到host和其他容器。

#### UTS namespace

> [!note] UTS namespace让容器有自己的hostname。
> 默认情况下，容器的hostname是它的短ID，可以通过 -h或 --hostname参数设置。
> `docker run -h hostname`

#### IPC namespace

> [!note] IPC namespace让容器拥有自己的共享内存和信号量（semaphore）来实现进程间通信，而不会与host和其他容器的IPC混在一起。
> 

#### PID namespace

> [!tip] 我们可以通过 `ps axf` 来查看进程信息。
> 

> [!note] 我们会发现，所有容器的进程都挂在dockerd进程下，同时也可以看到容器自己的子进程。如果我们进入到某个容器，`ps` 就只能看到自己的进程了。
> 而且进程的PID不同于host中对应进程的PID，容器中PID=1的进程当然也不是host的init进程。也就是说：容器拥有自己独立的一套PID，这就是PID namespace提供的功能。

#### Network namespace

> [!summary] Network namespace让容器拥有自己独立的网卡、IP、路由等资源
> 

#### User namespace

> [!summary] User namespace让容器能够管理自己的用户，host不能看到容器中创建的用户
> 
