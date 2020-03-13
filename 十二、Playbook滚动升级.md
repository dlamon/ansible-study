# 十二、Playbook 滚动升级

## 1、设置滚动更新批次大小

### 1.1、设置数值

可以通过 serial 设置滚动更新批次大小：

```ini
# file: hosts
172.17.0.3
172.17.0.4
172.17.0.5
```

```yml
---
# file: test.yml
- hosts: all
  remote_user: deploy
  gather_facts: no
  serial: 2
  tasks:
    - name: stop
      debug:
        msg: "stop"
    - name: update
      debug:
        msg: "update"
    - name: start
      debug:
        msg: "start"
```

### 1.2、设置百分比

serial 可以设置百分比，如果主机数量不能均分，则最后一次包含剩余的主机。

```yml
---
# file: test.yml
- hosts: all
  remote_user: deploy
  gather_facts: no
  serial: "30%"
  tasks:
    - name: stop
      debug:
        msg: "stop"
    - name: update
      debug:
        msg: "update"
    - name: start
      debug:
        msg: "start"
```

### 1.3、设置列表

serial 在 ansible 2.2 版本后，还支持使用列表配置滚动更新批次大小：

```yml
---
# file: test.yml
- hosts: all
  remote_user: deploy
  gather_facts: no
  serial: [2, 1]
  tasks:
    - name: stop
      debug:
        msg: "stop"
    - name: update
      debug:
        msg: "update"
    - name: start
      debug:
        msg: "start"
```

上述例子中，如果将 serial: [2, 1] 更改为 serial: [1, 1]，最后的一台主机 172.17.0.5 仍然会执行。

列表中除了可以设置数值外，也可以设置百分比：

```yml
---
# file: test.yml
- hosts: all
  remote_user: deploy
  gather_facts: no
  serial: ["40%", "60%"]
  tasks:
    - name: stop
      debug:
        msg: "stop"
    - name: update
      debug:
        msg: "update"
    - name: start
      debug:
        msg: "start"
```

### 1.4、混合使用

serial 可以同时使用数值和百分比配置：

```yml
---
# file: test.yml
- hosts: all
  remote_user: deploy
  gather_facts: no
  serial:
    - 1
    - 20%
    - 80%
  tasks:
    - name: stop
      debug:
        msg: "stop"
    - name: update
      debug:
        msg: "update"
    - name: start
      debug:
        msg: "start"
```

## 2、设置最大故障率

可以使用 max_fail_percentage 设置最大故障率：

```yml
---
# file: test.yml
- hosts: all
  remote_user: deploy
  gather_facts: no
  max_fail_percentage: 49
  serial: 2
  tasks:
    - name: cat test.log
      shell: "cat ~/test.log"
```

-*注意：最大故障率的百分比是必须超过，而不是相等。*
-*例如，如果将serial设置为2，并且您希望当其中一个系统出现故障时任务中止，则该百分比应设置为 49，而不是 50。*

## 3、任务委派

可以使用 delegate_to 进行任务委派，指定一台主机运行该任务，如果存在多台主机，则任务将执行多次：

```yml
---
# file: test.yml
- hosts: all
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: echo test.log
      shell: "echo '123' >> ~/test.log"
      delegate_to: 172.17.0.3
```

## 4、运行一次

如果需要为一批主机只运行一次任务，可以使用 run_once，run_once 默认在当前批次的第一台主机上执行：

```yml
---
# file: test.yml
- hosts: all
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: echo test.log
      shell: "echo '123' >> ~/test.log"
      run_once: true
```

如果需要制定特定的主机执行 run_once，可以和 delegate_to 配合使用：

```yml
---
# file: test.yml
- hosts: all
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: echo test.log
      shell: "echo '123' >> ~/test.log"
      run_once: true
      delegate_to: 172.17.0.4
```

上述功能也可以通过 when 实现：

```yml
---
# file: test.yml
- hosts: all
  remote_user: deploy
  gather_facts: no
  tasks:
    - name: echo test.log
      shell: "echo '123' >> ~/test.log"
      when: inventory_hostname == "172.17.0.4"
```

## 5、本地剧本

如果 playbook 仅仅需要在执行 ansible-playbook 命令的主机上执行，可以使用本地剧本配置：

```yml
---
# file: test.yml
- hosts: 127.0.0.1
  gather_facts: no
  connection: local
  tasks:
    - name: echo test.log
      shell: "echo '123' >> ~/test.log"
```

本地剧本一般用于本机 crontab 定时任务场景。
