---
title: Docker镜像
date: 2024-09-15 16:23
tags:
  - Note
  - Docker
---
> [!summary] Docker镜像
> 本文档主要介绍了Docker镜像的内部结构、创建方法、分层结构、构建过程以及如何分发镜像。
> 首先，探讨了**如何通过`docker pull hello-world`命令拉取基础镜像，并解释了Dockerfile的作用和base镜像**的概念。
> 接着，解释了**Docker镜像的分层结构和容器层的工作方式**，包括Copy-on-Write特性。
> 然后，介绍了**两种构建Docker镜像的方法**：使用`docker commit`命令和编写Dockerfile。
> 文档还展示了如何使用`docker history`命令查看镜像的构建历史，并讨论了Docker镜像缓存机制及其影响。
> 此外，提供了**Dockerfile中常用指令的说明**，并区分了`RUN`、`CMD`和`ENTRYPOINT`指令的用途和用法。
> 最后，讨论了**如何为镜像命名、打tag以及如何使用公共和私有Registry分发镜像**，包括在Docker Hub上存取镜像和搭建本地Registry的方法。

## 3.1 镜像的内部结构

### 怎么创建一个镜像？——容器界的hello world

```shell
docker pull hello-world
```

在我们拉取之后，可以通过 `docker images` 命令查看镜像信息。

那么，这个镜像里面包含了哪些内容呢？

> [!note] Dockerfile
> 每个镜像都对应着一个自己的镜像描述文件，定义了如何构建Docker镜像。
> 而 hello-world 的Dockerfile内容则如下所示：
> ```Dockerfile
> FROM scratch
> COPY hello /
> CMD ["/hello"]
> ```

### 什么是 base 镜像？

> [!note] base 镜像
> - 不依赖其他镜像，从 scratch 构建
> - 其他镜像可以以它为基础进行扩展
> 而一般被称作 base 镜像的，都是 Linux 发行版的 Docker 镜像

#### 为什么拉取一个 Linux 发行版镜像才几百MB？

![](Blog/Docker/assets/Pasted%20image%2020250301180759.png)

> [!note]
> - Linux操作系统由内核空间和用户空间组成
> - 内核空间为 kernel ， 刚启动时加载 bootfs 文件系统，然后 bootfs 会被卸载；而用户空间的文件系统则为 rootfs ，包含常见的 /dev、/proc、/bin 等等。
> - 对于 base 镜像，其只需要使用 Host 的 Kernel，镜像只提供 rootfs 。

> [!quote]- CentOS 7 Dockerfile
> ```Dockerfile
> FROM scratch
> ADD centos-7-docker.tar.xz /
> CMD ["/bin/bash"]
>```

> [!attention] base镜像只是在用户空间与发行版一致，kernel版本与发行版是不同的
> 

### 镜像的分层结构

 ![](Blog/Docker/assets/Pasted%20image%2020250301180808.png)
> [!note] 从base镜像上一层层叠加，每进行一步操作/安装一个软件，就在现有镜像上增加一层

#### 为什么采用这种分层结构？

> [!hint] 共享资源
> 有多个镜像都从相同的base镜像构建而来，那么Docker Host只需在磁盘上保存一份base镜像；同时内存中也只需加载一份base镜像，就可以为所有容器服务了，而且镜像的每一层都可以被共享。

#### 可写的容器层

> [!note] 当容器启动时，一个新的可写层被加载到镜像的顶部。这一层通常被称作“容器层”​，​“容器层”之下的都叫“镜像层”​。所有对容器的改动，无论添加、删除，还是修改文件都只会发生在容器层中。只有容器层是可写的，容器层下面的所有镜像层都是只读的。
> 

> [!note] 容器层
> 文件操作细节：
> （1）**添加文件**。在容器中创建文件时，新文件被添加到容器层中。
> （2）**读取文件**。在容器中读取某个文件时，Docker会从上往下依次在各镜像层中查找此文件。一旦找到，打开并读入内存。
> （3）**修改文件**。在容器中修改已存在的文件时，Docker会从上往下依次在各镜像层中查找此文件。一旦找到，立即将其复制到容器层，然后修改之。
> （4）**删除文件**。在容器中删除文件时，Docker也是从上往下依次在镜像层中查找此文件。找到后，会在容器层中记录下此删除操作。
> 只有当需要修改时才复制一份数据，这种特性被称作 **Copy-on-Write** 。可见，容器层保存的是镜像变化的部分，不会对镜像本身进行任何修改。

## 3.2 构建镜像

> [!note] Docker提供的两种构建镜像的方法
> 1. docker commit命令
> 2. Dockerfile构建文件

### docker commit

> [!note] 构建三个步骤
> - 运行容器
> - 修改容器
> - 将容器保存为新的镜像

> [!example] 在 Ubuntu 镜像中安装 vim 并保存为新镜像
> ```shell
> docker run -it ubuntu # -it 以交互模式进入容器
> # 进入容器
> vim # 检查是否安装vim
> apt-get install -y vim # 安装vim
> # 退出容器
> docker commit old-name new-name
> ```

### Dockerfile

> [!note] Dockerfile
> 一个文本文件，记录了镜像构建的所有步骤。

> [!example] 使用 Dockerfile 完成在 Ubuntu 镜像中安装 vim 并保存为新镜像
> - Dockerfile
> 	```Dockerfile
> 	FROM ubuntu
> 	RUN apt-get update && apt-get install -y vim
> 	``` 
> - docker build
> 	```shell
> 	docker build -t ubuntu-with-vi-dockerfile .
> 	```

> [!note] docker build
> 运行 docker build 流程
> - `-t` 将新镜像命名为 `ubuntu-with-vi-dockerfile`，命令末尾的 `.` 指明 build context 为当前目录。Docker 默认会从 build context 中查找 Dockerfile 文件，我们也可以通过 `-f` 参数指定Dockerfile的位置。
> - 首先 Docker 将 build context 中的所有文件发送给 Docker daemon。build context为镜像构建提供所需要的文件或目录。Dockerfile 中的 `ADD`、`COPY` 等命令可以将build context中的文件添加到镜像
> - 执行FROM，将Ubuntu作为base镜像
> - 启动ID为xxx的临时容器，在容器中通过apt-get安装vim。
> - 安装成功后，将容器保存为镜像
> - 删除临时容器
> - 镜像构建成功

#### 如何查看镜像分层结构？

> [!note] docker history
> `docker history` 会显示镜像的构建历史，也就是Dockerfile的执行过程。

```shell
xxx@VM-20-6-ubuntu:~$ sudo docker history mysql:8.0.21
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
8e85dd5c3255   3 years ago   /bin/sh -c #(nop)  CMD ["mysqld"]               0B
<missing>      3 years ago   /bin/sh -c #(nop)  EXPOSE 3306 33060            0B
<missing>      3 years ago   /bin/sh -c #(nop)  ENTRYPOINT ["docker-entry…   0B
<missing>      3 years ago   /bin/sh -c ln -s usr/local/bin/docker-entryp…   34B
<missing>      3 years ago   /bin/sh -c #(nop) COPY file:7cbb26bbdb8e71b3…   13.2kB
<missing>      3 years ago   /bin/sh -c #(nop) COPY dir:2e040acc386ebd23b…   1.12kB
<missing>      3 years ago   /bin/sh -c #(nop)  VOLUME [/var/lib/mysql]      0B
<missing>      3 years ago   /bin/sh -c {   echo mysql-community-server m…   409MB
<missing>      3 years ago   /bin/sh -c echo "deb http://repo.mysql.com/a…   55B
<missing>      3 years ago   /bin/sh -c #(nop)  ENV MYSQL_VERSION=8.0.21-…   0B
<missing>      3 years ago   /bin/sh -c #(nop)  ENV MYSQL_MAJOR=8.0          0B
<missing>      3 years ago   /bin/sh -c set -ex;  key='A4A9406876FCBD3C45…   2.61kB
<missing>      3 years ago   /bin/sh -c apt-get update && apt-get install…   52.2MB
<missing>      3 years ago   /bin/sh -c mkdir /docker-entrypoint-initdb.d    0B
<missing>      3 years ago   /bin/sh -c set -eux;  savedAptMark="$(apt-ma…   4.17MB
<missing>      3 years ago   /bin/sh -c #(nop)  ENV GOSU_VERSION=1.12        0B
<missing>      3 years ago   /bin/sh -c apt-get update && apt-get install…   9.34MB
<missing>      3 years ago   /bin/sh -c groupadd -r mysql && useradd -r -…   329kB
<missing>      3 years ago   /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>      3 years ago   /bin/sh -c #(nop) ADD file:0dc53e7886c35bc21…   69.2MB
```

以上就是一个示例，来自 `mysql:8.0.21` 镜像。

> [!tip] 关于missing
> missing表示无法获取IMAGE ID，通常从Docker Hub下载的镜像会有这个问题。

#### 镜像需要缓存吗？是如何缓存的？

> [!note] Docker镜像的缓存特性
> Docker会缓存已有镜像的镜像层，构建新镜像时，如果某镜像层已经存在，就直接使用，无须重新创建。

##### 如果不希望使用缓存呢？

> [!tip]
> 如果我们希望在构建镜像时不使用缓存，可以在 `docker build` 命令中加上`--no-cache`参数。

> [!attention] 缓存失效
> Dockerfile中每一个指令都会创建一个镜像层，上层是依赖于下层的。无论什么时候，**只要某一层发生变化，其上面所有层的缓存都会失效**。也就是说，如果我们改变Dockerfile指令的执行顺序，或者修改或添加指令，都会使缓存失效。虽然在逻辑上这种改动对镜像的内容没有影响，但由于**分层的结构特性**，Docker必须重建受影响的镜像层。

> [!note]
> `docker pull` 同样会使用镜像缓存。

#### Dockerfile 常用指令

- `FROM` 指定base镜像
- `MAINTAINER` 设置镜像的作者，可以是任意字符串
- `COPY` 将文件从build context复制到镜像。
- `ADD` 与`COPY`类似，从build context复制文件到镜像。不同的是，如果`src`是归档文件（tar、zip、tgz、xz等）​，文件会被自动解压到dest。
- `ENV` 设置环境变量，环境变量可被后面的指令使用。
- `EXPOSE` 指定容器中的进程会监听某个端口，Docker可以将该端口暴露出来。
- `VOLUME` 将文件或目录声明为volume。
- `WORKDIR` 为后面的`RUN`、`CMD`、`ENTRYPOINT`、`ADD`或`COPY`指令设置镜像中的当前工作目录。
- `RUN` 在容器中运行指定的命令。
- `CMD` 容器启动时运行指定的命令。（只会有最后一个生效）
- `ENTRYPOINT` 设置容器启动时运行的命令。Dockerfile中可以有多个 `ENTRYPOINT` 指令，但只有最后一个生效。`CMD`或`docker run`之后的参数会被当作参数传递给`ENTRYPOINT`。

## 3.3 RUN vs CMD vs ENTRYPOINT

> [!note] 三者的区别
> 1. RUN：执行命令并创建新的镜像层，RUN经常用于安装软件包。
> 2. CMD：设置容器启动后默认执行的命令及其参数，但CMD能够被docker run后面跟的命令行参数替换。
> 3. ENTRYPOINT：配置容器启动时运行的命令。

### 如何指定命令？采用什么格式？

> [!note] Shell格式
> 当指令执行时，shell格式底层会调用 `/bin/sh -c [command]`

```shell
<instruction> <command>
```

> [!note] Exec格式
> 当指令执行时，会直接调用 [command]​，不会被shell解析。

```shell
<instruction> ["executable", "param1", "param2", ...]
```

```shell
ENV name Cloud Man ENTRYPOINT ["/bin/echo", "Hello, $name"]
```

> [!tip]
> CMD和ENTRYPOINT推荐使用Exec格式，因为指令可读性更强，更容易理解。RUN则两种格式都可以。

### RUN

> [!note]
> RUN指令通常用于安装应用和软件包。RUN在当前镜像的顶部执行命令，并创建新的镜像层。Dockerfile中常常包含多个RUN指令。

> [!attention] 关于 RUN 的实践
> 我们可以将多个指令放在一个RUN指令中执行，这样可以只创建一个镜像层，同时还能避免以下情况：
> apt-get update和apt-get install被放在一个RUN指令中执行，这样能够保证每次安装的是最新的包。如果apt-get install在单独的RUN中执行，则会使用apt-get update创建镜像层，而这一层可能是很久以前缓存的。

### CMD

> [!note]
> CMD指令允许用户指定容器的默认执行的命令。
> 不过，如果 docker run 指定其他命令，那么会被覆盖，同时，多个 CMD 指令只会执行最后一个。

> [!tip] 可以使用 CMD 为 ENTRYPOINT 设置默认的参数，不过二者格式都需要采用 Exec。

### ENTRYPOINT

> [!note]
> ENTRYPOINT指令可让容器以应用程序或者服务的形式运行。
> ENTRYPOINT不会被忽略，一定会被执行，即使运行docker run时指定了其他命令！

### 最佳实践

> [!summary] 如何使用这些指令？
> 1. 使用RUN指令安装应用和软件包，构建镜像。
> 2. 如果Docker镜像的用途是运行应用程序或服务，比如运行一个MySQL，应该优先使用Exec格式的ENTRYPOINT指令。**CMD可为ENTRYPOINT提供额外的默认参数，同时可利用docker run命令行替换默认参数**。
> 3. 如果想为容器设置默认的启动命令，可使用CMD指令。用户可在docker run命令行中替换此默认命令。

## 3.4 分发镜像

> [!info]
> 本节主要讨论如何使用公共和私有Registry分发镜像。

### 如何为镜像命名？

前文提到，在 `build` 的时候采用 `-t` 参数来命名。

实际上，当我们采用 `docker images`  查看镜像信息的时候会发现：

```shell
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
mysql        8         750b67184e7a   7 weeks ago   584MB
mysql        8.0.21    8e85dd5c3255   3 years ago   544MB
```

实际上一个镜像的名字由两部分组成：

`[image name] = [repository]:[tag]`

当我们执行 `build` 没有指定 `tag` ，那么就会使用默认值 `latest`。

#### tag实践

> [!note] tag命名
> 每个repository可以有多个tag，而多个tag可能对应的是同一个镜像。
> 我们可以通过docker tag命令方便地给镜像打tag。
> 而打tag的方式则采用了多级分支的思想。

```shell
docker tag myimage-v1.9.1 myimage:1 
docker tag myimage-v1.9.1 myimage:1.9 
docker tag myimage-v1.9.1 myimage:1.9.1 
docker tag myimage- v1.9.1 myimage:latest
```

### 如何使用公共Registry？

#### 如何使用Docker Hub存取我们的镜像

- 在Docker Hub注册账号
- 使用 `docker login -u xxx`命令登陆
- 修改镜像的repository
- 通过`docker tag`命令重命名镜像
- 通过`docker push`将镜像上传到Docker Hub

### 如何搭建本地Registry？

> [!note] 启动Registry容器
> - 拉取官方镜像`registry`
> - 运行镜像容器 `docker run -d -p 5000:5000 -v /xx:/var/lib/registry registry`
> - 通过`docker tag`重命名镜像，使之与registry匹配 `docker tag username/xx:v1 registry.example.net:5000/username/xx:v1`

## 3.5 其他补充

### 如何删除本地的镜像？

> [!note] rmi
> - rmi只能删除host上的镜像
> - 如果一个镜像对应多个tag，只有当最后一个tag被删除，镜像才被真正删除

### 如何搜索 docker hub 中的镜像？

> [!note] search
> `docker search xxx`
