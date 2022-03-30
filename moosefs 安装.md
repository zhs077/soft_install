# 1.安装包制作

# 2.mfsMaster 安装

主配置文件 mfsexports.cfg

```
x.x.0.0/16                        /       rw,alldirs,maproot=0
#运行哪些网段IP接入
# Allow "meta".
#*                      .       rw
```

启动: /usr/local/mfs/sbin/mfsmaster start

# mfsMetalogger 安装

配置文件/usr/local/mfs/etc/mfs/mfsmetalogger.cfg

```
WORKING_USER = root
WORKING_GROUP = root
MASTER_HOST = xxx.xx.xx.xx
```



# 2.mfsChunk 安装

/usr/local/mfs/etc/mfs/mfschunkserver.cfg

```
WORKING_USER = root
WORKING_GROUP = root
MASTER_HOST = xxx.xxx.xx.xx
```

 /usr/local/mfs/etc/mfs/mfshdd.cfg

```
/cache1
/cache2
/cache3
/cache4
/cache7
/cache8
/cache9
```

# 3.mfsMount 安装

挂载:

```
mkdir -p /mnt/mfs
/usr/local/mfs/bin/mfsmount /mnt/mfs -H xxx.xxx.xx.xx
```



# 4.命令

设置副本个数

/usr/local/mfs/bin/mfssetgoal  -r  2 /mnt/mfs

设置垃圾回收时间单位s

/usr/local/mfs/bin/mfssettrashtime 86400 /mnt/mfs 



# 5.常见问题

1） MATOCS_MASTER_ACK - wrong meta data id. Can't connect to master   

```
如果是整个中心的chunkserver都连不上则处理如下
    删除/cache1/.metaid ..../cache9/.metaid  删除 /usr/local/mfs/var/mfs/chunkserverid.mfs
  重启chunkserver 
  /usr/local/mfs/sbin/mfschunkserver restart
修改/usr/local/mfs/var/mfs/chunkserverid.mfs 权限 

chown -R nobody:nobody /usr/local/mfs/var/mfs/chunkserverid.mfs
查看系统日志如果没有在出现该wrong meta data id. Can't connect to master 说明成功

/usr/local/mfs/sbin/mfschunkserver stop
rm -fr /cache1/.metaid
rm -fr /cache2/.metaid  
rm -fr /cache3/.metaid 
rm -fr /cache4/.metaid 
rm -fr /cache5/.metaid 
rm -fr /cache6/.metaid 
rm -fr /cache7/.metaid 
rm -fr /cache8/.metaid 
rm -fr /cache9/.metaid
rm -fr /cache10/.metaid
rm -fr /cache11/.metaid
rm -fr  /usr/local/mfs/var/mfs/chunkserverid.mfs
/usr/local/mfs/sbin/mfschunkserver start
```

# 故障恢复

```

mfsMaster故障恢复：
    有两种恢复数据方案：
    1.	用mfsmetarestore -a 恢复, 重启mfsmaster， 并无影响；
    2.	利用备份服务器metalogger， 来恢复最近的配置文件， 复制到master       的/var/lib/mfsmaster目录。重启master也ok。
几次测试发现：如果mfsmetarestore -a无法修复，则使用metalogger也无法修复，强制使用metadata.mfs.back创建     metadata.mfs，可以启动master，但应该会丢失1小时的数据。
    安装新的master服务器
    1.	从metalogger服务器复制备份文件metadata.mfs.back / metadata_m1.mfs.bak 到新的master服务器目录（metadata.mfs.back需要定时用crontab备份）。
    2.	从metalogger服务器复制master服务器目（ /usr/local/mfs/var/mfs）录到新的master服务器
    3.	修改log，chunk client配置中的MASTER_HOST为新master服务器ip
    4.	执行数据恢复操作 其命令为mfsmetarestore -m metadata.mfs.back -o metadata.mfs changelog_ml.*.mfs，mfsmetarestore -a
    恢复成功后再执行启动mfsmaster

master机器故障不可短期恢复时，metalogger机器替换为master具体操作行为如下：
1）整个集群所有机器（除故障的master机器）同时运行 /usr/local/mfs/op_dir/stopMfs.sh
2）metalogger机器上操作
a. 卸载原有metalogger服务 /usr/local/wslog-mfsMetalogger/bin/uninstall.sh
b. 拷贝安装master包所需要metalogger备份文件，确保新安装替换maser，集群仍保留有原来数据
/bin/cp -rf /usr/local/mfs/var/mfs/metadata_ml.mfs.back /usr/local/mfs/var/mfs/metadata.mfs.back
ll /usr/local/mfs/var/mfs/changelog_ml*.mfs | awk '{print $9}' | awk -F "_ml" '{print "/bin/cp -rf "$0,$1$2}' > /tmp/tmp.sh; /bin/sh /tmp/tmp.sh
c. 安装master
/usr/local/sdn/bin/sdn_get_file.sh -a 192.168.0.14:2012 -s /usr/local/src/wslog-mfsMaster-2.3.0-1.x86_64.zip -t forQossRootDir -o 4 -d /usr/local/data/wsLog/mfs/wslog-mfsMaster-2.3.0-1.x86_64.zip -u "Sdn-peer-for-lan:yes"

mkdir -p /usr/local/mfs/etc/mfs/
vi /usr/local/mfs/etc/mfs/mfsexports.cfg  xxx.xxx.xxx填写对应的ip网段
# Allow everything but "meta".   
xxx.xxx.xxx.0/24			/	rw,alldirs,maproot=0

# Allow "meta".
#*			.	rw

cd /usr/local/src/
unzip -o wslog-mfsMaster-2.3.0-1.x86_64.zip
cd wslog-mfsMaster-2.3.0-1/
shell/install.sh

误删回复
 mkdir -p  /mnt/mfsmeta/
 /usr/local/mfs/bin/mfsmount -m /mnt/mfsmeta/ -H 180.101.24.67

 cd trash/
 mv */Extern undel/
  mv */*Extern* undel/
```

