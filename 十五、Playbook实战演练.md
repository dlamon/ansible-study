# Playbook 实战演练

## 1、系统架构图

![avatar](/images/ansible-example.png)

说明：

- 系统需要部署三组应用，分别为 webservers ，appservers 和 manageservers
- webservers 需要安装 nginx 中间件和部署前端应用
- appservers 需要安装 jdk 环境和部署后端应用
- manageservers 需要安装 jdk 环境和部署管理端应用
- webservers 由于没有外层负载均衡的原因，只部署一台
- appservers 由 webservers 所在的 nginx 进行负载均衡，部署五台
- appservers 模拟核心系统在生产环境的部署方案，进行滚动更新
- 部署所需要的安装包（nginx，jdk）和部署版本文件模拟成生产环境，从文件服务器上获取
- 原则上数据库和文件服务器不需要我们自行部署，所以使用 docker 容器快速搭建

## 2、创建 docker 自定义网络

```shell
docker network create ansible-example
```

## 3、部署数据库

创建文件夹 mysql，其中目录结构如下：

```ini
.
|-- conf
|   ├── config-file.cnf
|-- data
|-- docker-compose.yml
```

config-file.cnf 文件内容如下：

```ini
[mysqld]
# 取消表名大小写敏感设置
lower_case_table_names=1
```

data 文件夹为空，用于保存容器中映射的数据文件

docker-compose.yml 文件内容如下：

```yml
version: "3"
services:
  mysql:
    image: "mysql:5.7.29"
    restart: unless-stopped
    container_name: mysql
    environment:
      - MYSQL_ROOT_PASSWORD=root123
      - TZ=Asia/Shanghai
    ports:
      - "3306:3306"
    volumes:
      - /Users/liaowei/Work/docker/mysql/conf:/etc/mysql/conf.d
      - /Users/liaowei/Work/docker/mysql/data:/var/lib/mysql
    networks:
        - ansible-example
networks:
  ansible-example:
    external: true
```

其中 */Users/liaowei/Work/docker/mysql* 为宿主机上的文件夹路径

运行 docker-compose 命令：

```shell
docker-compose up -d
```

完成数据库容器创建后需要手工导入相关的建表和数据插入脚本。
目前示例程序还未集成数据库自动化操作流程。

## 3、部署文件服务器

docker-compose.yml 文件内容如下：

```yml
version: "3"
services:
  nginx:
    image: "nginx:1.17.9"
    restart: unless-stopped
    container_name: file-server
    environment:
      - TZ=Asia/Shanghai
    ports:
      - "80:80"
    volumes:
      - /Users/liaowei/Work/docker/nginx/file-server/nginx.conf:/etc/nginx/nginx.conf:ro
      - /Users/liaowei/Work/docker/nginx/file-server/site:/var/site
    networks:
      - ansible-example
networks:
  ansible-example:
    external: true
```

nginx.conf 文件内容如下：

```conf
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;

    keepalive_timeout  65;

    gzip  on;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_comp_level 2;
    gzip_types text/plain application/javascript application/css  text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    gzip_vary off;
    gzip_disable "MSIE [1-6]\.";

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   /var/site;
            autoindex on;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

file-server/site 文件夹结构如下：

```ini
.
├── deploy-app
│   ├── ansible-front
│   │   └── ansible-front-1.0.0.zip
│   ├── ansible-server
│   │   ├── ansible-db_2020-03-20.sql
│   │   └── ansible-server-0.0.1-SNAPSHOT.jar
│   └── ansible-server-admin
│       └── ansible-server-admin-0.0.1-SNAPSHOT.jar
├── deploy-base
│   ├── gcc
│   │   ├── cpp-4.8.5-39.el7.x86_64.rpm
│   │   ├── gcc-4.8.5-39.el7.x86_64.rpm
│   │   ├── glibc-devel-2.17-292.el7.x86_64.rpm
│   │   ├── glibc-headers-2.17-292.el7.x86_64.rpm
│   │   ├── kernel-headers-3.10.0-1062.12.1.el7.x86_64.rpm
│   │   ├── libgomp-4.8.5-39.el7.x86_64.rpm
│   │   ├── libmpc-1.0.1-3.el7.x86_64.rpm
│   │   └── mpfr-3.1.1-4.el7.x86_64.rpm
│   ├── gcc-c++
│   │   ├── gcc-c++-4.8.5-39.el7.x86_64.rpm
│   │   └── libstdc++-devel-4.8.5-39.el7.x86_64.rpm
│   ├── java
│   │   └── jdk-8u241-linux-x64.tar.gz
│   ├── make
│   │   └── make-3.82-24.el7.x86_64.rpm
│   ├── nginx
│   │   └── nginx-1.16.1.tar.gz
│   ├── openssl
│   │   └── openssl-1.0.2r.tar.gz
│   ├── pcre
│   │   └── pcre-8.43.tar.gz
│   ├── perl
│   │   ├── groff-base-1.22.2-8.el7.x86_64.rpm
│   │   ├── perl-5.16.3-294.el7_6.x86_64.rpm
│   │   ├── perl-5.16.3.tar.gz
│   │   ├── perl-Carp-1.26-244.el7.noarch.rpm
│   │   ├── perl-constant-1.27-2.el7.noarch.rpm
│   │   ├── perl-Encode-2.51-7.el7.x86_64.rpm
│   │   ├── perl-Exporter-5.68-3.el7.noarch.rpm
│   │   ├── perl-File-Path-2.09-2.el7.noarch.rpm
│   │   ├── perl-File-Temp-0.23.01-3.el7.noarch.rpm
│   │   ├── perl-Filter-1.49-3.el7.x86_64.rpm
│   │   ├── perl-Getopt-Long-2.40-3.el7.noarch.rpm
│   │   ├── perl-HTTP-Tiny-0.033-3.el7.noarch.rpm
│   │   ├── perl-libs-5.16.3-294.el7_6.x86_64.rpm
│   │   ├── perl-macros-5.16.3-294.el7_6.x86_64.rpm
│   │   ├── perl-parent-0.225-244.el7.noarch.rpm
│   │   ├── perl-PathTools-3.40-5.el7.x86_64.rpm
│   │   ├── perl-Pod-Escapes-1.04-294.el7_6.noarch.rpm
│   │   ├── perl-podlators-2.5.1-3.el7.noarch.rpm
│   │   ├── perl-Pod-Perldoc-3.20-4.el7.noarch.rpm
│   │   ├── perl-Pod-Simple-3.28-4.el7.noarch.rpm
│   │   ├── perl-Pod-Usage-1.63-3.el7.noarch.rpm
│   │   ├── perl-Scalar-List-Utils-1.27-248.el7.x86_64.rpm
│   │   ├── perl-Socket-2.010-4.el7.x86_64.rpm
│   │   ├── perl-Storable-2.45-3.el7.x86_64.rpm
│   │   ├── perl-Text-ParseWords-3.29-4.el7.noarch.rpm
│   │   ├── perl-threads-1.87-4.el7.x86_64.rpm
│   │   ├── perl-threads-shared-1.43-6.el7.x86_64.rpm
│   │   ├── perl-Time-HiRes-1.9725-3.el7.x86_64.rpm
│   │   └── perl-Time-Local-1.2300-2.el7.noarch.rpm
│   └── zlib
│       └── zlib-1.2.11.tar.gz
└── deploy-tool
    ├── ansible-example
    │   └── ansible-example.zip
    └── ansible-rsa-tool
        └── ansible-rsa-tool.zip
```

目录说明：

- deploy-app: 需要部署应用版本
- deploy-base: 安装 nginx 和 java 需要的安装包
- deploy-tool: 自动化部署工具

运行 docker-compose 命令：

```shell
docker-compose up -d
```

## 4、搭建管理主机

```shell
docker run --privileged -itd --name ansible-deploy-master -p 20022:22 --network ansible-example centos:deploy
```

管理主机需要安装 ansible，可以参考 **二、搭建Ansible 环境 2.1、安装Ansible**

## 5、应用服务器列表

| 应用名称 | 服务器域名 | 服务器数量 | 所属机房 |
| :- | :- | :-: | :- |
| 前端应用 | ansible-front | 1 | 生产 |
| 后端服务 | ansible-server-sc1| 1 | 生产 |
| 后端服务 | ansible-server-sc2| 1 | 生产 |
| 后端服务 | ansible-server-sc3| 1 | 生产 |
| 后端服务 | ansible-server-tc1| 1 | 同城 |
| 后端服务 | ansible-server-zb1| 1 | 灾备 |
| 后端管理控制台 | ansible-server-admin | 1 | 生产 |

## 6、搭建应用服务器容器

搭建前端应用服务器容器：

```shell
docker run --privileged -itd --name ansible-front -p 20080:20080 --network ansible-example centos:deploy
```

搭建后端服务容器：

```shell
docker run --privileged -itd --name ansible-server-sc1 -p 20000:20000 --network ansible-example centos:deploy
```

```shell
docker run --privileged -itd --name ansible-server-sc2 -p 20001:20000 --network ansible-example centos:deploy
```

```shell
docker run --privileged -itd --name ansible-server-sc3 -p 20002:20000 --network ansible-example centos:deploy
```

```shell
docker run --privileged -itd --name ansible-server-tc1 -p 20003:20000 --network ansible-example centos:deploy
```

```shell
docker run --privileged -itd --name ansible-server-zb1 -p 20004:20000 --network ansible-example centos:deploy
```

搭建后端管理控制台容器：

```shell
docker run --privileged -itd --name ansible-server-admin -p 21080:21080 --network ansible-example centos:deploy
```

## 7、使用 ansible-rsa-tool

使用 ansible-rsa-tool 可以快速复制管理主机公钥到托管节点，不再需要逐台进行 ssh-copy-id 操作。
在管理主机 deploy 用户目录，获取 ansible-rsa-tool.zip 压缩包，解压后目录结构如下：

```ini
.
├── authkey.yml
├── bin
│   ├── ansible.cfg
│   ├── copy.sh
│   └── remove.sh
├── production
│   ├── group_vars
│   │   └── servers.yml
│   └── hosts
├── roles
│   └── common
│       └── tasks
│           └── main.yml
└── site.yml
```

在 ansible-rsa-tool 执行命令如下：

```shell
# 在管理主机上产生公私钥
ssh-keygen
```

```shell
# 复制管理主机公钥到托管节点
ansible-playbook -i production site.yml --tags="copy"
```

```shell
# 移除托管节点上管理主机公钥
ansible-playbook -i production site.yml --tags="remove"
```

或者直接使用 bin 文件夹中的 copy.sh 和 remove.sh 命令。

 <font size=3>*重要提示：*</font>

- 如果**托管节点**有节点新增，由于 authorized_key 模块支持幂等，可以直接运行 tags 为 copy 的命令新增
- 如果**托管节点**有节点减少，建议在减少节点前手工移除单个节点上的 authorized_keys 中关于管理主机的配置，或者使用 tags 为 remove 的命令全部移除，再重新复制
- 如果**托管节点**故障重新换机，IP或域名解析地址保持不变，除了复制公钥操作外，还需要同步修改**管理主机** .ssh 目录中 known_hosts 文件，否则会报错
- 在**托管节点**有变化时需要及时保存管理主机公私钥，以防止**管理主机**故障后无法恢复密钥和配置
- 本小节可以参考 ansible-rsa-tool 工程

## 8、使用 ansible-example

ansible-example 是一个自动化部署的示例工程。
ansible-example 可以通过简单的命令完成设置环境，基础中间件的安装，版本备份，版本部署，服务停止，服务启动和服务检查等多个功能。

在管理主机 deploy 用户目录，获取 ansible-example.zip 压缩包，解压后目录结构如下：

```output
.
├── ansible.cfg
├── appservers.yml
├── manageservers.yml
├── production
│   ├── group_vars
│   │   ├── all.yml
│   │   ├── appservers.yml
│   │   ├── manageservers.yml
│   │   └── webservers.yml
│   └── hosts
├── roles
│   ├── ansible-front
│   │   ├── tasks
│   │   │   ├── backup.yml
│   │   │   ├── cleanup.yml
│   │   │   ├── download.yml
│   │   │   ├── main.yml
│   │   │   ├── start.yml
│   │   │   ├── stop.yml
│   │   │   └── update.yml
│   │   ├── templates
│   │   │   └── nginx.conf
│   │   └── vars
│   │       └── main.yml
│   ├── ansible-server
│   │   ├── tasks
│   │   │   ├── backup.yml
│   │   │   ├── cleanup.yml
│   │   │   ├── download.yml
│   │   │   ├── main.yml
│   │   │   ├── start.yml
│   │   │   ├── stop.yml
│   │   │   └── update.yml
│   │   └── vars
│   │       └── main.yml
│   ├── ansible-server-admin
│   │   ├── tasks
│   │   │   ├── backup.yml
│   │   │   ├── cleanup.yml
│   │   │   ├── download.yml
│   │   │   ├── main.yml
│   │   │   ├── start.yml
│   │   │   ├── stop.yml
│   │   │   └── update.yml
│   │   └── vars
│   │       └── main.yml
│   ├── bash_profile
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── templates
│   ├── java
│   │   ├── tasks
│   │   │   ├── check.yml
│   │   │   ├── cleanup.yml
│   │   │   ├── download.yml
│   │   │   ├── install.yml
│   │   │   └── main.yml
│   │   ├── templates
│   │   └── vars
│   │       └── main.yml
│   └── nginx
│       ├── tasks
│       │   ├── check.yml
│       │   ├── cleanup.yml
│       │   ├── download.yml
│       │   ├── install.yml
│       │   └── main.yml
│       ├── templates
│       │   └── nginx.conf
│       └── vars
│           ├── download.yml
│           ├── install.yml
│           └── main.yml
├── rsa_keys
│   └── deploy@ansible-deploy-master.rsa
├── site.yml
└── webservers.yml
```

**注意：**

- 在进行操作前需要把当前管理主机使用的公钥拷贝到 rsa_keys 目录中覆盖 deploy@ansible-deploy-master.rsa
- 需要确认 deploy@ansible-deploy-master.rsa 文件权限为 600，如果不是，则使用一下命令设置：
  
```shell
chmod 600 deploy@ansible-deploy-master.rsa
```

**注意：**
由于安装 nginx 需要系统的 gcc 等相关依赖，底层依赖需要 root 用户才能安装，所以在安装 nginx 时，需要使用 --ask-become-pass , 并在执行命令后输入 root 用户密码，在 playbook 执行到安装 nginx 步骤时需要 root 用户密码进行切换。

可以实现的功能：

```shell
# 配置 .bash_profile， 安装 nginx 和部署前端应用 ansible-front
ansible-playbook -i production webservers.yml --ask-become-pass"
```

```shell
# 部署前端应用 ansible-front
ansible-playbook -i production webservers.yml --tags="deploy"
```

```shell
# 配置 .bash_profile， 安装 java 和部署后端应用 ansible-server
ansible-playbook -i production appservers.yml
```

```shell
# 部署后端应用 ansible-server
ansible-playbook -i production appservers.yml --tags="deploy"
```

```shell
# 配置 .bash_profile， 安装 java 和部署管理端应用 ansible-server-admin
ansible-playbook -i production manageservers.yml
```

```shell
# 部署管理端应用 ansible-server-admin
ansible-playbook -i production manageservers.yml --tags="deploy"
```

```shell
# 配置 .bash_profile， 安装所有中间件和部署所有应用（.bash_profile, nginx,java,ansible-front,ansible-server,ansible-server-admin）
ansible-playbook -i production site.yml --ask-become-pass
```

```shell
# 部署所有应用（ansible-front,ansible-server,ansible-server-admin）
ansible-playbook -i production site.yml --tags="deploy"
```

ansible-example 大致的逻辑关系：

```output
├── site.yml
    ├── webservers.yml
    │   ├── bash_profile(role)
    │   ├── nginx(role)
    │   └── ansible-front(role)
    ├── appservers.yml
    │   ├── bash_profile(role)
    │   ├── java(role)
    │   └── ansible-server(role)
    └── manageservers.yml
        ├── bash_profile(role)
        ├── java(role)
        └── ansible-server-admin(role)
```

## 参考资料

- [authorized_key 模块](https://docs.ansible.com/ansible/latest/modules/authorized_key_module.html#authorized-key-module)
