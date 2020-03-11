# 十二、Playbook角色介绍

## 1、什么是 角色（roles）

角色（roles）用于层次性，结构化地组织 playbook。
roles 能够根据层次型结构自动装载 vars_files、tasks 以及 handlers 等。

例如：如果需要部署一个系统，该系统采用前后端分离的框架进行开发，包含一个独立的前端应用（font），一个独立的后端应用（server）和一个消息队里（MQ），那么我们可以抽象化出三个角色，分别为 font，server 和 MQ。
针对不同的角色，编写不同的部署脚本，按照一定的层次关系整合在一个 Playbook 中。

角色（roles）的概念体现了面向对象的思想。

## 2、角色的目录结构

 ```tree
site.yml
webservers.yml
fooservers.yml
roles/
    common/
        tasks/
        handlers/
        files/
        templates/
        vars/
        defaults/
        meta/
    webservers/
        tasks/
        defaults/
        meta/
 ```

使用时，每个目录必须包含一个main.yml文件，其包含的内容为：

| 目录名 | main 文件说明 |
| :- | :- |
| tasks | 包含角色要执行的任务的主要列表 |
| handlers | 包含处理程序 |
| defaults | 角色的默认变量 |
| vars | 角色的其他变量 |
| files | 角色需要部署的文件 |
| templates | 角色部署的模板 |
| meta | 定义角色之间的依赖 |

## 3、角色的引用和执行

### 3.1、roles 语句

可以使用roles语句引用多个role。

#### 3.1.1、示例

文件结构：

```yml
.
├── hosts
├── roles
│   ├── font
│   │   └── tasks
│   │       └── main.yml
│   └── server
│       └── tasks
│           └── main.yml
└── site.yml
```

site.yml文件：

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  roles:
    - font
    - server
```

roles/font/tasks/main.yml

```yml
---
- name: font task 1
  debug:
    msg: "font task 1"
- name: font task 2
  debug:
    msg: "font task 2"
```

roles/server/tasks/main.yml

```yml
---
- name: server task 1
  debug:
    msg: "server task 1"
- name: server task 2
  debug:
    msg: "server task 2"
```

执行 shell 如下：

```shell
ansible-playbook -i ./hosts ./site.yml
```

### 3.1.2 加入 vars

文件结构：

```yml
.
├── hosts
├── roles
│   ├── font
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── vars
│   │       └── main.yml
│   └── server
│       ├── tasks
│       │   └── main.yml
│       └── vars
│           └── main.yml
└── site.yml
```

site.yml文件：

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  roles:
    - font
    - server
```

roles/font/tasks/main.yml

```yml
---
- name: font task 1
  debug:
    msg: "font task 1"
- name: echo port
  debug:
    msg: "{{ font_port }}"
```

roles/font/vars/main.yml

```yml
---
font_port: 8080
```

roles/server/tasks/main.yml

```yml
---
- name: server task 1
  debug:
    msg: "server task 1"
- name: echo port
  debug:
    msg: "{{ server_port }}"
```

roles/server/vars/main.yml

```yml
---
server_port: 20080
```

执行 shell 如下：

```shell
ansible-playbook -i ./hosts ./site.yml
```

-*注意：如果变量名称有重复，不论是否在不同的 roles/vars 文件夹，都会使用后一个变量值覆盖前一个变量值*

### 3.1.3、加入handlers

文件结构：

```yml
.
├── hosts
├── roles
│   ├── font
│   │   ├── handlers
│   │   │   └── main.yml
│   │   └── tasks
│   │       └── main.yml
│   └── server
│       ├── handlers
│       │   └── main.yml
│       └── tasks
│           └── main.yml
└── site.yml
```

site.yml文件：

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  roles:
    - font
    - server
```

roles/font/tasks/main.yml

```yml
---
- name: font task 1
  shell: "cat ~/test.log"
  notify:
    - restart font
```

roles/font/handlers/main.yml

```yml
---
- name: restart font
  debug:
    msg: "restart font handler"
```

roles/server/tasks/main.yml

```yml
---
- name: server task 1
  shell: "cat ~/test.log"
  notify:
    - restart server
```

roles/server/handlers/main.yml

```yml
---
- name: restart server
  debug:
    msg: "restart server handler"
```

执行 shell 如下：

```shell
ansible-playbook -i ./hosts ./site.yml
```

### 3.1.4、加入 meta

假定 server 需要在 db 完成部署后才能部署。

文件结构：

```yml
.
├── hosts
├── roles
│   ├── db
│   │   └── tasks
│   │       └── main.yml
│   └── server
│       ├── meta
│       │   └── main.yml
│       └── tasks
│           └── main.yml
└── site.yml
```

site.yml文件：

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  roles:
    - server
```

roles/db/tasks/main.yml

```yml
---
- name: db task
  debug:
    msg: "db task"
```

roles/server/tasks/main.yml

```yml
---
- name: server task
  debug:
    msg: "server task"
```

roles/server/meta/main.yml

```yml
---
dependencies:
  - role: db
```

执行 shell 如下：

```shell
ansible-playbook -i ./hosts ./site.yml
```

### 3.1.5、执行顺序

文件结构：

```yml
.
├── hosts
├── roles
│   ├── font
│   │   └── tasks
│   │       └── main.yml
│   └── server
│       └── tasks
│           └── main.yml
└── site.yml
```

site.yml文件：

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  pre_tasks:
    - name: pre task
      shell: "cat ~/test.log"
      notify:
        - pre task handler
  roles:
    - font
    - server
  tasks:
    - name: normal task
      shell: "cat ~/test.log"
      notify:
        - normal task handler
  post_tasks:
    - name: post task
      shell: "cat ~/test.log"
      notify:
        - post task handler
  handlers:
    - name: pre task handler
      debug:
        msg: "pre task handler"
    - name: normal task handler
      debug:
        msg: "normal task handler"
    - name: post task handler
      debug:
        msg: "post task handler"
```

roles/font/tasks/main.yml

```yml
---
- name: font task
  debug:
    msg: "font task"
```

roles/server/tasks/main.yml

```yml
---
- name: server task
  debug:
    msg: "server task"
```

执行 shell 如下：

```shell
ansible-playbook -i ./hosts ./site.yml
```

从结果分析，playbook 执行顺序如下：

- playbook 中定义的所有pre_tasks
- 被 pre_tasks 触发的 handlers
- 执行 roles 中定义的 role 对应的 task (不考虑meta)
- playbook 中定义的所有 tasks
- 被 tasks 触发的 handlers
- playbook 中定义的所有 post_tasks
- 被 post_tasks 触发的 handlers

### 3.2、import_role 和 include_role

从 Ansible 2.4 开始，可以使用 import_role 或 include_role 将角色与其他任何任务内联使用。

#### 3.2.1、示例

文件结构：

```yml
.
├── hosts
├── roles
│   ├── font
│   │   └── tasks
│   │       └── main.yml
│   └── server
│       └── tasks
│           └── main.yml
└── site.yml
```

site.yml 文件：

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tasks:
    - debug:
        msg: "before we run our role"
    - import_role:
        name: font
    - include_role:
        name: server
    - debug:
        msg: "after we ran our role"
```

roles/font/tasks/main.yml

```yml
---
- name: font task
  debug:
    msg: "font task"
```

roles/server/tasks/main.yml

```yml
---
- name: server task
  debug:
    msg: "server task"
```

执行 shell 如下：

```shell
ansible-playbook -i ./hosts ./site.yml
```

#### 3.2.2、和 When 一起使用

文件结构：

```yml
.
├── hosts
├── roles
│   ├── font
│   │   └── tasks
│   │       └── main.yml
│   └── server
│       └── tasks
│           └── main.yml
└── site.yml
```

site.yml 文件：

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tasks:
    - import_role:
        name: font
      when: target == "font" or target == "all"
    - include_role:
        name: server
      when: target == "server" or target == "all"
```

roles/font/tasks/main.yml

```yml
---
- name: font task
  debug:
    msg: "font task"
```

roles/server/tasks/main.yml

```yml
---
- name: server task
  debug:
    msg: "server task"
```

执行 shell 如下：

```shell
ansible-playbook -i ./hosts ./site.yml -e "target=font"
```

```shell
ansible-playbook -i ./hosts ./site.yml -e "target=server"
```

```shell
ansible-playbook -i ./hosts ./site.yml -e "target=all"
```

#### 3.2.3、和 tags 一起使用

文件结构：

```yml
.
├── hosts
├── roles
│   ├── font
│   │   └── tasks
│   │       └── main.yml
│   └── server
│       └── tasks
│           └── main.yml
└── site.yml
```

site.yml 文件：

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tasks:
    - import_role:
        name: font
      tags: font
    - import_role:
        name: server
      tags: server
```

roles/font/tasks/main.yml

```yml
---
- name: font task
  debug:
    msg: "font task"
```

roles/server/tasks/main.yml

```yml
---
- name: server task
  debug:
    msg: "server task"
```

执行 shell 如下：

```shell
ansible-playbook -i ./hosts ./site.yml --tags="font"
```

```shell
ansible-playbook -i ./hosts ./site.yml --tags="server"
```

- *注意：使用 tags 时需要特别注意，必须使用 import_role，不能使用 include_role*

### 3.3 Dynamic vs Static

Ansible 支持两种导入方式：动态（Dynamic）和静态（Static）。

- 如果使用 include* （include_tasks, include_role, etc.），就是动态导入
- 如果使用 import* （import_tasks, import_role, etc.），就是静态导入

两种导入方式的原理：

- 动态导入在运行到该任务时进行导入
- 静态导入在 Playbook 预处理期进行导入

在 Ansible 2.3 版本之前，角色（roles）都是通过特殊的 roles: 选项进行导入，并且第一个执行，除非设置 pre_tasks。
在 Ansible 2.3 版本之后，角色（roles）仍然可以通过 roles: 选项进行导入，但是 Ansible 2.3 引入了 include_role 来和 task 配合使用。

include* 和 import* 之间的权衡：

使用 include* 的缺点：

- 使用 --list-tags 无法列出被导入内容的 tags
- 使用 --list-tasks 无法列出被导入内容的 tasks
- 不能使用 notify 去触发在被导入内容的 handler
- 不能使用 --start-at-task 指定被导入内容的 task

使用 import* 的缺点：

- 循环（loop）不能和 import* 一起使用
- import* 方法调用的文件名称不可以有变量
- import* 会覆盖同名的 vars 和 handlers

### 4、官方文档参考

官方文档参考 [Roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)
