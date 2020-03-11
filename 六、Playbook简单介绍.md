# 六、Playbook简单介绍

## 1、什么是Playbook

Playbook是Ansible的配置，部署和编排语言。
Playbook可以描述远程系统执行的策略，或一般IT流程中的一组步骤。

Playbook结构：

- play: 单个剧本
- tasks: 具体执行的任务集合

一个Playbook由一个或多个play组成，一个play可以包含多个task，例如:

```yml
---
- hosts: webservers
  remote_user: root

  tasks:
    - name: ensure apache is at the latest version
      yum:
        name: httpd
        state: latest
    - name: write the apache config file
      template:
        src: /srv/httpd.j2
        dest: /etc/httpd.conf

- hosts: databases
  remote_user: root

  tasks:
    - name: ensure postgresql is at the latest version
      yum:
        name: postgresql
        state: latest
    - name: ensure that postgresql is started
      service:
        name: postgresql
        state: started
```

## 2、Playbook和ad-hoc区别

- Playbook功能比ad-hoc更全，支持丰富的配置语法，结构和逻辑控制
- 顺序执行，能够很好地控制前后执行的依赖关系
- 使用yml语法进行配置，展现更为直观
- 持久使用，可以保存为文件供以后使用
