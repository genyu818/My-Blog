# Docker：定义

#### 

### 1.	帮助启动类命令

启动docker： systemctl start docker

停止docker： systemctl stop docker

重启docker： systemctl restart docker

查看docker状态： systemctl status docker

查看docker概要信息： docker info

查看docker命令帮助文档： docker 具体命令 --help



### 2.	镜像命令

docker images : 列出本地主机上的镜像

docker search :搜索镜像

docker pull :下载镜像(没有TAG就是**最新版**)

docker system df : 查看镜像/容器/数据卷所占的空间

docker rmi -f 镜像名1:TAG 镜像名2:TAG : 删除镜像



### 3.	容器命令

docker run -it ：前台交互式启动镜像

docker run -d ：后台守护式启动镜像

docker logs 容器ID ：查看容器日志

docker top 容器ID：查看容器内运行的进程

docker inspect 容器ID：查看容器内部细节



#### 进入正在运行的容器并以命令行交互

docker exec -it 容器ID (exec 是在容器中打开新的终端，并且可以启动新的进程，用exit退出，不会导致容器的停止。)

docker attach 容器ID (attach 直接进入容器启动命令的终端，不会启动新的进程，用exit退出，会导致容器的停止。)

docker cp 容器ID:容器内路径 目的主机路径 :  从容器内拷贝文件到主机上

docker export 容器ID > 文件名.tar :  导出容器

docker commit -m="提交的描述信息" -a="作者" 容器ID 要创建的目标镜像名:[标签名] : docker commit提交容器副本使之成为一个新的镜像





### 4.	Docker容器卷

容器卷就是目录或文件，存在于一个或多个容器中，由docker挂载到容器，但不属于联合文件系统，因此能够绕过Union File System提供一些用于持续存储或共享数据的特性：

卷的设计目的就是数据的持久化，完全独立于容器的生存周期，因此Docker不会在容器删除时删除其挂载的数据卷。



#### **命令**

docker run -it --privileged=true -v /宿主机绝对路径目录:/容器内目录 镜像名 ： 宿主vs容器之间映射添加容器卷 （**读写**）

docker run -it --privileged=true -v /宿主机绝对路径目录:/容器内目录:ro 镜像名 ： 宿主vs容器之间映射添加容器卷 （**只读**）

docker run -it --privileged=true --volumes-from 父类 --name u2 ubuntu ： 容器卷的继承





### 5.	DockerFile

Dockerfile是用来构建Docker镜像的文本文件，是由一条条构建镜像所需的指令和参数构成的脚本。



##### 1.	构造三部曲

1. 编写Dockerfile文件
2. docker build命令构建镜像
3. docker run依镜像运行容器实例



##### 2.	Dockerfile内容基础知识

1. 每条保留字指令都必须为大写字母且后面要跟随至少一个参数
2. 指令按照从上到下，顺序执行
3. 每条指令都会创建一个新的镜像层并对镜像进行提交



##### 3.	DockerFile关键字

FROM ： 基础镜像，当前新镜像是基于哪个镜像的，指定一个已经存在的镜像作为模板，第一条必须是from

MAINTAINER： 镜像维护者的姓名和邮箱地址

RUN：容器构建时需要运行的命令 **(RUN是在 docker build时运行)**

EXPOSE: 当前容器对外暴露出的端口

WORKDIR : 指定在创建容器后，终端默认登陆的进来工作目录，一个落脚点

USER : 指定该镜像以什么样的用户去执行，如果都不指定，默认是root

ENV : 用来在构建镜像过程中设置环境变量

ADD : 将宿主机目录下的文件拷贝进镜像且会自动处理URL和解压tar压缩包

COPY : 类似ADD，拷贝文件和目录到镜像中。将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置

VOLUME : 容器数据卷，用于数据保存和持久化工作

CMD : 指定容器启动后的要干的事情 **（Dockerfile 中可以有多个 CMD 指令，但只有最后一个生效，CMD 会被 docker run 之后的参数替换， CMD是在docker run 时运行。）**

ENTRYPOINT：  类似于 CMD 指令，但是ENTRYPOINT不会被docker run后面的命令覆盖，而且这些命令行参数会被当作参数送给 ENTRYPOINT 指令指定的程序（**如果 Dockerfile 中如果存在多个 ENTRYPOINT 指令，仅最后一个生效。**）



范例 

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.16-alpine AS build

# Install tools required for project
# Run `docker build --no-cache .` to update dependencies
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies with Gopkg.toml and Gopkg.lock
# These layers are only re-built when Gopkg files are updated
COPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# Install library dependencies
RUN dep ensure -vendor-only

# Copy the entire project and build it
# This layer is rebuilt when a file changes in the project directory
COPY . /go/src/project/
RUN go build -o /bin/project
```

