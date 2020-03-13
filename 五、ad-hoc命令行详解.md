# 五、ad-hoc命令行详解

Ansible ad-hoc 命令使用 */usr/bin/ansible* 命令行工具来发起一个或多个受管节点上的单个任务。

Ansible ad-hoc 命令格式为：

```shell
ansible [pattern] -m [module] -a "[module options]"
```

## 1、pattern

### 1.1、常见模式

Description | Pattern(s) | Targets
| :- | :- | :-
| All hosts | all (or *)
| One host | host1
| Multiple hosts | host1:host2 (or host1,host2)
| One group | webservers
| Multiple groups | webservers:dbservers | all hosts in webservers plus all hosts in dbservers
| Excluding groups | webservers:!atlanta | all hosts in webservers except those in atlanta
| Intersection of groups | webservers:&staging | any hosts in webservers that are also in staging

所有模式可以按照任意方式组合:

```shell
webservers:dbservers:&staging:!phoenix
```

可以同时混合使用通配符模式和组:

```shell
one*.com:dbservers
```

-*建议：模式不要写得太复杂，尽量简单易读*

### 1.2、模式的局限性

模式的编写依赖于资产清单文件的编写，模式必须符合资产清单文件的语法，例如资产清单文件如下：

```ini
# file: hosts
[sc]
centos01 ansible_host=172.17.0.3
```

如果在资产清单中定义了别名，模式中就必须使用别名。

```shell
ansible centos01 -m ping
```

如果使用IP地址：

```shell
ansible 172.17.0.3 -m ping
```

将输出错误信息：

```output
[WARNING]: Could not match supplied host pattern, ignoring: 172.17.0.3

[WARNING]: No hosts matched, nothing to do
```

### 1.3、进阶模式选项

#### 1.3.1、使用变量

可以通过ansible-playbook的-e参数传递变量到模式中使用：

```shell
websersvers:!{{ excluded }}:&{{ required }}
```

#### 1.3.2、使用组的位置

如果定义了以下组：

```ini
# file: hosts
[servers]
172.17.0.3
172.17.0.4
172.17.0.5
```

可以使用下表在组内选择主机和范围：

```ini
servers[0] # == 172.17.0.3
servers[-1] # == 172.17.0.5
servers[0:2] # == 172.17.0.3,172.17.0.4
servers[1:] # == 172.17.0.4,172.17.0.5
servers[:3] # == 172.17.0.3,172.17.0.4,172.17.0.5
```

#### 1.3.3、使用正则表达式

```ini
~(web|db).*\.example\.com
```

模式部分官方文档参见 [Patterns: targeting hosts and groups](https://docs.ansible.com/ansible/latest/user_guide/intro_patterns.html#)

## 2、module

### 2.1、基本概念

Modules (also referred to as “task plugins” or “library plugins”) are discrete units of code that can be used from the command line or in a playbook task.

Ansible通常在远程目标节点上执行每个模块，并收集返回值。

```shell
ansible webservers -m service -a "name=httpd state=started"
ansible webservers -m ping
ansible webservers -m command -a "/sbin/reboot -t now"
```

在playbooks中，可以配置为配置为：

```yml
- name: restart webserver
  service:
    name: httpd
    state: restarted
```

```yml
- name: reboot the servers
  command: /sbin/reboot -t now
```

特征：

- 每个模块都支持使用参数
- 几乎所有模块都采用 key = value 参数，以空格分隔
- 一些模块不带任何参数，而 command / shell 模块仅采用要运行的命令的字符串
- 所有模块均返回JSON格式数据
- 模块应是幂等的，并且如果模块检测到当前状态与所需的最终状态匹配，则应避免进行任何更改

查看指定模块文档：

```shell
ansible-doc [module]
```

查看所有模块列表：

```shell
ansible-doc -l
```

## 2.2、常用模块

| 模块 | 归类 | 用途 | 示例 |
| :- | :- | :- | :- |
| command | 指令 | 默认模块 | ansible 172.17.0.3 -a "/sbin/reboot" |
| shell | 指令 | 执行shell指令 | ansible 172.17.0.3 -m shell "echo $JAVA_HOME" |
| script | 指令 | 在远程机器执行shell脚本，脚本无需在远程机器上存在 | ansible 172.17.0.3 -m script -a '/deploy/server/startup.sh' |
| file | 文件 | 创建，删除文件（夹）及权限控制| ansible 172.17.0.3 -m file -a 'path=/home/deploy/downloads state=directory' |
| copy | 文件 | 复制本地文件到远程主机 | ansible 172.17.0.3 -m copy -a 'src=/deploy/version/server.tar dest=/deploy/server/server.tar' |
| fetch | 文件 | 从远程机器上面拉取文件到本地 | ansible 172.17.0.3 -m fetch -a 'src=/deploy/logs dest=/logs' |
| user | 用户 | 创建、删除用户 | ansible 172.17.0.3 -m user -a 'name=test password=123456' |
| group | 用户 | 创建、删除用户组 | ansible 172.17.0.3 -m group -a 'name=test' |
| yum | 安装 | 在centos系统上安装软件 | ansible 172.17.0.3 -m yum -a 'name=httpd state=installed' |
| service | 服务 | 服务的启动、关闭、开机自启 | ansible 172.17.0.3 -m service -a 'name=nginx state=started enabled=yes' |
| cron | 定时任务 | 在远程主机上指定定时任务 | ansible 172.17.0.3 -m cron -a "name='clear data' minute=* hour=* day=* month=* weekday=* job='/bin/bash /deploy/clean.sh'" |
| setup | 收集 | 获取系统信息 | ansible 172.17.0.3 -m setup |

command 和 shell 区别和总结：

- command 模块命令将不会使用 shell 执行. 因此, 像 $JAVA_HOME 这样的变量是不可用的。还有像<, >, |, ;, &都将不可用
- shell 模块通过shell程序执行， 默认是/bin/sh, <, >, |, ;, & 可用。但这样有潜在的 shell 注入风险
- 两个模块都要避免使用， 应该优先考虑更具体的 ansible 模块
- 如果没有更具体的模块， 相对来说 command 更安全点
- 如果您需要用户环境和流式操作，则只能使用 shell 模块
- 如果你需要安全的使用带有变量的 shell 模块， 使用{{ var | quote }} 代替 {{ var }} , 确保输入不包含分号或者流式操作

所有模块详细介绍请参见：[All modules](https://docs.ansible.com/ansible/latest/modules/list_of_all_modules.html)

## 2.3、ad-hoc 常用参数介绍

| 缩写参数 | 完整参数 | 说明 |
| :- | :- | :- |
| -v | --verbose | <font size=2>输出更详细的执行过程信息，-vvv可得到所有执行过程信息</font> |
| -i <font size=2>*PATH*</font> | --inventory=<font size=2>*PATH*</font> | <font size=2>指定inventory信息，默认/etc/ansible/hosts</font> |
| -f <font size=2>*NUM*</font> | --forks=<font size=2>*NUM*</font> | <font size=2>并发线程数，默认5个线程</font> |
| --private-key=<font size=2>*PRIVATE_KEY_FILE*</font> | | <font size=2>指定密钥文件</font> |
| -m <font size=2>*NAME*</font> | --module-name=<font size=2>*NAME*</font> | <font size=2>指定执行使用的模块</font> |
| -M <font size=2>*DIRECTORY*</font> | --module-path=<font size=2>*DIRECTORY*</font> | <font size=2>指定模块存放路径，默认/usr/share/ansible，也可以通过ANSIBLE_LIBRARY设定默认路径</font> |
| -a <font size=2>*'ARGUMENTS'*</font> | --args=<font size=2>*'ARGUMENTS'*</font> | <font size=2>模块参数</font> |
| -k | --ask-pass <font size=2>*SSH*</font> | <font size=2>认证密码</font> |
| -K | --ask-sudo-pass <font size=2>*sudo*</font> | <font size=2>用户的密码（—sudo时使用）</font> |
| -o | --one-line | <font size=2>标准输出至一行</font> |
| -s | --sudo | <font size=2>相当于Linux系统下的sudo命令</font> |
| -t <font size=2>*DIRECTORY*</font> | --tree=<font size=2>*DIRECTORY*</font> | <font size=2>输出信息至DIRECTORY目录下，结果文件以远程主机名命名</font> |
| -T <font size=2>*SECONDS*</font> | --timeout=<font size=2>*SECONDS*</font> | <font size=2>指定连接远程主机的最大超时，单位是：秒</font> |
| -B <font size=2>*NUM*</font> | --background=<font size=2>*NUM*</font> | <font size=2>后台执行命令，超NUM秒后kill正在执行的任务</font> |
| -P <font size=2>*NUM*</font> | --poll=<font size=2>*NUM*</font> | <font size=2>定期返回后台任务进度</font> |
| -u <font size=2>*USERNAME*</font> | --user=<font size=2>*USERNAME*</font> | <font size=2>指定远程主机以USERNAME运行命令</font> |
| -U <font size=2>*SUDO_USERNAME*</font> | --sudo-user=<font size=2>*SUDO_USERNAME*</font> | <font size=2>使用sudo，相当于Linux下的sudo命令</font> |
| -c <font size=2>*CONNECTION*</font> | --connection=<font size=2>*CONNECTION*</font> | <font size=2>指定连接方式，可用选项paramiko (SSH), ssh, local。Local方式常用于crontab 和 kickstarts</font> |
| -l <font size=2>*SUBSET*</font> | --limit=<font size=2>*SUBSET*</font> | <font size=2>指定运行主机</font> |
| -l <font size=2>*~REGEX*</font> | --limit=<font size=2>*~REGEX*</font> | <font size=2>指定运行主机（正则）</font> |
| --list-hosts | |<font size=2>列出符合条件的主机列表,不执行任何其他命令</font> |

## 3、官方文档参考

官方文档参考[Introduction to ad-hoc commands](https://docs.ansible.com/ansible/latest/user_guide/intro_adhoc.html)
