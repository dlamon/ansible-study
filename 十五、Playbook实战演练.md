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

## 7、复制管理主机公钥

在管理主机 deploy 用户目录 建立 ansible-rsa-tool 文件夹，文件夹结构如下：

```ini
.
├── ansible.cfg
├── authkey.yml
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

关键配置如下：

```yml
---
# file: roles/common/tasks/main.yml
- name: 复制管理主机公钥到托管节点
  authorized_key:
    user: deploy
    state: present
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
  tags: copy
- name: 移除托管节点上管理主机公钥
  authorized_key:
    user: deploy
    state: absent
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
  tags: remove
```

配置密钥使用了 ansible 的 authorized_key 模块，上述配置支持公钥的添加和删除。

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

 <font size=3>*重要提示：*</font>

- 如果**托管节点**有节点新增，由于 authorized_key 模块支持幂等，可以直接运行 tags 为 copy 的命令新增
- 如果**托管节点**有节点减少，建议在减少节点前手工移除单个节点上的 authorized_keys 中关于管理主机的配置，或者使用 tags 为 remove 的命令全部移除，再重新复制
- 如果**托管节点**故障重新换机，IP或域名解析地址保持不变，除了复制公钥操作外，还需要同步修改**管理主机** .ssh 目录中 known_hosts 文件，否则会报错
- 在**托管节点**有变化时需要及时保存管理主机公私钥，以防止**管理主机**故障后无法恢复密钥和配置
- 本小节可以参考 ansible-rsa-tool 工程

## 8、配置资产清单文件

文件结构如下：

```ini
.
├── production
│   └── hosts
```

```ini
# file: production/hosts
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

## 7、配置私钥登录

目录结构：

```ini
.
├── ansible.cfg
├── production
│   ├── group_vars
│   │   └── all.yml
│   └── hosts
├── rsa_keys
│   └── deploy@ansible-deploy-master.rsa
```

**ansible.cfg:**

原因：当 ansible 第一次使用 ssh 进行托管节点连接时，由于本机的 known_hosts 文件中没有对应节点的 fingerprint key 串，会提示输入 yes 进行确认后再将 key 字符串加入到  ~/.ssh/known_hosts 文件中。
解决方案： 设置 ansible.cfg 配置文件，将 host_key_checking 设置为 False。

```ini
# file: ansible.cfg
[defaults]
host_key_checking = False
```

**all.yml：**

all.yml 文件中指定的变量对全组生效，在文件中可以配置所有托管节点的连接参数。
具体细节内容可以参考第十四章。

```yml
---
# file: production/group_vars/all.yml
ansible_user: deploy
ansible_ssh_private_key_file: ./rsa_keys/deploy@ansible-deploy-master.rsa
```

**deploy@ansible-deploy-master.rsa:**

在本章 7、配置管理主机免密登录章节中使用 ssh-keygen 命令产生的私钥文件（id_rsa）。
私钥文件建议统一规范化命名，方便后期管理。
命名格式：用户@IP地址或域名.rsa

***特别注意：***
rsa 文件权限只能为当前用户读和写，如果文件权限不正确，ansible 执行命令会报错，可以使用以下命令更改文件权限：

```shell
chmod 600 deploy@ansible-deploy-master.rsa
```

## 8、安装 Nginx

```shell
ansible-playbook -i production webservers.yml --ask-become-pass
```

## 9、安装 JDK

## 10、部署 ansible front

## 11、部署 ansible server

## 12、部署 ansible server admin

## 参考资料

- [authorized_key 模块](https://docs.ansible.com/ansible/latest/modules/authorized_key_module.html#authorized-key-module)
