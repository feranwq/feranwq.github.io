title: ansible基础.md
date: 2016-05-11 11:58:59
tags: 
mp3: 
cover: 
---

#### 安装
```
echo "deb http://ppa.launchpad.net/ansible/ansible/ubuntu trusty main" >> /etc/apt/sources.list
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367
$ sudo apt-get update
$ sudo apt-get install ansible
```
#### Ansible如何通过SSH与远程计算机进行通信的？
默认情况下，Ansible 1.3或以后的版本默认使用原生OpenSSH进行远程通信。
- 启用ControlPersist模块 (a performance feature)(开启SSH的ControlMaster并持久化socket连接，可以加速Ansible的执行速度，不需要在每次都经历SSH认证，单个服务器可能节约的时间仅在1秒左右，而上百台的服务器就能节省约1分钟左右的时间。)
- 启用Kerberos(安全模块)
- 另外还会启动`~/.ssh/config`的一些配置选项例如Jump Host配置
- Kerberized 

REDHAT6/CENTOS6由于openssh版本太低,会启用paramiko来通信,Kerberized SSH也可以用 Accelerated Mode来启用

1.2和以前的版本都是基于paramiko的,需要手动配置选择原生OpenSSH

偶尔会遇到不知sftp的机器,在配置文件里改为scp模式即可

默认使用SSH密钥连接,可以手动改为通过密码(不推荐).如果使用sudo功能，当sudo需要密码时，还要提供`--ask-become-pass`

#### 初始化
把ansible的公钥下发到被控端,`ssh-keygen -t rsa`生成,`ssh-copy-id 被控端IP`下发公钥


```
修改ansible/hosts配置文件
cat /etc/ansible/hosts
[webservers]
web1
web2
[haproxys]
haproxy1
haproxy2

# 测试连接
ansible all -m ping
web1 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
# 以用户bruce登录
$ ansible all -m ping -u bruce
# 以用户bruce登录，还要sudo到root
$ ansible all -m ping -u bruce --sudo
# 以用户bruce登录，还要sudo到用户batman
$ ansible all -m ping -u bruce --sudo --sudo-user batman

# 在最新版的Ansible中不再支持`sudo`了，最好用become参数
# 以用户bruce登录，还要sudo到root
$ ansible all -m ping -u bruce -b
# 以用户bruce登录，还要sudo到用户batman
$ ansible all -m ping -u bruce -b --become-user batman

#试着执行一条命令
$ ansible all -a "/bin/echo hello"
```

**注意**:如果一个被控端重做了系统，它的密钥和主控端中的“known_hosts”里面保存的那个就不一样了，此时主控端这边便会报错。如果确定关闭密钥校验，可以修改Ansible的配置文件`/etc/ansible/ansible.cfg`或者`~/.ansible.cfg`.

```
[defaults]
host_key_checking = False
```

#### Inventory(主机清单)
ansible通过Inventory来管理被控端,Inventory默认为/etc/ansible/hosts.可以单独指定为另外的一个或者多个,甚至可以动态获取

主机清单默认写法
```
# 未分组主机
mail.example.com

# webservers组下的主机
[webservers]
foo.example.com
bar.example.com
www[01:50].example.com # 批量

# dbservers组下的主机
[dbservers]
one.example.com
two.example.com
three.example.com
db-[a:f].example.com # 批量
```
YAML写法(推荐)
```
all:
  hosts:
    mail.example.com
  children:
    webservers:
      hosts:
        foo.example.com:
        bar.example.com:
        www[01:50].example.com:
    dbservers:
      hosts:
        one.example.com:
        two.example.com:
        three.example.com:
        db-[a:f].example.com:
```
针对反向ssh主机配置
```
# 被控端主机名-连接端口-连接IP
jumper ansible_port=5555 ansible_host=192.0.2.50
```
YAML格式
```
hosts:
  jumper:
    ansible_port: 5555
    ansible_host: 192.0.2.50
```
> 注:ansible_user, ansible_host, ansible_port都是内置变量,2.0版本以前为ansible_ssh_user, ansible_ssh_host, and ansible_ssh_port,[更多详情](http://docs.ansible.com/ansible/latest/intro_inventory.html#list-of-behavioral-inventory-parameters)

给主机单独定义变量(优先级最高)
```
[atlanta]
host1 http_port=80 maxRequestsPerChild=808
host2 http_port=303 maxRequestsPerChild=909
```
给组定义变量,所有的主机都会继承(回被覆盖,一个主机可以属于多个组,优先级顺序目前不明,待更)
```
[atlanta]
host1
host2

[atlanta:vars]
ntp_server=ntp.atlanta.example.com
proxy=proxy.atlanta.example.com
```
YAML格式
```
atlanta:
  hosts:
    host1:
    host2:
  vars:
    ntp_server: ntp.atlanta.example.com
    proxy: proxy.atlanta.example.com
```
**为什么推荐YAML格式?关于嵌套组,个人认为YAML更直观**

INI格式
```
[atlanta]
host1
host2

[raleigh]
host2
host3

[southeast:children]
atlanta
raleigh

[southeast:vars]
some_server=foo.southeast.example.com
halon_system_timeout=30
self_destruct_countdown=60
escape_pods=2

[usa:children]
southeast
northeast
southwest
northwest
```
YAML格式
```
all:
  children:
    usa:
      children:
        southeast:
          children:
            atlanta:
              hosts:
                host1:
                host2:
            raleigh:
              hosts:
                host2:
                host3:
          vars:
            some_server: foo.southeast.example.com
            halon_system_timeout: 30
            self_destruct_countdown: 60
            escape_pods: 2
       northeast:
       northwest:
       southwest:
```
- 父组-子组-主机三个角色
- 一旦一个组成为子组,那么组下的主机也属于父组
- 子组的变量有更高的优先级
- 每个组可以有多个父组也可以有多个子组,但不是循环关系
- 主机可以属于多个组,最终会把所有数据合成

默认组

有两个默认组分别是 `all` 和 `ungrouped`.`all`包含所有主机, `ungrouped`包含所有没有属组的主机,他们总是存在的,但是不会显示在`group_names`中


**主机和组的变量解耦**

不要将变量存在主inventory(/etc/ansible/hosts)文件中,把变量解耦到配置文件,如下例
```
# /etc/ansible/hosts
all:
  children:
    raleigh:
      hosts:
        foosball
    webservers:
      hosts:
        foosball
        
# /etc/ansible/group_vars/raleigh.yml
---
ntp_server: acme.example.org

# /etc/ansible/group_vars/webservers.yml
---
database_server: storage.example.org

# /etc/ansible/host_vars/foosball.yml 
zabbix_server: zabbix.example.org

# 也可以把组定义成目录,再解耦
# /etc/ansible/group_vars/raleigh/db_settings.yml
# /etc/ansible/group_vars/raleigh/cluster_settings.yml
```
> `group_vars`和`host_vars`目录从1.2之后支持,定义组目录从1.4之后支持

**最好把inventory(主文件和变量文件)维护在git或者其他版本控制工具之中**

#### 动态获取主机清单
待更新

#### Patterns
在ansible里用Patterns来定义要控制的终端，其实就是指定命令下发给哪些机器,可以理解为通过特定语法把目标主机加到临时组Patterns来批量控制

下面的例子webservers组就是patterns

`ansible <pattern_goes_here> -m <module_name> -a <arguments>`

`ansible webservers -m service -a "name=httpd state=restarted"`

下面来看一下patterns的语法
```
all                                      # 所有
*                                        # 所有
one.example.com                          # 指定单个主机
one.example.com:two.example.com          # 指定主机one加two(都会执行)
192.0.2.50                               # 通过IP
192.0.2.*                                # 通过IP段
webservers                               # 指定主机组
webservers:dbservers                     # 指定webservers组加dbservers组
webservers:!phoenix                      # 在webservers组但是不在phoenix组中的主机
webservers:&staging                      # 同时在webservers组和staging组的主机
webservers:dbservers:&staging:!phoenix   # (webservers组加dbservers组的主机)同时在staging组并且不在phoenix
one*.com:dbservers                       # 主机名和组混用
~(web|db).*\.example\.com                # ~开头启用正则
ansible-playbook site.yml --limit datacenter2 # 在命令行里可以用--limit来单独排除
ansible-playbook site.yml --limit @retry_hosts.txt # 排除retry_hosts.txt里的主机
```

## Introduction To Ad-Hoc Commands(临时/快速命令)

#### SHELL和COMMAND

```
# -a 执行的命令 
# -m 默认是command,可以省略.但是命令里如果有管道|或者重定向<>需要-m指定shell来执行
<注意> -a里需要用单引号来表示变量
# -f 启用N个进程去下发任务,默认5个.可以通过配置文件修改(官方推荐尽量设置高一点Feel free to push this value as high as your system can handle!)
# -u 指定执行命令的用户
# -u username --become  [otheruser] [--ask-become-pass] 指定执行sudu命令的用户,可以指定su其他用户,--ask-become-pass可以进入交互模式以提供sudo密码
root@vps-test-backup:/etc/ansible# time ansible webservers:haproxys -a "/bin/echo hello" -f 4
haproxy1 | SUCCESS | rc=0 >>
hello

web1 | SUCCESS | rc=0 >>
hello

haproxy2 | SUCCESS | rc=0 >>
hello

web2 | SUCCESS | rc=0 >>
hello


real	0m1.099s
user	0m0.716s
sys	0m0.144s
root@vps-test-backup:/etc/ansible# time ansible webservers:haproxys -a "/bin/echo hello" -f 1
web1 | SUCCESS | rc=0 >>
hello

web2 | SUCCESS | rc=0 >>
hello

haproxy1 | SUCCESS | rc=0 >>
hello

haproxy2 | SUCCESS | rc=0 >>
hello


real	0m1.485s
user	0m0.728s
sys	0m0.140s



root@vps-test-backup:/etc/ansible# ansible webservers:haproxys -m shell -a 'echo $PATH'
# 正常返回
haproxy2 | SUCCESS | rc=0 >>
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin

web1 | SUCCESS | rc=0 >>
/usr/share/jdk1.8.0_161/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin

web2 | SUCCESS | rc=0 >>
/usr/share/jdk1.8.0_161/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin

haproxy1 | SUCCESS | rc=0 >>
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin

root@vps-test-backup:/etc/ansible# ansible webservers:haproxys -m shell -a "echo $PATH"
# 可以看到显示的是ansible本地的PATH变量
web1 | SUCCESS | rc=0 >>
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

web2 | SUCCESS | rc=0 >>
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

haproxy1 | SUCCESS | rc=0 >>
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

haproxy2 | SUCCESS | rc=0 >>
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin


```

#### 文件传输
```
root@vps-test-backup:/etc/ansible# ansible webservers:haproxys -m copy -a "src=/etc/hosts dest=/tmp/hosts"
# -m copy 复制文件
web1 | SUCCESS => {
    "changed": true, 
    "checksum": "e38ae5206830b1c903e032463f63622253f88a45", 
    "dest": "/tmp/hosts", 
    "gid": 0, 
    "group": "root", 
    "md5sum": "8183b6a55f8569fb597b1ad52345c761", 
    "mode": "0644", 
    "owner": "root", 
    "secontext": "unconfined_u:object_r:admin_home_t:s0", 
    "size": 326, 
    "src": "/root/.ansible/tmp/ansible-tmp-1516949432.32-34449028488535/source", 
    "state": "file", 
    "uid": 0
}

root@vps-test-backup:/etc/ansible# ansible webservers -m file -a "dest=/tmp/hosts mode=600 owner=mail group=mail"
# -m file 修改文件或目录权限/所属权
web1 | SUCCESS => {
    "changed": true, 
    "gid": 12, 
    "group": "mail", 
    "mode": "0600", 
    "owner": "mail", 
    "path": "/tmp/hosts", 
    "secontext": "unconfined_u:object_r:admin_home_t:s0", 
    "size": 326, 
    "state": "file", 
    "uid": 8
}

root@vps-test-backup:/etc/ansible# ansible web1 -m file -a "dest=/path/to/c mode=755 state=directory"
# 创建目录,类似mkdir -p
web1 | SUCCESS => {
    "changed": true, 
    "gid": 0, 
    "group": "root", 
    "mode": "0755", 
    "owner": "root", 
    "path": "/path/to/c", 
    "secontext": "unconfined_u:object_r:default_t:s0", 
    "size": 6, 
    "state": "directory", 
    "uid": 0
}

root@vps-test-backup:/etc/ansible# ansible web1 -m file -a "dest=/path/to/c state=absent"
# 递归删除目录
web1 | SUCCESS => {
    "changed": true, 
    "path": "/path/to/c", 
    "state": "absent"
}


```

#### 软件包管理
支持yum和apt
```
ansible webservers -m yum -a "name=acme state=present"           # 确保acme已安装,不更新
ansible webservers -m yum -a "name=acme-1.5 state=present"       # 确保acme是1.5版本,不更新
ansible webservers -m yum -a "name=acme state=latest"            # 确保acme已安装,更新到最新版本
ansible webservers -m yum -a "name=acme state=absent"            # 确保acme没有安装


ansible all -m user -a "name=foo password=<crypted password here>" # 创建用户
ansible all -m user -a "name=foo state=absent"                     # 删除用户

ansible webservers -m git -a "repo=https://foo.example.org/repo.git dest=/srv/myapp version=HEAD" # 直接用git部署应用
```

#### 服务管理

```
ansible webservers -m service -a "name=httpd state=started"
ansible webservers -m service -a "name=httpd state=restarted"
ansible webservers -m service -a "name=httpd state=stopped"
```

#### 限时后台操作

```
# 后台执行命令,超时时间为1800秒,每60秒获取一次状态,超时/执行完成都会结束后台进程
ansible all -B 1800 -P 60 -a "some command"
# 传入任务ID手动查询状态
 ansible web1.example.com -m async_status -a "jid=488359678239.2844"
```

#### Gathering Facts收集机器信息
`ansible all -m setup`