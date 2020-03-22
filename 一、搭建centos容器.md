# 一、搭建centos容器

## 1、自定义centos镜像

使用 *Dockerfile* 基于 *centos:centos7.7.1908* 版本创建自定义 centos 镜像

```Dockerfile
# 自定义centos:deploy镜像
FROM centos:centos7.7.1908
# 创建维护者信息
LABEL maintainer="LiaoWei"
# 创建描述信息
LABEL description="Test ansible image"
# 修改root用户密码
RUN echo 'root:root123' | chpasswd
# 创建deploy用户
RUN useradd --create-home --no-log-init --shell /bin/bash deploy
# 修改deploy用户密码
RUN echo 'deploy:deploy123' | chpasswd
# 将deploy用户加入到sudoers
RUN echo 'deploy ALL=(ALL) ALL'>> /etc/sudoers
# 安装系统软件
RUN yum update -y
RUN yum install sudo wget net-tools tree zip unzip initscripts openssh-server openssh-clients -y
# 配置CMD
CMD ["/usr/sbin/init"]
```

编写上述内容并保存到 *Dockerfile* 文件，在 *Dockerfile* 文件当前目录运行以下命令

```shell
docker build -t centos:deploy .
```

命令执行完成后，使用以下命令查看镜像

```shell
docker images
```

如果上述操作成功完成，则会输出 REPOSITORY 为 *centos* ，TAG 为 *deploy* 的镜像

```console
REPOSITORY                     TAG                 IMAGE ID            CREATED             SIZE
centos                         deploy              86b3d885eb55        About an hour ago   494MB
```

## 2、创建测试容器

### 2.1、创建管理主机（ master ）容器

运行以下命令，创建管理主机（ master ）容器

```shell
docker run --privileged -itd -p 20022:22 --name deploy-master centos:deploy
```

- *--privileged* 由于在容器中需要启动系统服务(比如 SSH )，所以使用特权模式
- *-i* 以交互模式运行容器，通常与 *-t* 同时使用
- *-t* 为容器重新分配一个伪输入终端，通常与 *-i* 同时使用
- *-d* 后台运行容器，并返回容器ID
- *-p* 指定宿主机和容器之间的端口映射，将容器内的22端口映射到宿主机的20022端口
- *--name* 指定创建的容器名称为deploy-master
  
使用以下命令查看创建的容器状态

```shell
docker ps
```

或者使用

```shell
docker ps -a | grep deploy-master
```

### 2.2、创建托管节点（ client ）容器

运行以下命令，创建托管节点（ client ）容器（三台）

```shell
docker run --privileged -itd --name deploy-client-1 centos:deploy
```

```shell
docker run --privileged -itd --name deploy-client-2 centos:deploy
```

```shell
docker run --privileged -itd --name deploy-client-3 centos:deploy
```

由于两台测试容器不需要从宿主机器拷贝文件等资源，只需要和主控（ master ）通信，则不需要另行映射端口

使用以下命令查看创建的容器状态

```shell
docker ps
```

或者使用

```shell
docker ps -a | grep deploy-client
```
