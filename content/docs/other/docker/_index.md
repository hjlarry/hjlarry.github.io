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