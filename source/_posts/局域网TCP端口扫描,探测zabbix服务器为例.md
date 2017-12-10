title: 局域网TCP端口扫描,探测zabbix服务器为例
date: 2017-10-23 21:58:22
tags: 
mp3: 
cover: http://wx2.sinaimg.cn/large/77b6881dgy1fmc3vi5br5j20rs0rsdic.jpg
---
通过获取本机地址反推本段(一个C)的IP列表,然后批量扫描TCP端口是否可用,后续要了解是否可以直接通过掩码来反推一个局域网IP的生成器出来提高效率
```
#!/usr/bin/env python
# -*- coding: utf8 -*-


'''
脚本通过默认路由探测本网段地址内的zabbix proxy服务器
'''
from threading import Thread, activeCount
import socket
import os
import re

#测试ip和端口是否开放
def test_port(dst,port):
    #os.system('title '+str(port))

    cli_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    try:

        indicator = cli_sock.connect_ex((dst, port))
        if indicator == 0:
            print(dst,port)
        cli_sock.close()
    except:

        pass

def get_network_ip():
    default_route = None
    network_ip_list = []

    for line in os.popen('ip route show'):
        if re.match('^\s*default\s+via\s+(\S+)\s+dev\s+(\S+)\s.*?\n$', line):
            m = re.match('^\s*default\s+via\s+(\S+)\s+dev\s+(\S+)\s.*?\n$', line)
            default_route = {
                'ip_address': m.group(1),
                'interface_name': m.group(2)
            }
            break
    ip_prefix = '.'.join("".join(m.group(1)).split('.')[:-1])
    for i in range(1,255):
        ip = '%s.%s'%(ip_prefix,i)
        network_ip_list.append(ip)

    return network_ip_list

if __name__=='__main__':
    #扫描ip
    dst_list = get_network_ip()
    i = 10051
    #20线程，扫描0~100端口
    #while i < 10060:
    if activeCount() <= 20:
        for dst in dst_list:
            Thread(target = test_port, args = (dst, i)).start()

    #        i = i + 1
    while True:
        if activeCount() == 2:
            break
