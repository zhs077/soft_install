---
title: Linux 常用命令
date: 2020-02-09 11:20:28
tags:
categories:
- 运营
- Linux
---

# 1.网络相关

## 1.1添加路由

```shell
route add -net 10.17.0.0 netmask 255.255.0.0 gw 10.17.149.1 dev eth0   
echo "any net 10.17.0.0/16 gw 10.17.137.1" >> /etc/sysconfig/static-routes
```

​    方法2重启网卡才能生效， service network restart  方法1机器重启后会失效

## 1.2添加网关

   ` route add default gw 192.168.2.1 `

## 1.3 查看网卡的带宽

` ethtool eth1`

## 1.4 iptables

```shell
iptables -I INPUT -p tcp --dport 48866 -j DROP
iptables -I INPUT -s 59.61.78.224/28 -p tcp --dport 48866 -j ACCEPT
iptables -I INPUT -s 59.61.78.128/28 -p tcp --dport 48866 -j ACCEPT
iptables -I INPUT -s 218.85.123.224/27 -p tcp --dport 48866 -j ACCEPT
iptables -I INPUT -s 218.107.205.208/28 -p tcp --dport 48866 -j ACCEPT
iptables -I INPUT -s 183.253.10.80/28 -p tcp --dport 48866 -j ACCEPT

iptables -L -n #可以查看iptables规则
iptables -L INPUT --line-numbers  #查看规则所在的行
iptables -D INPUT 1  #删除所在行的规则
service iptables save
chkconfig iptables on

#端口转发
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
iptables -t nat -A PREROUTING -d xxx.xx.xx.xx -p tcp -m tcp --dport 50070 -j DNAT --to-destination 10.18.251.22:50070
 service iptables save
#IP转发
#转发机器添加
sed -i 's/net.ipv4.ip_forward.*/net.ipv4.ip_forward = 1/g' /etc/sysctl.conf
sysctl -p
iptables -t nat -A POSTROUTING -j MASQUERADE
#需要访问的IP
ip route add default via 10.18.251.11 dev bond0  
#删除路由
ip route del default via 10.18.251.11 dev bond0
```

# 2.系统相关

## 2.1 尽量不使用交换区内存

```shell
echo 'vm.swappiness=0'>> /etc/sysctl.conf
sysctl -p
```

## 2.2 释放buffere/cache

```shell
echo 1 > /proc/sys/vm/drop_caches
```

# 3.问题排查

## 3.1 system 进程占用内存高

```shell
原因: 系统日志太多
vim /etc/systemd/journald.conf 
修改: Storage=none
重启: systemctl restart systemd-journald
释放内存:systemctl daemon-reexec 
       systemctl daemon-reload
```

# 4.shell常用命令

## 4.1 foreach

```shell
cat ip | foreach -w 20 "ssh -o ConnectTimeout=10 root@#1 ' /bin/sh /tmp/tmp.sh ' "
```

## 4.2 xargs

```shell
ps -ef|grep /root/upa_get_vivo_log.pl |awk '{print $2}' |xargs -t -I {} kill -9  {}
```

## 4.3 文件每行按符号切分

```shell
cat 1.tmp |awk -F',' '{split($0,arr,",")}END{for(i=1;i<=NF;i++)printf("%s\n",arr[i])}'|wc -l
```

## 4.4

```
ll -l /cache8/zookeeper/logs/version-2/|grep -v " Dec 15" |awk '{print "/opt/zookeeper/logs/version-2/"$NF}'  |xargs -t -I {} rm -fr {}
ll -l /cache8/zookeeper/data/version-2/|grep -v " Dec 15" |awk '{print "/opt/zookeeper/data/version-2/"$NF}'  |xargs -t -I {} rm -fr {}
ll -l /opt/zookeeper/logs/version-2/|grep -v " Dec 15" |awk '{print "/opt/zookeeper/logs/version-2/"$NF}'  |xargs -t -I {} rm -fr {}
ll -l /opt/zookeeper/data/version-2/|grep -v " Dec 15" |awk '{print "/opt/zookeeper/data/version-2/"$NF}'  |xargs -t -I {} rm -fr {}
```

## 4.5

先看看磁盘是否正常，坏道这些问题，随机读写能力是否正常。磁盘正常的话：

echo 1 > /proc/sys/vm/block_dump
然后dmesg能看见类似的输出：
                printk(KERN_DEBUG
                       "%s(%d): dirtied inode %lu (%s) on %s\n",
                       current->comm, task_pid_nr(current), inode->i_ino,
                       name, inode->i_sb->s_id);
然后看看是哪个进程在写哪些文件，是否符合预期

sar -d -p

## 4.6

`find -name "*.gz" |xargs zcat "*.gz"|head`

## 4.7 jemalloc 内存泄漏排查

```
MALLOC_CONF=prof_leak:true,lg_prof_sample:3,prof_final:true LD_PRELOAD=/usr/lib64/libjemalloc.so.2 ./TestUPAWeak
jeprof --show_bytes `which ./TestUPAWeak` jeprof.24736.0.f.heap

jeprof --show_bytes --gif  `which ./TestUPAWeak`  jeprof.24736.0.f.heap > /tmp/1.gif
```

https://learnku.com/go/wikis/38122
