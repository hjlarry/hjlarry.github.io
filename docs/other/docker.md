# Docker概要

镜像
-------

操作系统分为内核空间和用户空间。对于Linux，内核启动后，会挂载root文件系统为其提供用户空间支持。而Docker中的镜像(Image)，就相当于是一个root文件系统，只是它有些特殊，除了提供容器运行时所需的程序、库、资源、配置等文件外，还会包含一些为运行时准备的配置参数(如匿名卷、环境变量、用户等)。镜像不包含任何动态数据，其内容在构建之后不会改变。

Docker使用[Union FS](https://en.wikipedia.org/wiki/Union_mount)技术，将镜像设计为分层存储的架构。构建时，会一层层构建，前一层是后一层的基础，每层构建完就不会改变，后一层的任何改变只是发生在自己这一层。例如，删除前一层的文件并不是真正的删除，而只是在本层中标记上一层的文件是删除的。

### 获取镜像
使用如下命令可以从镜像仓库中拉取镜像:
```
docker pull [选项] [镜像仓库的地址[:端口号]/]仓库名[:标签]
```
镜像仓库的地址默认为Docker Hub的地址，对于Docker Hub，仓库名默认为library，所以我们往往可以简写为`docker pull ubuntu:18.04`。

### 列出镜像
我们可以这样列出全部的镜像:
```powershell
PS C:\Users\hejl> docker image ls
REPOSITORY                                       TAG                 IMAGE ID            CREATED             SIZE
hjlarry/cheers2019                               latest              f0c36061dc59        3 hours ago         4.01MB
<none>                                           <none>              1487cd0b23aa        3 hours ago         356MB
ubuntu                                           18.04               ccc6e87d482b        4 days ago          64.2MB
ubuntu                                           latest              ccc6e87d482b        4 days ago          64.2MB
golang                                           1.11-alpine         e116d2efa2ab        5 months ago        312MB
gcr.azk8s.cn/google_containers/hyperkube-amd64   v1.9.2              583687acb4de        2 years ago         618MB
```
我们可以看到镜像ID是镜像的唯一标识，但标签可以有多个，例如ubuntu的18.04和latest是同一个镜像。

镜像的体积可能会比Docker Hub中显示的要大一些，因为Docker Hub进行了一定的压缩。实际这些镜像占用的总空间也不是把他们加起来就能算出来，因为共同的层可以复用。镜像占用的总大小可以通过`docker system df`看到。

有一种特殊的镜像，镜像名和标签均为<none>，被称为虚悬镜像(dangling image)。它们没有什么用，可以通过`docker image prune`一键删除。

还有一种镜像叫中间层镜像，用来加速构建、重复利用资源，有的中间层镜像也没有标签和名称，它们不能被删除。可以通过`docker image ls -a`观察到包含中间层镜像的所有镜像。

### 删除镜像
使用`docker image rm <镜像1> [<镜像2> ...]`命令可删除镜像。这里既可以用镜像的名称，也就是`<仓库名>:<标签>`来表示镜像；也可以用镜像的ID，一般取前3位，足以区分出别的镜像即可。

在删除镜像时，我们往往会看到删除信息中既有`Untagged`，也有`Deleted`。`Untagged`实际上是因为我们要删除的是某个tag标签下的镜像，需要去取消标签。

可以使用多个命令配合来批量删除一些镜像，例如`docker image rm $(docker image ls -q redis)`可以删除所有仓库名为redis的镜像，`docker image rm $(docker image ls -q -f before=mongo:3.2)`可以删除所有mongo版本3.2之前的镜像。

### 定制镜像
镜像的定制实际上就是定制每层所添加的配置和文件，我们可以把每一层的修改、安装、构建、操作的命令都写入一个Dockerfile，通过它来构建、定制镜像。

**FROM**

用来指定基础镜像。定制镜像一定是以一个镜像为基础，在其上进行定制，FROM是必备的且必须是第一条指令。

我们一般可以用一些服务类的镜像作为基础镜像，例如nginx、redis、python、golang等，也可以使用更为基础的操作系统镜像，如ubuntu、centos、alpine等。还以为用scratch作为基础镜像，它表示一个空白的镜像，接下来所写的指令为镜像的第一层，因为有一些静态编译的程序并不需要操作系统提供运行时支持。

**RUN**

用来执行命令行命令的，可以像shell脚本一样使用，但我们不应该把每个命令都去对应一个RUN，例如:
```dockerfile
FROM debian:stretch

RUN apt-get update
RUN apt-get install -y gcc libc6-dev make wget
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz"
RUN mkdir -p /usr/src/redis
RUN tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1
RUN make -C /usr/src/redis
RUN make -C /usr/src/redis install
```
这样构建的镜像非常臃肿，添加了很多运行时不需要的东西，增加了构建部署的时间，也容易出错。应该这样写:
```dockerfile
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
```
所有的命令都是一个目的，即编译、安装redis可执行文件，没必要多层。此外，这组命令的最后添加了清理工作的命令，删除为了编译构建所需的文件，清理了所有下载、展开的文件，还清理了apt的缓存文件。镜像构建时，一定要确保每一层只添加真正需要添加的东西，任何无关的东西都应该清理掉。

**COPY**

该指令从构建上下文的目录中复制文件到镜像内的目标路径位置，源路径可以是多个，也可以用通配符，通配符规则是Go的[filepath.Match](https://golang.org/pkg/path/filepath/#Match)规则。还可以通过添加`--chown=<user>:<group>`选项来改变文件的所属用户和组。例如:
```
COPY hom* /mydir/
COPY hom?.txt /mydir/
COPY --chown=55:mygroup files* /mydir/
COPY --chown=bin files* /mydir/
```

**ADD**

该命令和COPY的格式和性质基本一致，按最佳实践，COPY的语义更加明确应尽可能的使用，尽在需要自动解压缩的场合使用ADD。

**CMD**

因为容器是进程，那么在启动容器的时候，就需要指定所运行的程序及参数，CMD就用来指定默认的容器主进程的启动命令。比如，ubuntu镜像的CMD设置的是/bin/bash，所以我们运行容器`docker run -it ubuntu`会进入到bash，但当我们使用`docker run -it ubuntu cat /etc/os-release`运行容器时就用`cat /etc/os-release`替换掉了`/bin/bash`命令。

该指令既支持shell格式，`CMD <命令>`，也支持exec格式，`CMD ["可执行文件", "参数1", "参数2"...]`。但是shell格式会被包装一个`sh -c`的参数。例如`CMD echo $HOME`在实际的执行时会被替换为`CMD [ "sh", "-c", "echo $HOME" ]`。

另外，docker中的应用都是以前台执行的，而不是像虚拟机、物理机那样，可以用systemd去启动一个后台服务，容器内没有后台服务的概念。类似于`CMD service nginx start`会发现容器执行后就立即退出了，甚至`systemctl`命令结果却根本执行不了，因为容器就是为了主进程而存在的，主进程退出容器就没有存在的意义也会退出，这条命令会被翻译为`CMD [ "sh", "-c", "service nginx start"]`，sh是主进程，当`service nginx start`命令结束后，sh就结束了，主进程退出自然容器就会退出。正确的做法是直接执行nginx可执行文件并以前台形式运行，如`CMD ["nginx", "-g", "daemon off;"]`。

**ENV**

用来设置环境变量，设置后无论是之后的指令，还是运行时的应用都可以直接使用定义的环境变量。支持格式:

* ENV <key> <value>
* ENV <key1>=<value1> <key2>=<value2> ...

例如:
```dockerfile
ENV VERSION=1.0 DEBUG=on NAME="Happy Feet"
RUN curl -SLO "https://nodejs.org/dist/v$VERSION/node-v$NAME-linux-x64.tar.xz"
```

**ARG**

和ENV类似，也是设置环境变量，只是它在容器运行时不会存在这些环境变量。

**VOLUME**

因为容器的运行应保持容器存储层不发生写操作，那么对于数据库类需要动态保存数据的应用，其数据库文件应保存于卷(volume)中。该指令用于挂载一个目录为匿名卷，如果容器运行时用户未指定匿名卷，就会用它做匿名卷，向其写入数据即可避免向容器存储层写入大量数据；如果用户指定，则会覆盖掉这个指令中的设置。

其格式为`VOLUME <路径>`或`VOLUME ["<路径1>", "<路径2>"...]`。用户指定命名卷可以这样写`docker run -d -v mydata:/data xxxx`，即把mydata这个命名卷挂载到了/data位置，会替代Dockerfile中定义的匿名卷挂载配置。

**EXPOSE**

只是一个声明，容器运行时提供的服务端口。容器不会因为这个声明就自动开启这个端口的服务，只是帮助镜像使用者理解，或是使用`docker run -P`做随机端口映射时可自动映射到该指令设置的端口。

**WORKDIR**

使用该指令可以指定当前目录(或者称为工作目录)，以后各层的当前目录就被设置为它，如该目录不存在则会自动创建该目录。

比如我们可能会这样写:
```dockerfile
RUN cd /app
RUN echo "hello" > world.txt
```

这样构建运行后，会发现找不到/app/world.txt这个文件，因为两个RUN代表两层，它们的执行环境不同，运行到第二层时启动的是一个全新的容器。这个时候就应该用`WORKDIR`指令进行设置。

**USER**

使用该指令可以切换到指定的用户，其后的每一层执行RUN、CMD以及ENTRYPOINT之类命令都会是这个新的身份。这个用户必须是事先建立好的。

### 构建镜像
使用`docker build [选项] <上下文路径/URL/->`命令可进行镜像构建。

#### 上下文
我们经常使用`docker build .`，往往会理解为`.`表示当前目录，下意识的认为这是dockerfile的所在路径，但这么理解是不准确的，实际上这是在指定上下文路径。

那么什么是上下文呢？Docker在运行时分为服务端守护进程和客户端工具，我们输入docker相关命令都是客户端操作，通过[Docker Remote API](https://docs.docker.com/develop/sdk/)与服务端docker引擎交互。当我们进行镜像构建时，就是通过`docker build`在服务端进行构建，上下文路径中的内容会被发送过去，这个`.`就是在指定上下文路径，这样当在dockerfile中遇到文件复制这样的语句，例如`COPY ./package.json /app/`时并不是要复制dockerfile下的package.json，而是去复制上下文路径下的package.json。所以类似`COPY ../package.json /app/`或者`COPY /opt/*** /app`都超出了上下文路径而无效，如果需要它们的话就得把它们复制到上下文目录中去。

dockerfile一般放在一个空目录或项目的根目录下，如果有东西不希望发送给docker引擎，可以使用`.dockerignore`剔除掉。默认情况下，会将上下文目录下的名为`Dockerfile`的文件作为Dockerfile，实际上可以通过如`-f ../Dockerfile.php`指定某个文件为Dockerfile。

#### 其他构建方式
还可以直接使用git repo的方式构建:
```powershell
PS C:\Users\hejl>  docker build https://github.com/twang2218/gitlab-ce-zh.git#:11.1
Sending build context to Docker daemon  6.144kB
Step 1/23 : FROM gitlab/gitlab-ce:11.1.4-ce.0 as builder
11.1.4-ce.0: Pulling from gitlab/gitlab-ce
8ee29e426c26: Pull complete                                                                    6e83b260b73b: Pull complete                                                                    e26b65fd1143: Pull complete      
...
```
该命令指定了构建所需的git repo，指定了默认的master分支，构建目录为`/11.1/`，然后docker会自己去clone项目切换到指定分支，并进入指定目录进行构建。

也可以通过tar压缩包构建，例如`docker build http://url/context.tar.gz`，会去下载tar压缩包并自动解压缩，以其为上下文，开始构建。


容器
-------
镜像是静态的定义，而容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

容器的实质是进程，但与直接在宿主执行的进程是不同的，容器进程运行于属于自己独立的命名空间，有自己的root文件系统、网络配置、进程空间、用户ID空间等，也就是它是一个隔离的环境。

每个容器运行时，是以镜像为基础层，在其上创建一个当前容器的存储层用于容器运行时读写，该存储层的生命周期和容器是一样的。容器存储层应保持无状态化，所有对文件的写入操作都应该使用数据卷或绑定宿主目录，在这些位置的读写就会跳过容器存储层，直接对宿主发生读写，其性能和稳定性更高。

### 创建容器
使用`docker create <image>`可以基于镜像创建一个容器，创建后其处于停止状态，需要用启动命令启动。
```powershell
PS C:\Windows\system32> docker create -it ubuntu:latest
1179ee64f25b6ddbe95e916c4b888ff4d05878ebabd77c94658c9cbc8686b360
PS C:\Windows\system32> docker start -i 117
root@1179ee64f25b:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

### 启动容器
使用`docker start <container_id/container_name>`可以将一个终止状态的容器启动运行。
```powershell
PS C:\Windows\system32> docker container start -i f8b
root@f8be09e399ae:/# ps
  PID TTY          TIME CMD
    1 pts/0    00:00:00 bash
   10 pts/0    00:00:00 ps
root@f8be09e399ae:/# exit
exit
```
使用`ps`或`top`可以查看进程信息，说明容器中仅运行了默认的bash应用。

### 新建并启动
使用`docker run`命令，相当于`docker create`+`docker start`，例如:
```powershell
PS C:\Users\hejl> docker run -i -t ubuntu:latest
root@f8be09e399ae:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

`-t`选项让docker分配一个伪终端来绑定到容器的标准输入上，`-i`选项让容器的标准输入保持打开。

当执行`docker run`时，Docker后台实际上运行了这些操作:

1. 检查本地是否存在指定的镜像，不存在就从公有仓库下载
2. 利用镜像创建并启动一个容器
3. 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
4. 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
5. 从地址池配置一个ip地址给容器
6. 执行用户指定的应用程序
7. 执行完毕后容器被终止

### 后台运行
很多时候，需要让Docker在后台运行而不是直接把执行命令的结果输出在当前宿主机上，此时可以通过添加`-d`参数来实现。`docker run`和`docker create`命令都可以使用该参数。

但容器是否会长久运行，只取决于`docker run`到底跑了什么进程，和`-d`参数无关。如:
```powershell
PS C:\Users\hejl> docker run -d ubuntu:18.04 /bin/sh -c "while true; do echo hello world; sleep 1; done"
6ec4042f4d06f94a46aadebe8d05f33d9916985b19bae6d8d63ed867d89b3714
PS C:\Users\hejl> docker logs 6ec
hello world
hello world
hello world
...
```
使用`docker logs`可以看到某后台运行的容器的输出信息。

### 查看所有容器
使用`docker ps`可以看到正在运行的容器，`docker ps -a`会看到包括终止状态的容器，它和`docker container ls -a`等价。

### 进入容器
对于后台运行的容器，有时需要进入容器进行操作，可以使用`docker exec`或`docker attach`命令，但attach进入后如果输入`exit`退出容器会导致容器状态停止，所以更推荐`exec`的方式。
```powershell
PS C:\Users\hejl> docker exec -it 6f8 bash
root@6f8a756d46fe:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@6f8a756d46fe:/# exit
exit
PS C:\Users\hejl> docker attach 6f8
root@6f8a756d46fe:/#    
```

### 终止和删除
使用`docker stop <container_id/container_name>`可停止运行中的容器。

使用`docker rm <container_id/container_name>`可删除，添加`-f`参数会强制删除即使该容器正在运行。

使用`docker container prune`可以清理掉所有处于终止状态的容器。


数据管理
-------
在docker内部及容器之间管理数据主要有两种方式，即通过数据卷和挂载主机目录。

### 数据卷
数据卷是一个特殊的目录，它会绕过Union FS，提供一些有用的特性:

* 可以在容器之间共享和重用
* 对数据卷的修改会立即生效
* 对数据卷的更新不会影响镜像
* 数据卷一直存在，即使容器被删除

可以通过`docker volume create my-vol`创建一个数据卷，然后使用`docker run -it -v my-vol:/webapp  ubuntu`来将创建的数据卷挂载到容器中，可同时挂载多个数据卷。

### 挂载主机目录
也可以直接将本地主机的一个目录挂载在容器中，本地目录必须是一个绝对路径，挂载后默认是可读写权限，可在挂载时设置为只读。例`docker run -it -v C:\Users\hejl:/mytt:ro ubuntu`。


网络
-------
启动容器时如果不指定参数，则在容器外部无法通过网络来访问容器内的网络应用和服务。

可以使用`-P`参数，大写P使得Docker会随机映射一个49000~49900的端口作为容器的开放网络端口。

也可通过`-p`小写p指定端口，如`docker run -p 5000:5000 -p 3000:80 ubuntu`。


Docker Compose
-------
Compose的定位是定义和运行多个Docker容器的应用，其中有两个重要的概念:

* 服务(service): 一个应用的容器，实际上可以包括若干运行相同镜像的容器实力。
* 项目(project): 由一组关联的应用容器组成的一个完整业务单元, 在`docker-compose.yml`文件中定义。

### 主要命令
对于Compose来说，大部分命令的对象既可以是项目本身，也可以是项目中的服务，如不特别说明，命令的对象就是项目，也就是说项目中的所有服务都会受到命令的影响。

**build**

用于构建或重新构建项目中的服务容器。

服务容器一旦构建后，将会带上一个标记名，例如对于web项目中的一个db容器，可能是web_db。

有一些额外选项:

* --force-rm，删除构建过程中的临时容器
* --no-cache，构建镜像过程中不使用缓存
* --pull，始终尝试通过pull来获取更新版本的镜像

**config**

验证`docker-compose.yml`文件中的格式是否正确，如果正确会显示配置，错误则显示错误原因。

**exec**

进入指定的容器，容器必须是运行状态的。例如`docker-compose exec web sh`

**run**

用于在指定服务上运行一个命令。

类似于启动容器后运行指定的命令，相关的卷、链接等也会按照配置自动创建，如果存在关联的服务则关联的服务也会自动启动。但指定命令会覆盖掉原有的自动运行的命令，且不会自动创建端口以免冲突。

额外选项:

* -d，后台运行容器
* --no-deps，不自动启动关联的容器
* -e KEY=VAL，设置环境变量
* --rm，运行命令后自动删除容器
* --service-ports，配置服务端口并映射到本地主机

例如: `docker-compose run --no-deps web python manage.py shell`

**up**

它将尝试自动完成包括构建镜像、(重新)创建服务，启动服务，并关联服务相关容器的一系列操作。大部分时候可以通过该命令来启动一个项目。

默认情况下，启动的容器都在前台，会输出容器的输出信息便于调试。生产环境下建议后台启动。

额外选项:

* -d，后台运行容器
* --no-deps，不启动服务链接的容器 
* --force-recreate，强制重新创建容器
* --no-create，如果容器已经存在了，则不重新创建
* -t，停止容器时的超时

### 模板文件
默认的文件名称为`docker-compose.yml`，下面主要介绍核心指令的含义。

每个服务都必须通过image指令或build指令来自动构建生成镜像，如果使用build，Dockerfile中设置的选项(如CMD、EXPOSE、VOLUME、ENV)将会自动被获取，无需在模板文件中重复设置。

**build**

指定Dockerfile所在文件夹的路径，可以是绝对路径或是相对于`docker-compose.yml`文件的相对路径。如:
```yml
version: '3'
services:

  webapp:
    build: ./dir
```

也可以使用context指令指定Dockerfile所在的文件夹路径，使用dockerfile指令指定Dockerfile的文件名，使用arg指令指定构建镜像时的变量。如：
```yml
version: '3'
services:

  webapp:
    build:
      context: ./dir
      dockerfile: Dockerfile-alternate
      args:
        buildno: 1
```

**command**

会覆盖掉容器启动后默认执行的命令。例如:
```yml
version: "3"
services:

  db:
     image: mysql:8.0
     command:
      - --character-set-server=utf8mb4
```
没有command时只启动`mysqld`，加上以后变为`mysqld --character-set-server=utf8mb4`

**depends_on**

解决容器之间的依赖、启动先后问题。

**environment**

设置环境变量，可以使用如下两种格式:
```yml
environment:
  RACK_ENV: development
  SESSION_SECRET:

environment:
  - RACK_ENV=development
  - SESSION_SECRET
```
在这里设置的环境变量，不能用于容器的构建过程中，只能在容器运行以后才能获取到。例如若dockerfie中有一层是`RUN flask db migrate`，则它无法获取到`docker-compose.yml`中设置的数据库连接的环境变量。正确的方法是dockerfile中没有这一层，而是通过`docker-compose run --rm web flask db migrate`来达到目的。

**expose**

暴漏端口，但不映射到宿主机，只有被连接的服务能通过该端口访问。

**image**

指定为镜像名称或镜像ID，如果镜像在本地不存在，将会尝试拉取这个镜像。
