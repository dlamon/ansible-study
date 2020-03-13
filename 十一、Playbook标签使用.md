# Playbook标签使用

## 1、标签基础用法

如果有一个大型的Playbook，如果只需要运行其中的特定部分而不是运行剧本中的所有内容，可以使用 tags：

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  tasks:
  - name: Upload new version
    debug:
      msg: "Upload new version"
  - name: Backup old version
    debug:
      msg: "Backup old version"
    tags: "backup"
  - name: Stop Service
    debug:
      msg: "Stop Service"
    tags:
    - "stop"
    - "restart"
  - name: Check Service is stop
    debug:
      msg: "Check Service is stop"
    tags: "restart"
  - name: Deploy new version
    debug:
      msg: "Deploy new version"
  - name: Start Service
    debug:
      msg: "Start Service"
    tags: "restart"
  - name: Check Service is start
    debug:
      msg: "Check Service is start"
    tags: "restart"
```

使用标签规则：

- 对一个对象打一个标签
- 对一个对象打多个标签
- 对多个对象打同一个标签
- 打标签的对象包括：单个 task任务， include 对象， roles 对象等

执行 Playbook 时，可以通过两种方式基于标签过滤任务：

- ansible-playbook 命令行，使用 *--tags* 或者 *--skip-tags* 选项
- Ansible配置中，使用 *TAGS_RUN* 和 *TAGS_SKIP* 选项（没用过）

```shell
ansible-playbook -i ./hosts ./test.yml --tags="restart"
```

```shell
ansible-playbook -i ./hosts ./test.yml --tags="backup,restart"
```

```shell
ansible-playbook -i ./hosts ./test.yml --skip-tags="backup"
```

可以使用 --list-tasks, 查看将会执行哪些任务：

```shell
ansible-playbook -i ./hosts ./test.yml --tags="restart" --list-tasks
```

## 2、标签继承

tags 标签可以用于任务以外的地方，tags 标签下的所有任务都会继承该标签。

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tags: task1
  tasks:
    - name: Run task 1.1
      debug:
        msg: "run task 1.1"
    - name: Run task 1.2
      debug:
        msg: "run task 1.2"
- hosts: 172.17.0.4
  remote_user: deploy
  gather_facts: no
  tags: task2
  tasks:
    - name: Run task 2.1
      debug:
        msg: "run task 2.1"
    - name: Run task 2.2
      debug:
        msg: "run task 2.2"
```

运行所有任务：

```shell
ansible-playbook -i ./hosts ./test.yml
```

可以使用 tags 分别运行第一组任务和第二组任务：

```shell
ansible-playbook -i ./hosts ./test.yml --tags="task1"
```

```shell
ansible-playbook -i ./hosts ./test.yml --tags="task2"
```

## 3、 在 block 中使用标签

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tasks:
    - block:
        - name: Run task 1.1
          debug:
            msg: "run task 1.1"
        - name: Run task 1.2
          debug:
            msg: "run task 1.2"
        tags: block1
    - name: Run other task
      debug:
        msg: "run other task"
```

运行 block 中的 task:

```shell
ansible-playbook -i ./hosts ./test.yml --tags="block1"
```

## 4、在 roles、import_role 和 import_tasks 中使用标签

```yml
roles:
  - role: webserver
    vars:
      port: 5000
    tags: [ web, foo ]
```

```yml
- import_role:
    name: myrole
  tags: [ web, foo ]
- import_tasks: foo.yml
  tags: [ web, foo ]
```

## 5、 特殊标签

如果需要始终运行该任务，可以使用 always 标记，除非明确跳过（--skip-tags always）

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tasks:
    - debug:
        msg: "Always runs"
      tags:
      - always
    - debug:
        msg: "run task1"
      tags:
      - task1
```

```shell
ansible-playbook -i ./hosts ./test.yml --tags="task1"
```

如果需要正常流程不执行该任务，只在特殊需要时执行，可以使用 never 标签：

```yml
tasks:
  - debug: msg="{{ showmevar }}"
    tags: [ never, debug ]
```

上述 task 仅在指定 never 或 debug 标签时执行。

另外还有三个特殊标签，如下：

| 标签 | 说明 |
| :- | :- |
| tagged | 运行有 tags 的任务 |
| untagged | 运行没有 tags 的任务 |
| all | 运行所有任务 |

## 6、官方文档参考

官方文档参考 [Tags](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html)
