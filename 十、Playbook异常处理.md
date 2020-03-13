# 十、Playbook错误处理

## 1、忽略失败的任务

Playbook 默认会停止在任务失败的主机上执行任何其他步骤。
如果需要忽略错误，可以使用 ignore_errors: yes ：

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: test ignore_errors
      shell: "cat ~/test.log"
      register: result
      ignore_errors: yes
    - debug:
        msg: "{{ result }}"
```

## 2、自定义错误

如果任务执行成功，但是需要根据自定义返回值判断是否有错误，可以使用 failed_when ：

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: test failed_when
      shell: "cat ~/test.log"
      register: result
      failed_when: result.stdout == "abc"
    - debug:
        msg: "{{ result }}"
```

也可以使用 fail 和 when 组合：

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: test failed_when
      shell: "cat ~/test.log"
      register: result
    - fail:
        msg: "result is not abc"
      when: result.stdout != "abc"
    - debug:
        msg: "{{ result }}"
```

## 3、自定义Change状态

如果需要基于返回码或输出知道它没有进行任何更改，并希望覆盖“已更改”的结果，以使其不出现在输出中或不会导致处理程序触发，则可以使用 changed_when ：

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: test changed_when
      shell: "cat ~/test.log"
      register: result
      changed_when: result.stdout == "abc"
    - debug:
        msg: "{{ result }}"
```

## 4、中止任务

如果需要在任何一台主机上发生错误，中止整个任务列表继续执行，可以使用 any_errors_fatal 。
any_errors_fatal 可以用于单个任务，也可以用于整个任务列表：

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: test any_errors_fatal
      shell: "cat ~/test.log"
      register: result
      any_errors_fatal: true
    - debug:
        msg: "{{ result }}"
```

或者：

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  any_errors_fatal: true
  tasks:
    - name: test any_errors_fatal
      shell: "cat ~/test.log"
      register: result
    - debug:
        msg: "{{ result }}"
```

可以使用 max_fail_percentage 进行更细粒度的控制，在给定百分比的主机发生故障后中止任务运行。

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  max_fail_percentage: 60
  tasks:
    - name: test any_errors_fatal
      shell: "cat ~/test.log"
      register: result
    - debug:
        msg: "{{ result }}"
```

## 5、使用块

使用 block 和 rescue 可以完成类似于 try catch 的功能：

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: Handle the error
      block:
        - name: "test shell"
          shell: "cat ~/test.log"
          register: result
        - debug:
            msg: "{{ result }}"
        rescue:
        - debug:
            msg: "caught an error"
```

使用 block、rescue 和 always 可以完成类似于 try catch finally 的功能：

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: Handle the error
      block:
        - name: "test shell"
          shell: "cat ~/test.log"
          register: result
        - debug:
            msg: "{{ result }}"
      rescue:
        - debug:
            msg: "caught an error"
      always:
        - debug:
            msg: "do always"
```

block 还可以和 when 一起使用，判断多个符合条件的任务：

```yml
---
# file: test.yml
- hosts: 172.17.0.3,172.17.0.4
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: "test shell"
      shell: "cat ~/test.log"
      register: result
    - name: test block & when
      block:
      - debug:
          msg: "{{ result }}"
      - debug:
          msg: "do something else"
      when: result.stdout == "abc123"
```

## 6、官方文档参考

官方文档参考 [Error Handling In Playbooks](https://docs.ansible.com/ansible/latest/user_guide/playbooks_error_handling.html)
