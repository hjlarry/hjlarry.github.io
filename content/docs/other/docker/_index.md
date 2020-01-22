# Docker概要

镜像
-------

操作系统分为内核空间和用户空间。对于Linux，内核启动后，会挂载root文件系统为其提供用户空间支持。而Docker中的镜像(Image)，就相当于是一个root文件系统，只是它有些特殊，除了提供容器运行时所需的程序、库、资源、配置等文件外，还会包含一些为运行时准备的配置参数(如匿名卷、环境变量、用户等)。镜像不包含任何动态数据，其内容在构建之后不会改变。

Docker使用[Union FS](https://en.wikipedia.org/wiki/Union_mount)技术，将镜像设计为分层存储的架构。构建时，会一层层构建，前一层是后一层的基础，每层构建完就不会改变，后一层的任何改变只是发生在自己这一层。例如，删除前一层的文件并不是真正的删除，而只是在本层中标记上一层的文件是删除的。

### 基本操作
#### 获取镜像
使用如下命令可以从镜像仓库中拉取镜像:
```
docker pull [选项] [镜像仓库的地址[:端口号]/]仓库名[:标签]
```
镜像仓库的地址默认为Docker Hub的地址，对于Docker Hub，仓库名默认为library，所以我们往往可以简写为`docker pull ubuntu:18.04`。

#### 列出镜像
我们可以这样列出全部的镜像:
{{< highlight sh>}}
PS C:\Users\hejl> docker image ls
REPOSITORY                                       TAG                 IMAGE ID            CREATED             SIZE
hjlarry/cheers2019                               latest              f0c36061dc59        3 hours ago         4.01MB
<none>                                           <none>              1487cd0b23aa        3 hours ago         356MB
ubuntu                                           18.04               ccc6e87d482b        4 days ago          64.2MB
ubuntu                                           latest              ccc6e87d482b        4 days ago          64.2MB
golang                                           1.11-alpine         e116d2efa2ab        5 months ago        312MB
gcr.azk8s.cn/google_containers/hyperkube-amd64   v1.9.2              583687acb4de        2 years ago         618MB
{{< /highlight >}}
我们可以看到镜像ID是镜像的唯一标识，但标签可以有多个，例如ubuntu的18.04和latest是同一个镜像。

镜像的体积可能会比Docker Hub中显示的要大一些，因为Docker Hub进行了一定的压缩。实际这些镜像占用的总空间也不是把他们加起来就能算出来，因为共同的层可以复用。镜像占用的总大小可以通过`docker system df`看到。

有一种特殊的镜像，镜像名和标签均为<none>，被称为虚悬镜像(dangling image)。它们没有什么用，可以通过`docker image prune`一键删除。

还有一种镜像叫中间层镜像，用来加速构建、重复利用资源，有的中间层镜像也没有标签和名称，它们不能被删除。可以通过`docker image ls -a`观察到包含中间层镜像的所有镜像。

#### 删除镜像
使用`docker image rm <镜像1> [<镜像2> ...]`命令可删除镜像。这里既可以用镜像的名称，也就是`<仓库名>:<标签>`来表示镜像；也可以用镜像的ID，一般取前3位，足以区分出别的镜像即可。

在删除镜像时，我们往往会看到删除信息中既有`Untagged`，也有`Deleted`。`Untagged`实际上是因为我们要删除的是某个tag标签下的镜像，需要去取消标签。

可以使用多个命令配合来批量删除一些镜像，例如`docker image rm $(docker image ls -q redis)`可以删除所有仓库名为redis的镜像，`docker image rm $(docker image ls -q -f before=mongo:3.2)`可以删除所有mongo版本3.2之前的镜像。

### 定制镜像
镜像的定制实际上就是定制每层所添加的配置和文件，我们可以把每一层的修改、安装、构建、操作的命令都写入一个Dockerfile，通过它来构建、定制镜像。

#### FROM
用来指定基础镜像。定制镜像一定是以一个镜像为基础，在其上进行定制，FROM是必备的且必须是第一条指令。

我们一般可以用一些服务类的镜像作为基础镜像，例如nginx、redis、python、golang等，也可以使用更为基础的操作系统镜像，如ubuntu、centos、alpine等。还以为用scratch作为基础镜像，它表示一个空白的镜像，接下来所写的指令为镜像的第一层，因为有一些静态编译的程序并不需要操作系统提供运行时支持。

#### RUN
用来执行命令行命令的，可以像shell脚本一样使用，但我们不应该把每个命令都去对应一个RUN，例如:
{{< highlight dockerfile>}}
FROM debian:stretch

RUN apt-get update
RUN apt-get install -y gcc libc6-dev make wget
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz"
RUN mkdir -p /usr/src/redis
RUN tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1
RUN make -C /usr/src/redis
RUN make -C /usr/src/redis install
{{< /highlight >}}
这样构建的镜像非常臃肿，添加了很多运行时不需要的东西，增加了构建部署的时间，也容易出错。应该这样写:
{{< highlight dockerfile>}}
FROM debian:stretch

RUN buildDeps='gcc libc6-dev make wget' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
{{< /highlight >}}
所有的命令都是一个目的，即编译、安装redis可执行文件，没必要多层。此外，这组命令的最后添加了清理工作的命令，删除为了编译构建所需的文件，清理了所有下载、展开的文件，还清理了apt的缓存文件。镜像构建时，一定要确保每一层只添加真正需要添加的东西，任何无关的东西都应该清理掉。

### 构建镜像
使用`docker build [选项] <上下文路径/URL/->`命令可进行镜像构建。

#### 上下文
我们经常使用`docker build .`，往往会理解为`.`表示当前目录，下意识的认为这是dockerfile的所在路径，但这么理解是不准确的，实际上这是在指定上下文路径。

那么什么是上下文呢？Docker在运行时分为服务端守护进程和客户端工具，我们输入docker相关命令都是客户端操作，通过[Docker Remote API](https://docs.docker.com/develop/sdk/)与服务端docker引擎交互。当我们进行镜像构建时，就是通过`docker build`在服务端进行构建，上下文路径中的内容会被发送过去，这个`.`就是在指定上下文路径，这样当在dockerfile中遇到文件复制这样的语句，例如`COPY ./package.json /app/`时并不是要复制dockerfile下的package.json，而是去复制上下文路径下的package.json。所以类似`COPY ../package.json /app/`或者`COPY /opt/*** /app`都超出了上下文路径而无效，如果需要它们的话就得把它们复制到上下文目录中去。

dockerfile一般放在一个空目录或项目的根目录下，如果有东西不希望发送给docker引擎，可以使用`.dockerignore`剔除掉。默认情况下，会将上下文目录下的名为`Dockerfile`的文件作为Dockerfile，实际上可以通过如`-f ../Dockerfile.php`指定某个文件为Dockerfile。

#### 其他构建方式
还可以直接使用git repo的方式构建:
{{< highlight sh>}}
PS C:\Users\hejl>  docker build https://github.com/twang2218/gitlab-ce-zh.git#:11.1
Sending build context to Docker daemon  6.144kB
Step 1/23 : FROM gitlab/gitlab-ce:11.1.4-ce.0 as builder
11.1.4-ce.0: Pulling from gitlab/gitlab-ce
8ee29e426c26: Pull complete                                                                    6e83b260b73b: Pull complete                                                                    e26b65fd1143: Pull complete      
...
{{< /highlight >}}
该命令指定了构建所需的git repo，指定了默认的master分支，构建目录为`/11.1/`，然后docker会自己去clone项目切换到指定分支，并进入指定目录进行构建。

也可以通过tar压缩包构建，例如`docker build http://url/context.tar.gz`，会去下载tar压缩包并自动解压缩，以其为上下文，开始构建。