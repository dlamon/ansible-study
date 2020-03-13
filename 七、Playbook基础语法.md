# Playbook基础语法

## 1、Playbook命令

```shell
ansible-playbook xxx.yml [options]
```

说明：

- *xxx.yml* : 自定义的 Playbook yaml 配置文件
- *options* : 命令行参数
  
执行结果共有三种颜色的信息：

- 红色：表示有 task 执行失败或有提示的信息
- 黄色：表示执行了并且改变了远程主机状态
- 绿色：表示执行成功

详细的 options 参数如下表：
| 缩写参数 | 完整参数 | 说明 |
| :- | :- | :- |
| -u <font size=2>*REMOTE_USER*</font> | --user=<font size=2>*REMOTE_USER*</font> | <font size=2>ssh 连接的用户名</font> |
| -k | --ask-pass | <font size=2>ssh 登录认证密码</font> |
| -s | --sudo |  <font size=2>sudo 到root用户</font> |
| -U <font size=2>*SUDO_USER*</font> | --sudo-user=<font size=2>*SUDO_USER*</font> | <font size=2>sudo 到对应的用户</font> |
| -K | --ask-sudo-pass | <font size=2>用户的密码（-sudo时使用)</font> |
| -T <font size=2>*TIMEOUT*</font> | --timeout=<font size=2>*TIMEOUT*</font> | <font size=2>ssh 连接超时，默认 10 秒</font> |
| -C | --check | <font size=2>指定该参数后，执行 playbook 文件不会真正去执行，而是模拟执行一遍，然后输出本次执行会对远程主机造成的修改</font> |
| -e <font size=2>*EXTRA_VARS*</font> | --extra-vars=<font size=2>*EXTRA_VARS*</font> | <font size=2>设置额外的变量如：key=value 形式 或者 YAML or JSON，以空格分隔变量，或用多个-e</font> |
| -f <font size=2>*FORKS*</font> | --forks=<font size=2>*FORKS*</font> | <font size=2>进程并发处理，默认 5</font> |
| -i <font size=2>*INVENTORY*</font> | --inventory-file=<font size=2>*INVENTORY* | <font size=2>指定 hosts 文件路径，默认 /etc/ansible/hosts</font> |
| -l <font size=2>*SUBSET*</font> | --limit=<font size=2>*SUBSET*</font> | <font size=2>指定一个 pattern，对- hosts:匹配到的主机再过滤一次</font> |
| - | --list-hosts | <font size=2>输出所属主机列表</font> |
| - | --list-tasks | <font size=2>输出所属任务列表</font> |
| - | --private-key=<font size=2>*PRIVATE_KEY_FILE* | <font size=2>私钥路径</font> |
| - | --step | <font size=2>同一时间只执行一个 task，每个 task 执行前都会提示确认一遍</font> |
| - | --syntax-check | <font size=2>只检测 playbook 文件语法是否有问题，不会执行该 playbook</font> |
| -t <font size=2>*TAGS* | --tags=<font size=2>*TAGS* | <font size=2>当 play 和 task 的 tag 为该参数指定的值时才执行，多个 tag 以逗号分隔</font> |
| - | --skip-tags=<font size=2>*SKIP_TAGS* | <font size=2>当 play 和 task 的 tag 不匹配该参数指定的值时，才执行</font> |
| -v | --verbose | <font size=2>输出更详细的执行过程信息，-vvv可得到所有执行过程信息</font> |

 如果需要了解 yaml 语法格式，请参考 [YAML Syntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)

## 2、Playbook yml

### 2.1、主机和用户

#### 2.1.1、配置

```yml
---
# file: test.yml
- hosts: webservers
  remote_user: root
```

hosts：是一个或多个组或主机模式的列表，以冒号分隔。组和主机模式配置可以参考《四、资产清单文件详解》。
remote_user：登录远程机器的用户。

#### 2.1.2、主机执行顺序

如果在资产清单文件中配置了多台主机，默认按照在清单文件中配置的顺序依次执行。
可以使用order关键字对主机执行顺序进行配置。

```yml
---
# file: test.yml
- hosts: all
  order: sorted
```

order 配置项如下：

- inventory：默认值，按照资产清单文件中定义的顺序执行
- reverse_inventory：反向按照资产清单文件中定义的顺序执行
- sorted：按照主机名称的字母顺序排序执行
- reverse_sorted：反向按照主机名称的字母顺序排序执行
- shuffle： 按照随机的顺序执行

#### 2.1.3、主机用户切换

remote_user 可以定义在每个任务中，以便在不同任务中切换用户：

```yml
---
# file: test.yml
- hosts: webservers
  remote_user: root
  tasks:
    - name: test connection
      ping:
      remote_user: yourname
```

become 也可以用来进行用户切换,将 deploy 用户切换为root用户：

```yml
---
# file: test.yml
- hosts: webservers
  remote_user: deploy
  become: yes
```

become 可以使用 become_method 指定权限切换方式（su，sudo）：

```yml
---
# file: test.yml
- hosts: webservers
  remote_user: deploy
  tasks:
    - service:
        name: nginx
        state: started
      become: yes
      become_method: sudo
```

become 可以使用 become_user 指定切换后的用户（不指定默认切换到root）：

```yml
---
# file: test.yml
- hosts: webservers
  remote_user: deploy
  become: yes
  become_user: postgres
```

需要特别注意： 如果在权限切换过程中，需要为 sudo, su 指定密码，在运行ansible-play 命令行时需要带 --ask-become-pass 或 -K 参数运行。

### 2.2、任务列表

任务列表（tasks）从上到下顺序执行。任务失败的主机将从整个Play中删除。

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  tasks:
    - name: make sure apache is running
      service:
        name: httpd
        state: started
    - name: create test log
      shell: "touch ~/test.log"
```

- name： 每个任务都应有一个名称，该名称将在运行剧本过程中输出到控制台
- module:options： 指定任务的实际操作，由模块名和选项构成，选项采用 key = value 格式
- command 和 shell 模块仅有的一组参数，不需要采用 key = value 格式

### 2.3、使用变量

#### 2.3.1、Playbook yaml 文件中定义变量进行赋值

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  vars:
    file_name: test.log
  tasks:
    - name: create test log
      shell: "touch ~/{{file_name}}"
```

变量定义在 yaml 文件 vars 关键字下。
引用变量使用 {{ }}。

#### 2.3.2、通过-e 或 --extra-vars 对变量进行赋值

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: create test log
      shell: "touch ~/{{file_name}}"
```

ansible-playbook 命令:

```shell
ansible-playbook -i ./test.yml -e "file_name=test.log"
```

#### 2.3.3、通过资产清单文件定义变量进行赋值

hosts 资产清单文件：

```ini
# file: hosts
172.17.0.3 file_name=test03.log
172.17.0.4 file_name=test04.log
```

playbook 文件：

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: create test log
      shell: "touch ~/{{file_name}}"
```

ansible-playbook 命令:

```shell
ansible-playbook -i ./hosts ./test.yml
```

如果资产清单以组的方式配置，则配置变量如下：

```ini
# file: hosts
[sc]
172.17.0.3
172.17.0.4
[sc:vars]
file_name=test.log
```

使用上述 Playbook 的输出内容中会出现 WARNING ，提示应该使用 file 模块来完成操作。
可以修改为以下内容：

```yml
---
# file: test.yml
- hosts: sc
  remote_user: deploy
  vars:
    file_name: test.log
  tasks:
    - name: create test log
      file:
        path: ~/{{file_name}}
        state: touch
```

#### 2.3.4、通过注册变量对变量进行赋值

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: get time
      shell: "date"
      register: date_output
    - name: echo output
      shell: "echo {{date_output.stdout}} > test.log"
```

### 2.4、pre_tasks 和 post_tasks

pre_tasks 在所有任务执行前执行。
post_tasks 在所有任务执行后执行。

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  pre_tasks:
    - name: pre task
      debug:
        msg: "pre task"
  tasks:
    - name: task
      debug:
        msg: "task"
    #- name: failure
      #command: /bin/false
  post_tasks:
    - name: post task
      debug:
        msg: "post task"
```

### 2.5、handlers

handlers 知识点：

- handlers 也是一些 task 的列表，通过名字来引用，它们和一般的 task 并没有什么区别。
- handlers 是由通知者进行 notify，如果没有被 notify，handlers 不会执行。
- 在执行 notify 任务时，该任务的状态需要是 changed 时，才能进入 handlers。
- 不管有多少个通知者进行了 notify，等到 play 中的所有 task 执行完成之后，handlers 也只会被执行一次。
- handlers 最佳的应用场景是用来重启服务，或者触发系统重启操作。

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: test task
      shell: "cat ~/test.log"
      notify:
        - restart
  handlers:
    - name: restart
      debug:
        msg: "restart handlers"
```

## 3、官方文档参考

官方文档参考：
[Intro to Playbooks](https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html)
[Using Variables](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html)
