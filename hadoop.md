# 1.CDH安装：基础解压与软链

```shell

chattr -i /etc/gshadow
chattr -i /etc/group
chattr -i /etc/shadow
chattr -i /etc/passwd

if ! [ `grep "^dsp:" /etc/group` ] ; then
/usr/sbin/groupadd -r dsp
if ! [ `grep "^dsp:" /etc/passwd` ] ; then
/usr/sbin/useradd -r -m -g dsp -K UMASK=022 dsp --groups dsp
fi
fi

chattr +i /etc/gshadow
chattr +i /etc/group
chattr +i /etc/shadow
chattr +i /etc/passwd



if ! [ -d /opt/CDH-6.3.2-1.cdh6.3.2.p0.1605554 ] ; then
mkdir -p /opt/
if ! [ -f /opt/CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel ] ; then
rsync -av --prune-empty-dirs rsync://10.8.219.87/software/CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel /opt/CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel
#if ! [ -f /opt/CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel ] ; then
# echo "Failed to get CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel"
#fi
fi
cd /opt/
tar -zxf CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel
chown root.root /opt/CDH-6.3.2-1.cdh6.3.2.p0.1605554 -R
ln -s CDH-6.3.2-1.cdh6.3.2.p0.1605554 CDH

for i in `ls /opt/CDH/etc/default/`;do ln -s /opt/CDH/etc/default/$i /etc/default/$i;done

elif ! [ -L /opt/CDH ] ; then
ln /opt/CDH-6.3.2-1.cdh6.3.2.p0.1605554 /opt/CDH
fi



sed -i 's#/usr/lib/#/opt/CDH/lib/#g' /opt/CDH/etc/default/hadoop
sed -i 's#/usr/lib/#/opt/CDH/lib/#g' /opt/CDH/etc/default/hadoop-mapreduce-historyserver
sed -i 's#/usr/lib/#/opt/CDH/lib/#g' /opt/CDH/etc/default/hive-webhcat-server
sed -i 's#/usr/lib/#/opt/CDH/lib/#g' /opt/CDH/etc/default/impala
sed -i 's#/usr/lib/#/opt/CDH/lib/#g' /opt/CDH/etc/rc.d/init.d/zookeeper-server
if ! [ -L /opt/CDH/lib/hadoop/apache-log4j-extras-1.2.17.jar ] ; then
ln -s /opt/CDH/jars/apache-log4j-extras-1.2.17.jar /opt/CDH/lib/hadoop/apache-log4j-extras-1.2.17.jar
fi



chattr -i /etc/gshadow
chattr -i /etc/group
chattr -i /etc/shadow
chattr -i /etc/passwd

if ! [ `grep "^hadoop:" /etc/group` ] ; then
/usr/sbin/groupadd -r hadoop

fi

if ! [ `grep "^hdfs:" /etc/group` ] ; then
/usr/sbin/groupadd -r hdfs
if ! [ `grep "^hdfs:" /etc/passwd` ] ; then
/usr/sbin/useradd -r -m -g hdfs -K UMASK=022 --home /var/lib/hadoop-hdfs --comment "Hadoop HDFS" --shell /sbin/nologin --groups hadoop hdfs
fi
fi

chattr +i /etc/gshadow
chattr +i /etc/group
chattr +i /etc/shadow
chattr +i /etc/passwd
```

# 2.Hadoop集群服务部署

## 2.1HDFS的ZOOKEEPER服务集群搭建

1. portal上操作：创建zk_hdfs组（zk001.hdfs\zk002.hdfs\zk003.hdfs）与hosts下发

2. CDH包获取与解压、软链（参见评论**【CDH安装：基础解压与软链】**）

3. zookeeper用户、组创建

4. ```shell
   chattr -i /etc/gshadow /etc/group /etc/shadow /etc/passwd
    
   if ! [ `grep "^zookeeper:" /etc/group` ] ; then
       /usr/sbin/groupadd -r zookeeper
   fi
   if ! [ `grep "^zookeeper:" /etc/passwd` ] ; then
       /usr/sbin/useradd -r -m -g zookeeper -K UMASK=022 --home /var/lib/zookeeper --comment ZooKeeper --shell /sbin/nologin zookeeper
   fi
    
   chattr +i /etc/gshadow /etc/group /etc/shadow /etc/passwd
   ```

   

5. 执行安装

   ```shell
   mkdir -p -m 0755 /etc/zookeeper
   /usr/sbin/update-alternatives --install /etc/zookeeper/conf zookeeper-conf /opt/CDH/etc/zookeeper/conf.dist 10
   /usr/sbin/update-alternatives --install /usr/bin/zookeeper-client zookeeper-client /opt/CDH/bin/zookeeper-client 10
   /usr/sbin/update-alternatives --install /usr/bin/zookeeper-server zookeeper-server /opt/CDH/bin/zookeeper-server 10
   /usr/sbin/update-alternatives --install /usr/bin/zookeeper-server-initialize zookeeper-server-initialize /opt/CDH/bin/zookeeper-server-initialize 10
   ```

   



6. 配置：/etc/zookeeper/conf/zoo.cfg配置文件，添加关键信息，如（假设安装zookeeper服务的是3台机器，分别为server.1\server.2\server.3——zk001.hdfs\zk002.hdfs\zk003.hdfs

   ```shell
   # Licensed to the Apache Software Foundation (ASF) under one or more
   # contributor license agreements.  See the NOTICE file distributed with
   # this work for additional information regarding copyright ownership.
   # The ASF licenses this file to You under the Apache License, Version 2.0
   # (the "License"); you may not use this file except in compliance with
   # the License.  You may obtain a copy of the License at
   #
   #     http://www.apache.org/licenses/LICENSE-2.0
   #
   # Unless required by applicable law or agreed to in writing, software
   # distributed under the License is distributed on an "AS IS" BASIS,
   # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   # See the License for the specific language governing permissions and
   # limitations under the License.
   
   maxClientCnxns=50
   # The number of milliseconds of each tick
   tickTime=2000
   # The number of ticks that the initial
   # synchronization phase can take
   initLimit=10
   # The number of ticks that can pass between
   # sending a request and getting an acknowledgement
   syncLimit=5
   # the directory where the snapshot is stored.
   dataDir=/cache1/zookeeper/data
   # the port at which the clients will connect
   clientPort=2181
   # the directory where the transaction logs are stored.
   dataLogDir=/var/lib/zookeeper
   server.1=hadoop.zk001:2888:3888
   server.2=hadoop.zk002:2888:3888
   server.3=hadoop.zk003:2888:3888
   ```

7. 在对应的server.x机器上分别调整配置（myid=1 要改为对应的1，2，3值）

```shell
mkdir -p /var/log/zookeeper
chown zookeeper.zookeeper /var/log/zookeeper -R
chown zookeeper.zookeeper /var/lib/zookeeper -R
mkdir -p /cache1/zookeeper/data
chown zookeeper.zookeeper /cache1/zookeeper/data
su -s /bin/bash -c "/opt/CDH/bin/zookeeper-server-initialize --myid=1" zookeeper
```

8. 启动服务

```shell
ln -s /opt/CDH/etc/rc.d/init.d/zookeeper-server /etc/init.d/zookeeper-server
service zookeeper-server start
/sbin/chkconfig zookeeper-server on
/opt/CDH/bin/zookeeper-server status
/opt/CDH/lib/zookeeper/bin/zkServer.sh status
```



## 2.2**journalnode服务搭建（与hdfs的zk机器合设）**

1. 参见评论进行**【CDH安装：基础解压与软链】**

2.  执行安装

   ```shell
   mkdir -p -m 0755 /etc/hadoop
   /usr/sbin/update-alternatives --install /usr/bin/hdfs hdfs /opt/CDH/bin/hdfs 10
   /usr/sbin/update-alternatives --install /etc/hadoop/conf hadoop-conf /opt/CDH/etc/hadoop/conf.empty 10
   mkdir -p /var/log/hadoop-hdfs
   chown hdfs:hdfs /var/log/hadoop-hdfs -R
   mkdir -p /cache1/jn_dir
   chown hdfs:hdfs /cache1/jn_dir
   mkdir -p /usr/local/hadoop_conf/hdfs/xmtest/
   rsync -av [rsync://10.8.219.87/hadoop_conf/hdfs/xmtest/](rsync://10.8.219.87/hadoop_conf/hdfs/xmtest/) /usr/local/hadoop_conf/hdfs/xmtest/ && /bin/cp -rf /usr/local/hadoop_conf/hdfs/xmtest/* /etc/hadoop/conf/
   ```

   

3. 服务启动：

   ```shell
   ln -s /opt/CDH/etc/rc.d/init.d/hadoop-hdfs-journalnode /etc/init.d/hadoop-hdfs-journalnode
   service hadoop-hdfs-journalnode start
   ```

   

## 2.3 **namenode服务搭建**

1. 参见评论进行**【CDH安装：基础解压与软链】**

2. 执行安装

   ```shell
   mkdir -p -m 0755 /etc/hadoop
   /usr/sbin/update-alternatives --install /usr/bin/hdfs hdfs /opt/CDH/bin/hdfs 10
   /usr/sbin/update-alternatives --install /etc/hadoop/conf hadoop-conf /opt/CDH/etc/hadoop/conf.empty 10
   ```

3. 创建目录

   ```shell
   mkdir -p /cache2/hdfs/dfs/image
   mkdir -p /cache1/hdfs/dfs/edits
   chown hdfs:hdfs /cache2/hdfs/dfs/image
   chown hdfs:hdfs /cache1/hdfs/dfs/edits
   ```

4. mkdir -p /usr/local/hadoop_conf/hdfs/xmtest/
   rsync -av [rsync://10.8.219.87/hadoop_conf/hdfs/xmtest/](rsync://10.8.219.87/hadoop_conf/hdfs/xmtest/) /usr/local/hadoop_conf/hdfs/xmtest/ && /bin/cp -rf /usr/local/hadoop_conf/hdfs/xmtest/* /etc/hadoop/conf/

5. nn001上执行

   ```shell
   ln -s /opt/CDH/etc/rc.d/init.d/hadoop-hdfs-namenode /etc/init.d/hadoop-hdfs-namenode
   sudo -u hdfs hdfs namenode -format
   service hadoop-hdfs-namenode start
   ```

6. nn002上执行

   ```shell
   ln -s /opt/CDH/etc/rc.d/init.d/hadoop-hdfs-namenode /etc/init.d/hadoop-hdfs-namenode
   sudo -u hdfs hdfs namenode -bootstrapStandby
   service hadoop-hdfs-namenode start
   ```

7. 查看

   1. http://10.8.218.87:50070/dfshealth.html#tab-overview (http://10.8.218.87:50070/dfshealth.html#tab-datanode)
   2. http://10.8.219.91:50070/dfshealth.html#tab-overview (http://10.8.219.91:50070/dfshealth.html#tab-datanode)

8. hadoop-hdfs-zkfc 设置,NN高可用管理

   1. nn001上执行

      ```shell
      hdfs zkfc -formatZK
      ln -s /opt/CDH/etc/rc.d/init.d/hadoop-hdfs-zkfc /etc/init.d/hadoop-hdfs-zkfc
      service hadoop-hdfs-zkfc start
      ```

   2. nn002上执行

      ```shell
      ln -s /opt/CDH/etc/rc.d/init.d/hadoop-hdfs-zkfc /etc/init.d/hadoop-hdfs-zkfc
      service hadoop-hdfs-zkfc start
      ```

   3. 主备状态确认

      1. hdfs haadmin -ns xmtest -getServiceState nn001
      2. hdfs haadmin -ns xmtest -getServiceState nn002

## 2.4 **datanode服务搭建**

1. 参见评论进行**【CDH安装：基础解压与软链】**

2. 执行安装

   ```shell
   /usr/sbin/update-alternatives --install /usr/bin/hdfs hdfs /opt/CDH/bin/hdfs 10
   mkdir -p -m 0755 /etc/hadoop
   /usr/sbin/update-alternatives --install /etc/hadoop/conf hadoop-conf /opt/CDH/etc/hadoop/conf.empty 10
   mkdir -p /var/log/hadoop-hdfs
   chown hdfs.hdfs /var/log/hadoop-hdfs -R
   mkdir -p /var/run/hadoop-hdfs
   chown hdfs.hdfs /var/run/hadoop-hdfs
   ```

3. 创建数据盘

   ```shell
   for i in {1..2}
   do
    mkdir -p /cache$i/hdfs/dfs/data
    chown hdfs:hdfs /cache$i/hdfs/dfs/data
   done
   mkdir -p /usr/local/hadoop_conf/hdfs/xmtest/
   rsync -av [rsync://10.8.219.87/hadoop_conf/hdfs/xmtest/](rsync://10.8.219.87/hadoop_conf/hdfs/xmtest/) /usr/local/hadoop_conf/hdfs/xmtest/ && /bin/cp -rf /usr/local/hadoop_conf/hdfs/xmtest/* /etc/hadoop/conf/
   ```

4. 启动服务

   `ln -s /opt/CDH/etc/rc.d/init.d/hadoop-hdfs-datanode /etc/init.d/hadoop-hdfs-datanode`
   `service hadoop-hdfs-datanode start`

# 3.**kafka集群搭建**

1. 先部署通用zookeeper集群，再部署kafka集群
2. zookeeper服务启动：/etc/init.d/zk.sh start
   1. /usr/local/src/zookeeper-3.4.8/bin/zkServer.sh status
   2. /etc/init.d/zk.sh status
3. kafka服务启动：/etc/init.d/kafka.sh start
   1. /etc/init.d/kafka.sh status
4. 创建topic测试：/usr/local/src/kafka_2.11-0.10.1.0/bin/kafka-topics.sh --create --zookeeper zk001.common:2181,zk002.common:2181,zk003.common:2181/kafka --topic first_topic --partitions 1 --replication-factor 1
5. kafka所有topic查看：/usr/local/src/kafka_2.11-0.10.1.0/bin/kafka-topics.sh --list --zookeeper zk001.common:2181,zk002.common:2181,zk003.common:2181/kafka