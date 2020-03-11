# 三、Ansible配置文件

## 1、主配置文件

ansible 默认主配置文件路径为：*/etc/ansible/ansible.cfg*
可以使用环境变量ANSIBLE_CONFIG更改配置文件路径，例如：

```shell
export $ANSIBLE_CONFIG=/ansible/ansible.cfg
```

ansible 读取配置路径的优先级：

- ANSIBLE_CONFIG 环境变量指向的配置文件
- ./ansible.cfg 当前目录下的配置文件
- ~/ansible.cfg 当前用户目录下的配置文件
- /etc/ansible/ansible.cfg 默认配置文件
  
## 2、资产清单文件

ansible 资产清单文件默认路径为： */etc/ansible/hosts*
ansible 资产清单文件路径可以在 ansible.cfg 文件中配置

## 3、自定义配置文件

以下示例演示了如何自定义配置ansible主配置文件和资产清单文件：

进入配置管理主机（ master ）容器：

```shell
docker exec -it deploy-master bash
```

切换到deploy用户：

```shell
su - deploy
```

在deploy用户目录中创建ansible目录：

```shell
mkdir ~/ansible
```

在ansible目录中创建ansible.cfg文件，文件内容如下：

```ini
[defaults]
inventory = ~/ansible/hosts
log_path = ~/ansible/log/ansible.log
```

在ansible目录中创建hosts文件，文件内容如下：

```ini
172.17.0.3
172.17.0.4
172.17.0.5
```

在ansible目录中创建log文件夹和ansible.log文件

```shell
mkdir ~/ansible/log
touch ~/ansible/log/ansible.log
```

进入ansible目录，运行命令查看所有配置的主机：

```shell
ansible all --list-hosts
```

如果输出一下内容，则配置成功：

```output
hosts (2):
    172.17.0.3
    172.17.0.4
    172.17.0.5
```
