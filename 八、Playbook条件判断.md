# 八、Playbook条件判断

## 1、运算符

在Ansible Playbook中，使用 when 关键字进行条件判断。
when 关键字后面的变量不需要使用 {{ }}。

```yml
tasks:
  - name: "shut down Debian flavored systems"
    command: /sbin/shutdown -t now
    when: ansible_facts['os_family'] == "Debian"
    # note that all variables can be used directly in conditionals without double curly braces
```

when 条件可以进行组合，可以使用关键字 or 或者 and

```yml
tasks:
  - name: "shut down CentOS 6 and Debian 7 systems"
    command: /sbin/shutdown -t now
    when: (ansible_facts['distribution'] == "CentOS" and ansible_facts['distribution_major_version'] == "6") or
          (ansible_facts['distribution'] == "Debian" and ansible_facts['distribution_major_version'] == "7")
```

也可以使用列表的方式表示 and 关键字：

```yml
tasks:
  - name: "shut down CentOS 6 systems"
    command: /sbin/shutdown -t now
    when:
      - ansible_facts['distribution'] == "CentOS"
      - ansible_facts['distribution_major_version'] == "6"
```

如果需要对字符串值进行数值的比较判断，可以使用 *|int* 进行转换：

```yml
tasks:
  - shell: echo "only on Red Hat 6, derivatives, and later"
    when: ansible_facts['os_family'] == "RedHat" and ansible_facts['lsb']['major_release']|int >= 6
```

如果需要对特定字符串（‘yes’, ‘on’, ‘1’, ‘true’）进行布尔值判断，可以使用 *|bool* 进行转换，例如：

```yml
vars:
  epic: true
  monumental: "yes"
tasks:
  - shell: echo "This certainly is epic!"
    when: epic or monumental|bool
```

也可以使用 not 关键字进行布尔判断，例如：

```yml
tasks:
  - shell: echo "This certainly isn't epic!"
    when: not epic
```

运算符如下：
| 运算符 | 类型 | 说明 |
| :- | :- | :- |
|==|比较运算符|比较两个对象是否相等，相等则返回真。可用于比较字符串和数字|
|!=|比较运算符|比较两个对象是否不等，不等则为真|
|\>|比较运算符|比较两个对象的大小，左边的值大于右边的值，则为真|
|<|比较运算符|比较两个对象的大小，左边的值小于右边的值，则为真|
|\>=|比较运算符|比较两个对象的大小，左边的值大于等于右边的值，则为真|
|<=|比较运算符|比较两个对象的大小，左边的值小于等于右边的值，则为真|
|and|逻辑运算符|逻辑与，当左边和右边两个表达式同时为真，则返回真|
|or|逻辑运算符|逻辑或，当左右和右边两个表达式任意一个为真，则返回真|
|not|逻辑运算符|逻辑否，对表达式取反|
|()|逻辑运算符|当一组表达式组合在一起，形成一个更大的表达式，组合内的所有表达式都是逻辑与的关系|

## 2、关键字

### 2.1、判断路径

| 关键字 | 说明 |
| :- | :- |
| file | 判断指定路径是否为一个文件，是则为真 |
| directory | 判断指定路径是否为一个目录，是则为真 |
| link | 判断指定路径是否为一个软链接，是则为真 |
| mount | 判断指定路径是否为一个挂载点，是则为真 |
| exists | 判断指定路径是否存在，存在则为真 |

-*注意：关于路径的所有判断均是判断主控端上的路径，而非被控端上的路径*

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  vars:
    testpath: ~/downloads
  tasks:
    - debug:
        msg: "path exist"
      when: testpath is exists
```

### 2.2、判断变量定义

| 关键字 | 说明 |
| :- | :- |
| defined | 判断变量是否已定义，已定义则返回真 |
| undefined | 判断变量是否未定义，未定义则返回真 |
| none | 判断变量的值是否为空，如果变量已定义且值为空，则返回真 |

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  vars:
    var1: "test"
    var3:
  tasks:
    - debug:
        msg: "var1 is defined"
      when: var1 is defined
    - debug:
        msg: "var2 is undefined"
      when: var2 is undefined
    - debug:
        msg: "var3 is none"
      when: var3 is none
```

### 2.3、判断执行结果

| 关键字 | 说明 |
| :- | :- |
| sucess | 通过任务执行结果返回的信息判断任务的执行状态，任务执行成功则返回true |
| failure | 任务执行失败则返回true |
| change | 任务执行状态为changed则返回true |
| skip | 任务被跳过则返回true |

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  vars:
    doshell: true
  tasks:
    - shell: 'cat ~/test.log'
      when: doshell
      register: result
      ignore_errors: true
    - debug:
        msg: "success"
      when: result is success
    - debug:
        msg: "failed"
      when: result is failure
    - debug:
        msg: "changed"
      when: result is change
    - debug:
        msg: "skip"
      when: result is skip
```

### 2.4、判断字符串

| 关键字 | 说明 |
| :- | :- |
| lower | 判断字符串中的所有字母是否都是小写，是则为真 |
| upper | 判断字符串中的所有字母是否都是大写，是则为真 |

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  vars:
    str1: "abc"
    str2: "ABC"
  tasks:
    - debug:
        msg: "str1 is all lowercase"
      when: str1 is lower
    - debug:
        msg: "str2 is all uppercase"
      when: str2 is upper
```

### 2.5、判断整除

| 关键字 | 说明 |
| :- | :- |
| even | 判断数值是否为偶数，是则为真 |
| odd | 判断数值是否为奇数，是则为真 |
| divisibleby(num) | 判断是否可以整除指定的数值，是则为真 |

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  vars:
    num1: 6
    num2: 7
    num3: 15
  tasks:
    - debug:
        msg: "num1 is an even number"
      when: num1 is even
    - debug:
        msg: "num2 is an odd number"
      when: num2 is odd
    - debug:
        msg: "num3 can be divided exactly by 3"
      when: num3 is divisibleby(3)
```

### 2.6、版本比较

使用 version 关键字进行版本的大小比较。
-*注意：2.5 版本此 test 从 version_compare 更名为 version*
比较操作符如下：

| 说明 | 操作符 |
| :- | :- |
| 大于 | >, gt |
| 大于等于 | >=, ge |
| 小于 | <, lt |
| 小于等于 | <=, le |
| 等于 | =, ==, eq |
| 不等于 | !=, <>, ne |

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  vars:
    ver1: 1.2.1
    ver2: 1.3.1
  tasks:
    - debug:
        msg: "ver2 is greater than ver1"
      when: ver2 is version(ver1,"gt")
```

### 2.7、父子集判断

| 关键字 | 说明 |
| :- | :- |
| subset | 判断一个list是不是另一个list的子集 |
| superset | 判断一个list是不是另一个list的父集 |

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  vars:
    a:
      - 2
      - 5
    b: [1,2,3,4,5]
  tasks:
    - debug:
        msg: "A is a subset of B"
      when: a is subset(b)
    - debug:
        msg: "B is the parent set of A"
      when: b is superset(a)
```

### 2.8、子串判断

| 关键字 | 说明 |
| :- | :- |
| in | 判断一个字符串是否存在于另一个字符串中，也可用于判断某个特定的值是否存在于列表中 |

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  vars:
    str1: "si"
    supported_distros:
      - RedHat
      - CentOS
  tasks:
    - debug:
        msg: "si is in ansible"
      when: str1 in "ansible"
    - debug:
        msg: "{{ ansible_distribution }} in supported_distros"
      when: ansible_distribution in supported_distros
```

### 2.9、判断变量类型

| 关键字 | 说明 |
| :- | :- |
| string | 判断对象是否为一个字符串，是则为真 |
| number | 判断对象是否为一个数字，是则为真 |

```yml
---
# file: test.yml
- hosts: 172.17.0.3
  remote_user: deploy
  gather_facts: no
  vars:
    var1: 1
    var2: "1"
    var3: a
  tasks:
    - debug:
        msg: "var1 is a number"
      when: var1 is number
    - debug:
        msg: "var2 is a string"
      when: var2 is string
    - debug:
        msg: "var3 is a string"
      when: var3 is string
```

## 3、官方文档参考

官方文档参考 [Conditionals](https://docs.ansible.com/ansible/latest/user_guide/playbooks_conditionals.html)
