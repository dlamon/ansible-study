# Playbook 实战演练

## 1、系统架构图

![avatar](/images/ansible-example.png)

说明：

- ansible-font 由于没有外层负载均衡的原因，只部署一台
- ansible-server 由 ansible-font 所在的 nginx 进行负载均衡，部署五台
- ansible-server 模拟核心系统在生产环境的部署方案，进行滚动更新
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

运行 docker-compose 命令：

```shell
docker-compose up -d
```

## 3、部署文件服务器

创建文件夹 file-server，其中目录结构如下：

```ini
.
├── docker-compose.yml
├── nginx.conf
└── site
    ├── ansible-front
    │   ├── ...
    ├── ansible-server
    │   ├── ansible-db_2020-03-20.sql
    │   └── ansible-server-0.0.1-SNAPSHOT.jar
    ├── ansible-server-admin
    │   └── ansible-server-admin-0.0.1-SNAPSHOT.jar
    ├── gcc
    │   ├── cpp-4.8.5-39.el7.x86_64.rpm
    │   ├── gcc-4.8.5-39.el7.x86_64.rpm
    │   ├── glibc-devel-2.17-292.el7.x86_64.rpm
    │   ├── glibc-headers-2.17-292.el7.x86_64.rpm
    │   ├── kernel-headers-3.10.0-1062.12.1.el7.x86_64.rpm
    │   ├── libgomp-4.8.5-39.el7.x86_64.rpm
    │   ├── libmpc-1.0.1-3.el7.x86_64.rpm
    │   └── mpfr-3.1.1-4.el7.x86_64.rpm
    ├── gcc-c++
    │   ├── gcc-c++-4.8.5-39.el7.x86_64.rpm
    │   └── libstdc++-devel-4.8.5-39.el7.x86_64.rpm
    ├── jdk
    │   └── jdk-8u241-linux-x64.tar.gz
    ├── nginx
    │   └── nginx-1.16.1.tar.gz
    ├── openssl
    │   ├── make-3.82-24.el7.x86_64.rpm
    │   └── openssl-1.0.2k-19.el7.x86_64.rpm
    ├── pcre
    │   └── pcre-devel-8.32-17.el7.x86_64.rpm
    └── zlib
        └── zlib-devel-1.2.7-18.el7.x86_64.rpm
```

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

运行 docker-compose 命令：

```shell
docker-compose up -d
```

## 4、搭建主控服务器

```shell
docker run --privileged -itd --name ansible-deploy-master -p 20022:22 --network ansible-example centos:deploy
```

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
docker run --privileged -itd --name ansible-front -p 20080:80 --network ansible-example centos:deploy
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

## 7、配置应用服务器私钥

建立 rsa-private-tools 文件夹：
```ini

# 

```

## 8、配置资产清单文件

```ini
[sc_front]
ansible-front

[sc_server]
ansible-server-sc1
ansible-server-sc2
ansible-server-sc3

[tc_server]
ansible-server-tc1

[zb_server]
ansible-server-zb1

[sc_server_admin]
ansible-server-admin

# front in all geos
[front:children]
sc_front

# server in all geos
[server:children]
sc_server
tc_server
zb_server

# server admin in all geos
[server_admin:children]
sc_server_admin

# everything in the sc geo
[sc:children]
sc_front
sc_server
sc_server_admin

# everything in the tc geo
[tc:children]
tc_server

# everything in the zb geo
[tc:children]
zb_server
```

## 7、安装 nginx

## 8、配置 JAVA 环境
