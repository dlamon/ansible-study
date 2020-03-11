# 九、Playbook循环控制

## 1、版本说明

官方文档介绍：

```output
We added loop in Ansible 2.5. It is not yet a full replacement for with_<lookup>, but we recommend it for most use cases.
We have not deprecated the use of with_<lookup> - that syntax will still be valid for the foreseeable future.
```

## 2、标准循环

### 2.1、遍历简单列表

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  vars:
    data: [1, 2, 3, 4, 5]
  tasks:
    - name: "loop"
      debug:
        msg: "{{ item }}"
      loop: "{{ data }}"
```

有的模块参数支持列表形式（例如 yum, apt等），如果模块参数支持列表，则将列表传递给参数比遍历任务更好，例如：

```yml
- name: optimal yum
  yum:
    name: "{{  list_of_packages  }}"
    state: present
```

### 2.2、遍历哈希表

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  vars:
    data:
    - { id: 1, name: user1 }
    - { id: 2, name: user2 }
    - { id: 3, name: user3 }
  tasks:
    - name: "loop"
      debug:
        msg: "id: {{ item.id }} & name: {{ item.name }}"
      loop: "{{ data }}"
```

## 3、使用循环注册变量

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tasks:
    - shell: "echo {{ item }}"
      loop:
        - "one"
        - "two"
      register: echo
    - debug:
        msg: "{{ echo }}"
    - name: Fail if return code is not 0
      fail:
        msg: "The command ({{ item.cmd }}) did not have a 0 return code"
      when: item.rc != 0
      loop: "{{ echo.results }}"
```

## 4、重试任务

可以使用 until 关键字重试任务，直到满足特定条件为止。

```yml
- shell: /usr/bin/foo
  register: result
  until: result.stdout.find("all systems go") != -1
  retries: 5
  delay: 10
```

上述任务最多运行5次，每次尝试之间要延迟 10 秒。
如果任何尝试的结果在其stdout中显示 “all system go” ，则任务成功。
until retries 的默认值为 3，delay 为 5。

## 5、循环内的控制

### 5.1 循环内暂停

如果需要控制任务循环中每个项目执行之间的时间（以秒为单位），可以使用带有 loop_control 的 pause 指令。

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tasks:
    - debug:
      msg: "{{ item }}"
      loop: [1, 2, 3]
      loop_control:
        pause: 3
```

### 5.2、循环下标

如果需要获取当前循环的下标，可以使用带有 loop_control 的 index_var 指令。

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tasks:
    - debug:
        msg: "{{ item }} with index {{ item_index }}"
      loop: [1, 2, 3]
      loop_control:
        index_var: item_index
```

## 6、循环和判断混合使用

示例1：

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  tasks:
    - debug:
        msg: "{{ item }} with index {{ item_index }}"
      loop: [1, 2, 3]
      loop_control:
        index_var: item_index
      when: item_index > 1
```

示例2：

```yml
---
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  vars:
    data:
    - { id: 1, name: user1 }
    - { id: 2, name: user2 }
    - { id: 3, name: user3 }
  tasks:
    - name: "loop"
      debug:
        msg: "id: {{ item.id }} & name: {{ item.name }}"
      loop: "{{ data }}"
      when: item.id == 2
```

## 7、旧循环语句列表

循环语句 | 说明
- | -
with_items | 标准循环
with_dict | 遍历字典
with_fileglob | 遍历目录
with_nested | 嵌套循环
with_together | 并行遍历列表
with_indexed_items | 遍历列表和索引
with_file | 遍历文件列表的内容
with_first_found | 查找第一个匹配文件
with_random_choice | 随机选择
with_sequence | 在序列中循环
until | 循环重试

## 8、官方文档参考

官方文档参考 [Loops](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#)
