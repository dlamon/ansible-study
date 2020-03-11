# 二、搭建Ansible环境

## 1、Ansible基本概念

Ansible 默认通过 SSH 协议管理机器。
Ansible 节点类型可以分为管理主机和托管节点。

### 管理主机要求

目前，只要机器上安装了 Python 2.6 或 Python 2.7 （windows系统不可以做控制主机），都可以运行Ansible。

主机的系统可以是 Red Hat, Debian, CentOS, OS X, BSD的各种版本，等等。

### 托管节点要求

通常我们使用 ssh 与托管节点通信，默认使用 sftp.如果 sftp 不可用，可在 ansible.cfg 配置文件中配置成 scp 的方式。 在托管节点上也需要安装 Python 2.4 或以上的版本。如果版本低于 Python 2.5 ,还需要额外安装一个模块：

```shell
yum install python-simplejson -y
```

## 2、配置管理主机

### 2.1、安装Ansible

进入配置管理主机（ master ）容器：

```shell
docker exec -it deploy-master bash
```

更新 yum 安装源：

```shell
yum update -y
```

通过 yum 安装最新发布版本：

```shell
yum install epel-release ansible -y
```

验证 Ansible 是否成功安装：

```shell
ansible --version
```

如果命令正确执行且输出如下内容，则安装成功。

```console
ansible 2.9.3
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /bin/ansible
  python version = 2.7.5 (default, Aug  7 2019, 00:51:29) [GCC 4.8.5 20150623 (Red Hat 4.8.5-39)]
```

### 2.2、配置 SSH 免密登录

#### 假定容器IP地址列表

- *deploy-master*：172.17.0.2
- *deploy-client-1*: 172.17.0.3
- *deploy-client-2*: 172.17.0.4
- *deploy-client-2*: 172.17.0.5

进入配置管理主机（ master ）容器：

```shell
docker exec -it deploy-master bash
```

切换到deploy用户：

```shell
su - deploy
```

使用ssh-keygen命令生成公私钥对：

```shell
# 输入yes或回车直到生成成功
ssh-keygen
```

使用ssh-copy-id命令将公钥复制到远程机器（需要输入远程主机密码）：

```shell
ssh-copy-id deploy@172.17.0.3
```

```shell
ssh-copy-id deploy@172.17.0.4
```

```shell
ssh-copy-id deploy@172.17.0.5
```

测试免密登录是否配置成功：

```shell
ssh deploy@172.17.0.3
```

```shell
ssh deploy@172.17.0.4
```

```shell
ssh deploy@172.17.0.5
```

如果上述命令不需要输入密码即可登入，则免密登录配置成功

### 2.3、测试 Ansible 是否连通

修改 Ansible 默认配置文件 */etc/ansible/hosts* ，添加托管节点IP地址

```shell
sudo sh -c "echo '172.17.0.3'>> /etc/ansible/hosts"
```

```shell
sudo sh -c "echo '172.17.0.4'>> /etc/ansible/hosts"
```

```shell
sudo sh -c "echo '172.17.0.5'>> /etc/ansible/hosts"
```

使用命令测试是否连通：

```shell
ansible all -m ping --user=deploy
```

如果已经连通，则会输出如下内容：

```console
172.17.0.3 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
172.17.0.4 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
172.17.0.5 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

最后移除在默认配置文件 */etc/ansible/hosts* 中添加的 172.17.0.3 、 172.17.0.4 和 172.17.0.5 节点配置
