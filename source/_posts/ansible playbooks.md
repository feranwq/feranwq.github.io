title: BLOG部署
date: 2016-05-13 11:58:59
tags: 
mp3: 
cover:
---

## 初识Playbooks
写好剧本,让服务器按照你的剧本尽情表演.

playbooks的基础用法是用来配置和管理远程服务器,但是还有更多高级功能例如涉及滚动更新的多重部署操作进行排序,并且可以和监控服务器负载均衡服务器交互.

每个playbooks都由多个plays组成,每个play可以理解为一场戏,让主机们来扮演不同的角色,可以理解为让他们来调用不同的模块(-m somemoudle).

playbooks是yaml格式
```
# 安装运行apache
---
- hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
  - name: ensure apache is at the latest version
    yum: name=httpd state=latest
  - name: write the apache config file
    template: src=/srv/httpd.j2 dest=/etc/httpd.conf
    notify:
    - restart apache
  - name: ensure apache is running (and enable it at boot)
    service: name=httpd state=started enabled=yes
  handlers:
    - name: restart apache
      service: name=httpd state=restarted
```

```
# 两场戏
---
- hosts: webservers
  remote_user: root

  tasks:
  - name: ensure apache is at the latest version
    yum: name=httpd state=latest
  - name: write the apache config file
    template: src=/srv/httpd.j2 dest=/etc/httpd.conf

- hosts: databases
  remote_user: root

  tasks:
  - name: ensure postgresql is at the latest version
    yum: name=postgresql state=latest
  - name: ensure that postgresql is started
    service: name=postgresql state=started
```
配置文件逐个分析

```
# hosts 和remote_user
# 指定patterns和执行任务的用户,可以特定任务指定特定用户,是否需要sudo
---
- hosts: webservers
  remote_user: yourname
  
---
- hosts: webservers
  remote_user: yourname
  tasks:
    - service: name=nginx state=started
      remote_user: yourname
      become: yes
      become_method: sudo
```

```
# tasks list
# 根据playbooks从上到下表演时，task失败的主机将被移出整个palybooks。
# tasks里的模块是幂等的,模块执行一次/多次效果应该是一样的,模块会检查是否达到了期望状态,如果达到状态就退出不再执行.每个play的模块都是幂等的话,整个playbooks就是幂等的,重复运行playbooks是安全的
tasks:
  - name: make sure apache is running # 每个任务都有一个name,可以理解为任务注释,最好简介明了的表明任务目标
    service: name=httpd state=started # 调用的模块,传入模块所需参数
    
tasks:
  - name: run this command and ignore the result
    shell: /usr/bin/somecommand # shell/command模块直接提供命令
    ignore_errors: True
    
tasks:
  - name: create a virtual host file for {{ vhost }} # 支持调用变量
    template: src=somefile.j2 dest=/etc/httpd/conf.d/{{ vhost }}    
    
template: src=templates/foo.j2 dest=/etc/foo.conf # 新版本模块调用方法,推荐
action: template src=templates/foo.j2 dest=/etc/foo.conf # 0.8以前的老版本,现在版本也可以用
```
#### Handlers: Running Operations On Change 产生变化时的操作
在一个play里无论有多少个task调用了notify,它只会在全部执行完成的时候被调用一次

```
# 只有当foo.conf文件产生变化时重启服务,这里的notify就是一个handlers

handlers:
    - name: restart memcached
      service: name=memcached state=restarted
    - name: restart apache
      service: name=apache state=restarted

- name: template configuration file
  template: src=template.j2 dest=/etc/foo.conf
  notify:
     - restart memcached
     - restart apache
     
#2.2之后的版本可以listen
handlers:
    - name: restart memcached
      service: name=memcached state=restarted
      listen: "restart web services"
    - name: restart apache
      service: name=apache state=restarted
      listen: "restart web services"

tasks:
    - name: restart everything
      command: echo "this task will restart the web services"
      notify: "restart web services"
```
- notify和listen总是按handlers定义的顺序执行,不会因为在plays里notify定义的顺序改变
- handlers和listen定义好在playbooks全局生效
- 虽然handlers的name可以重复,但是只有一个可以执行,最好区别开
- You cannot notify a handler that is defined inside of an include. As of Ansible 2.1, this does work, however the include must be static.
- handlers定义在`pre_tasks`,`tasks`和`post_tasks`中时,会在被notify之后的最后自动刷新
- handlers定义在`roles`中时,会tasks部分最后自动刷新,但是会早于所有的tasks's handlers

#### 执行剧本
```
ansible-playbook playbook.yml -f 10
--syntax-check 可以检查yaml语法
--verbose 详细输出
--list-hosts 在下发playbooks之前查看pattens
```

## Creating Reusable Playbooks创建可重用的剧本(解耦)
有些plays可以独立出来给不同的playbooks重复使用,ansible为此提供了`includes`,`imports`和`roles`

`includes`和`imports`是2.4版本的新特性,允许用户把大的playbooks拆分成多个小的playbooks被包含/引用甚至可以在同一个playbooks里重复使用

`roles`不仅可以把tasks打包到一起,还可以包含`vars`,`handlers`,`modules`和`plugins`

#### 动态和静态
重用plays的方法分两种,一种是动态包含`include*`(`include_tasks`, `include_role`, etc.),一种是静态导入`import*` (`import_playbook`, `import_tasks`, etc.)
#### 动态和静态的差异
这两种操作模式十分简单
- 静态导入是在加载时展开
- 动态包含是在运行时展开

当涉及到`task`的选项如`tags`或者条件语句(例如`when`)时:
-  静态导入时,when在被import的文件里的每个task, 都会重新检查一次. 因为是加载时展开的, 文件名的变量不能是动态设定的.
- 动态包含时,when只应用一次,被include的文件名可以使用变量

#### 动态包含和静态引入的权衡和坑
使用`include *`语句的主要优点是循环。当循环与`include`一起使用时，被动态包含的`task`或`role`将为循环中的每个项目执行一次。
与`include *`语句相比，使用`import *`有一些限制：
- 只存在于动态包含内的`tags`不会显示在`–list-tags`的输出里
- 只存在于动态包含内的`tasks`不会显示在`–list-tasks`的输出里
- 不能使用`notify`来触发来自动态包含内部的`handler`
- 不能使用`--start-at-task`开始执行动态包含内的`task`

与`import *`语句相比，使用`include *`也有一些限制：
- 循环不能用于静态导入
- When using variables for the target file or role name, variables from inventory sources (host/group vars, etc.) cannot be used.

示例
```
x.yml
- hosts: 127.0.0.1
  gather_facts: False
  tasks:
    - set_fact: mode=1
    - include_tasks: y.yml
      when: mode == "1"

    - set_fact: mode=1
    - import_tasks: y.yml
      when: mode == "1"
      
y.yml
- set_fact: mode="2"

- debug:
    msg: >
      Display in only `include_tasks`.
      `include_tasks` does NOT re-evaluate `mode` for every step.
      `import_tasks` DOES re-evaluate condition.
```
运行`ansible-play -b x.yml` 后: debug 的message只在`include_tasks` 里被执行了.

第2个`import_tasks`中的debug被skip掉了.

因为`mode`被改变之后, `include_tasks` 不会重新evaluate mode,`import_tasks` 会根据变化后的mode值重新evaluate每个task的条件.
示例结果
```
TASK [set_fact] *******************************************************
ok: [127.0.0.1]

TASK [include_tasks] **************************************************
included: /root/devops/ansible/y.yml for 127.0.0.1

TASK [set_fact] *******************************************************
ok: [127.0.0.1]

TASK [debug] **********************************************************
ok: [127.0.0.1] => {
    "msg": "Display in only `include_tasks`.
            `include_tasks` does NOT re-evaluate mode for every step.
            `import_tasks` DOES re-evaluate condition\n"
}

TASK [set_fact] *******************************************************
ok: [127.0.0.1]

TASK [set_fact] *******************************************************
ok: [127.0.0.1]

TASK [debug] **********************************************************
skipping: [127.0.0.1]
```