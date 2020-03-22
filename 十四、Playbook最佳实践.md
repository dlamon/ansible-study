# 十四、Playbook最佳实践

## 1、目录布局

### 1.1、建议的目录布局

```yaml
production                # inventory file for production servers
staging                   # inventory file for staging environment

group_vars/
   group1.yml             # here we assign variables to particular groups
   group2.yml
host_vars/
   hostname1.yml          # here we assign variables to particular systems
   hostname2.yml

library/                  # if any custom modules, put them here (optional)
module_utils/             # if any custom module_utils to support modules, put them here (optional)
filter_plugins/           # if any custom filter plugins, put them here (optional)

site.yml                  # master playbook
webservers.yml            # playbook for webserver tier
dbservers.yml             # playbook for dbserver tier

roles/
    common/               # this hierarchy represents a "role"
        tasks/            #
            main.yml      #  <-- tasks file can include smaller files if warranted
        handlers/         #
            main.yml      #  <-- handlers file
        templates/        #  <-- files for use with the template resource
            ntp.conf.j2   #  <------- templates end in .j2
        files/            #
            bar.txt       #  <-- files for use with the copy resource
            foo.sh        #  <-- script files for use with the script resource
        vars/             #
            main.yml      #  <-- variables associated with this role
        defaults/         #
            main.yml      #  <-- default lower priority variables for this role
        meta/             #
            main.yml      #  <-- role dependencies
        library/          # roles can also include custom modules
        module_utils/     # roles can also include custom module_utils
        lookup_plugins/   # or other types of plugins, like lookup in this case

    webtier/              # same kind of structure as "common" was above, done for the webtier role
    monitoring/           # ""
    fooapp/               # ""
```

### 1.2、备用的目录布局

特点：

- 每个清单文件及其 group_vars / host_vars 放入到不同的文件夹，完全分离了不同环境的变量
- 如果 group_vars / host_vars 在不同环境中差异较大，建议使用备用目录布局
- 由于文件太多，难以维护

```yml
inventories/
   production/
      hosts               # inventory file for production servers
      group_vars/
         group1.yml       # here we assign variables to particular groups
         group2.yml
      host_vars/
         hostname1.yml    # here we assign variables to particular systems
         hostname2.yml

   staging/
      hosts               # inventory file for staging environment
      group_vars/
         group1.yml       # here we assign variables to particular groups
         group2.yml
      host_vars/
         stagehost1.yml   # here we assign variables to particular systems
         stagehost2.yml

library/
module_utils/
filter_plugins/

site.yml
webservers.yml
dbservers.yml

roles/
    common/
    webtier/
    monitoring/
    fooapp/
```

## 2、动态资产清单

如果使用云部署，建议使用[动态资产清单](https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html#intro-dynamic-inventory)。
如果没有使用云部署，如果有另一个系统维护系统列表(例如：CMDB)，也建议使用动态资产清单。

## 3、资产清单建议格式

建议根据主机的用途（角色）以及地理位置或数据中心位置来定义组：

```yml
# file: production

[atlanta_webservers]
www-atl-1.example.com
www-atl-2.example.com

[boston_webservers]
www-bos-1.example.com
www-bos-2.example.com

[atlanta_dbservers]
db-atl-1.example.com
db-atl-2.example.com

[boston_dbservers]
db-bos-1.example.com

# webservers in all geos
[webservers:children]
atlanta_webservers
boston_webservers

# dbservers in all geos
[dbservers:children]
atlanta_dbservers
boston_dbservers

# everything in the atlanta geo
[atlanta:children]
atlanta_webservers
atlanta_dbservers

# everything in the boston geo
[boston:children]
boston_webservers
boston_dbservers
```

## 4、组变量和主机变量

### 4.1、组变量

如果当前地区需要使用时间同步服务器地址，可以对地区组设置组变量：

```yml
---
# file: group_vars/atlanta
ntp: ntp-atlanta.example.com
backup: backup-atlanta.example.com
```

如果某一个应用需要设置应用服务器连接参数，可以对应用组设置组变量：

```yml
---
# file: group_vars/webservers
apacheMaxRequestsPerChild: 3000
apacheMaxClients: 900
```

如果有一些所有组都公用的变量，可以放在 group_vars/all 文件中：

```yml
---
# file: group_vars/all
ntp: ntp-boston.example.com
backup: backup-boston.example.com
```

### 4.2 主机变量

如果需要对某一台主机进行特定的参数配置，可以使用主机变量，但是不推荐使用：

```yml
---
# file: host_vars/db-bos-1.example.com
foo_agent_port: 86
bar_agent_port: 99
```

## 5、使用角色（role）构建 playbook

在 site.yml 文件中，需要定义整个顶层 playbook 的结构。
建议 site.yml 文件尽量精简，使用角色（role）机制分离不同角色的操作，仅在 site.yml 文件中进行导入。

```yml
---
# file: site.yml
- import_playbook: webservers.yml
- import_playbook: dbservers.yml
```

```yml
---
# file: webservers.yml
- hosts: webservers
  roles:
    - common
    - webtier
```

这样配置的好处：

- 可以选择直接运行整个 playbook
- 可以选择运行 playbook 中的子集

运行整个 playbook：

```shell
ansible-playbook site.yml
```

运行 playbook 子集：

```shell
ansible-playbook site.yml --limit webservers
```

或

```shell
ansible-playbook webservers.yml
```

## 6、角色（role）中的 task 和 handler

```yml
---
# file: roles/common/tasks/main.yml
- name: be sure ntp is installed
  yum:
    name: ntp
    state: present
  tags: ntp

- name: be sure ntp is configured
  template:
    src: ntp.conf.j2
    dest: /etc/ntp.conf
  notify:
    - restart ntpd
  tags: ntp

- name: be sure ntpd is running and enabled
  service:
    name: ntpd
    state: started
    enabled: yes
  tags: ntp
```

```yml
---
# file: roles/common/handlers/main.yml
- name: restart ntpd
  service:
    name: ntpd
    state: restarted
```

建议在角色中增加 tags，比如 stop 和 start 等，以便在后期进行维护：

```shell
ansible-playbook webservers.yml --tags="stop"
```

## 7、使用示例

```shell
ansible-playbook -i production site.yml
```

```shell
ansible-playbook -i production site.yml --tags ntp
```

```shell
ansible-playbook -i production webservers.yml
```

```shell
ansible-playbook -i production webservers.yml --limit boston
```

```shell
ansible-playbook -i production webservers.yml --limit boston[0:9]
ansible-playbook -i production webservers.yml --limit boston[10:19]
```

```shell
ansible boston -i production -m ping
ansible boston -i production -m command -a '/sbin/reboot'
```

```shell
ansible-playbook -i production webservers.yml --tags ntp --list-tasks

# confirm what hostnames might be communicated with if I said "limit to boston"
ansible-playbook -i production webservers.yml --limit boston --list-hosts

```

## 8、测试环境和生产环境

测试环境和生产环境建议使用不同的资产清单文件进行配置。
在执行时使用 -i 指定不同环境的资产清单文件。
测试环境和生产环境的差异性可以使用组变量（group variables）配置去控制。

## 9、滚动升级

如果升级的是需要提供24小时服务的系统，建议使用滚动升级策略，尽量做到服务不中断或者尽可能减少系统升级过程中导致的服务中断时间。
滚动升级需要充分考虑到各中心机房机器数量，服务提供方式，负载均衡配置等多方面的因素。
如果升级的是不需要提供24小时服务的系统，可以不采用滚动升级策略。

## 10、明确state

目前官方提供的模块中很多都支持 state 选项，建议将 state 选项保留在 playbook 中并明确说明。

## 11、按角色分组

官方再次强调了按角色分组的重要性 -_-!!

## 12、空格和注释

建议使用空格分解内容，多使用注释进行解释（#）。

## 13、任务命名

建议给任务命名，在运行 playbook 时可以清楚看到任务执行过程。

## 14、简单化

建议尽量将 playbook 简单化，清晰易懂。

## 15、版本控制

建议将 playbook 和资产清单文件保存在git（或其他版本控制系统）中，并在进行更改时提交。
可以审计跟踪，了解何时以及为何更改。

## 16、敏感数据加密

参见官方文档 [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
