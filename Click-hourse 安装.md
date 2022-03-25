---
title: Click-Hourse 安装(分布式表无副本)
date: 2020-05-28 12:20:28
tags:
categories:
- 部署
- Linux
---

# 1.RPM安装

查看https://clickhouse.tech/docs/zh/getting_started/install/

## 1.1 添加用户

```shell
chattr  -i  /etc/group
chattr -i /etc/gshadow
chattr  -i /etc/passwd
chattr -i /etc/shadow
adduser clickhouse
useradd -g clickhouse  clickhouse
chattr  +i  /etc/group
chattr +i /etc/gshadow
chattr  +i /etc/passwd
chattr +i /etc/shadow
```



## 1.2 权限设置

```shell
1.修改配置权限
chown -R clickhouse:clickhouse /etc/clickhouse-server/
chown -R clickhouse:clickhouse /var/log/clickhouse-server/
chown -R clickhouse:clickhouse /var/lib/clickhouse/
#数据目录
mkdir -p /cache1/var/
chown -R clickhouse:clickhouse /cache1/var/
```



# 2.配置文件

## 2.1 主配置 

/etc/clickhouse-server/config.xml 

一些主要配置项

```xml
 <listen_host>0.0.0.0</listen_host>
 <keep_alive_timeout>60</keep_alive_timeout>
<path>/cache1/var/lib/clickhouse/</path>
<tmp_path>/cache1/var/lib/clickhouse/tmp/</tmp_path>
<user_files_path>/cache1/var/lib/clickhouse/user_files/</user_files_path>
<access_control_path>/cache1/var/lib/clickhouse/access/</access_control_path>


<!--分片-->
    <remote_servers incl="clickhouse_remote_servers" >
        <!-- Test only shard config for testing distributed storage -->
        <logs> <!-- 集群名称-->
            <shard>
                <weight>1</weight>
                <replica>
                    <host>xxxxx</host>
                    <port>9000</port>
		    <user>xxxx</user>
                    <password>xxxx</password>
                </replica>
            </shard>
            <shard>
                <weight>1</weight>
                <replica>
                    <host>xxxxx</host>
                    <port>9000</port>
	<user>default</user>
                    <password>xxxx</password>
                </replica>
            </shard>
        </logs>
       
    </remote_servers>


```

## 2.2用户配置

/etc/clickhouse-server/users.xml

新增用户

```xml
        
<users>
         <upa>

            <password>xxxx</password>


            <networks incl="networks" replace="replace">
                <ip>::/0</ip>
            </networks>

            <!-- Settings profile for user. -->
            <profile>default</profile>

            <!-- Quota for user. -->
            <quota>upa_tmp</quota>
               <allow_databases>
        <database>UPA_TMP</database>
                   <!-- 管理员-->
       <access_management>1</access_management>
    </allow_databases>
        </upa>
</users>
<quotas>
<upa>
            <!-- Limits for time interval. You could specify many intervals with different limits. -->
            <interval>
                <!-- Length of interval. -->
                <duration>3600</duration>

                <!-- No limits. Just calculate resource usage for time interval. -->
                <queries>0</queries>
                <errors>0</errors>
                <result_rows>0</result_rows>
                <read_rows>0</read_rows>
                <execution_time>0</execution_time>
            </interval>
        </upa>

</quotas>

<!--监控相关开启-->
 <profiles>
 <log_queries>1</log_queries>
 </profiles>
```

## 2.3 磁盘配置

/etc/clickhouse-server/config.d/storage.xml

```xml
<yandex>
<storage_configuration>
    <disks>
      <default>
         <!--
             You can reserve some amount of free space
             on any disk (including default) by adding
             keep_free_space_bytes tag
         -->
         <keep_free_space_bytes>1024</keep_free_space_bytes>
      </default>

      <diskA>
         <!--
         disk path must end with a slash,
         folder should be writable for clickhouse user
         -->
         <path>/cache9/clickhouse/</path>
      </diskA>

      <diskB>
          <path>/cache8/clickhouse/</path>
      </diskB>
    </disks>
	    <policies>
      <diskA_only> <!-- name for new storage policy -->
        <volumes>
          <diskA_volume> <!-- name of volume -->
            <!--
                we have only one disk in that volume
                and we reference here the name of disk
                as configured above in <disks> section
            -->
            <disk>diskA</disk>
          </diskA_volume>
        </volumes>
      </diskA_only>
      <diskB_only> <!-- name for new storage policy -->
        <volumes>
          <diskA_volume> <!-- name of volume -->
            <!--
                we have only one disk in that volume
                and we reference here the name of disk
                as configured above in <disks> section
            -->
            <disk>diskB</disk>
          </diskA_volume>
        </volumes>
      </diskB_only>
    </policies>
  </storage_configuration>
  </yandex>
```

创建表时候指定磁盘类型

比如 storage_policy='diskB_only'`

# 3.Grafana监控 

## 3.1 安装

[https://grafana.com/grafana/download](https://links.jianshu.com/go?to=https%3A%2F%2Fgrafana.com%2Fgrafana%2Fdownload)

## 3.2 安装clickhouse 插件

grafana-cli plugins install vertamedia-clickhouse-datasource

## 3.3 安装clickhouse  query

https://grafana.com/grafana/dashboards/2515





# 4 2分片+2副本

2台机器1个分片，机器1 192.168.0.1 机器2 192.168.0.2

```xml
<logs-1shard-2replica>
    <shard>
        <internal_replication>true</internal_replication>
        <weight>1</weight>
        <replica>
            <host>192.168.0.1</host>
            <port>9000</port>
            <user>xxxx</user>
            <password>xxxx</password>
        </replica>
        <replica>
            <host>192.168.0.2</host>
            <port>9000</port>
            <user>xx</user>
            <password>xxxx</password>
        </replica>
    </shard>
</logs-1shard-2replica>

```

## 4.1 测试

两台机器分别创建本地表 ,表结构参考https://clickhouse.tech/docs/en/single/?query=internal_replication#ontime

```sql
CREATE TABLE ontime_local1
(
  xxxx 
)
ENGINE = MergeTree(FlightDate, (Year, FlightDate), 8192)
```

两台机器创建分布式表

```sql
 CREATE TABLE ontime_all_1 AS ontime_local1 ENGINE = Distributed('logs-1shard-2replica', default, ontime_local1, rand());
```

插入测试数据

```sql
 INSERT INTO ontime_all_1 SELECT * FROM ontime ;
```

两台机器的ontime_all_1数量一致

```
192.168.0.1 :) select count() from ontime_all_1 ;

SELECT count()
FROM ontime_all_1

┌─count()─┐
│ 3900975 │
└─────────┘

192.168.0.2 :) select count() from ontime_local1;

SELECT count()
FROM ontime_local1

┌─count()─┐
│ 3900975 │
└─────────┘
```



## 4.2 一致性

既然有2个副本，那么在写数据的时候如果一台挂了，会怎么样

1. 停掉192.168.0.2 ,集群能正常工作，往192.168.0.1插入部分数据

   ```shell
   service clickhouse-server stop
   ```

2. 通过分布式表ontime_local1 插入部分数据

```sql
 INSERT INTO ontime_all_1 SELECT * FROM ontime limit 10;
```

3. 重启192.168.0.2 服务

    发现两台服务数据一致，说明两个节点的数据能够自动同步



# 5. 2分片+2副本(不使用zookeeper)

4台机器 ，机器1 192.168.0.1 机器2 192.168.0.2 机器3 192.168.0.3 机器4 192.168.0.4

```xml
 <logs-2shard-2replica>
     <shard>
     <internal_replication>false</internal_replication>
     <weight>1</weight>
     <replica>
     <host>127.0.0.1</host>
     <port>9000</port>
     </replica>
     <replica>
     <host>127.0.0.2</host>
     <port>9000</port>
     </replica>
     </shard>
     <shard>
     <internal_replication>false</internal_replication>
     <weight>1</weight>
     <replica>
     <host>127.0.0.3</host>
     <port>9000</port>
     </replica>
     <replica>
     <host>127.0.0.4</host>
     <port>9000</port>
     </replica>
     </shard>
 </logs-2shard-2replica>

```

**internal_replication 如果设置为true,写入数据时，是先写健康的副本，然后由表自身复制，这就要求表需要自我复制，如果设置为false,则写入数据时，是写入所有的副本，是无法保证一致性**

## 5.1 初始化

1. 创建本地表

   ```
   CREATE TABLE ontime_local2
   (
     xxxx 
   )
   ENGINE = MergeTree(FlightDate, (Year, FlightDate), 8192)
   ```

2. 创建分布式表

   ```
    CREATE TABLE ontime_all_2 AS ontime_local2 ENGINE = Distributed('logs-2shard-2replica', default, ontime_local2, rand());
   ```
   
3. 初始化数据

   ```
   INSERT INTO ontime_all_2 SELECT * FROM ontime;
   ```

   

4. 可用性验证

   关闭127.0.0.2 服务和127.0.0.3 服务，整个集群还是可用





# 6. 2分片+2副本(使用zookeeper)

1. 配置文件

   ```xml
   <logs-2shard-2replica-zookeeper>
        <shard>
        <internal_replication>true</internal_replication>
        <weight>1</weight>
        <replica>
        <host>127.0.0.1</host>
        <port>9000</port>
        </replica>
        <replica>
        <host>127.0.0.2</host>
        <port>9000</port>
        </replica>
        </shard>
        <shard>
        <internal_replication>true</internal_replication>
        <weight>1</weight>
        <replica>
        <host>127.0.0.3</host>
        <port>9000</port>
        </replica>
        <replica>
        <host>127.0.0.4</host>
        <port>9000</port>
        </replica>
        </shard>
    </logs-2shard-2replica-zookeeper>
   
   <!-- zookeeper 配置-->
   <zookeeper >
       <node index="1">
           <host>127.0.0.1</host>
           <port>2181</port>
       </node>
       <node index="2">
           <host>127.0.0.2</host>
           <port>2181</port>
       </node>
       <node index="3">
           <host>127.0.0.3</host>
           <port>2181</port>
       </node>
   </zookeeper>
   <!-- 宏定义 方便创建表同一变量, shard 和 replica 每台机器不一样-->
   
   <macros>
       <shard>01</shard>
       <replica>01</replica>
   </macros>
   ```

2. /etc/hosts 添加ip和hostname 的对应关系，否则数据同步可能会出现找不到hostname问题

3. 初始化,每台机器执行

   ```sql
   CREATE TABLE `ontime_replica` (
   		xxx
   )
     ENGINE = ReplicatedMergeTree('/clickhouse/tables/ontime/{shard}', '{replica}', FlightDate, (Year, FlightDate), 8192);
     
     
    CREATE TABLE ontime_replica_all AS ontime_replica ENGINE = Distributed('logs-2shard-2replica-zookeeper', default, ontime_replica, rand());
    
   ```

4. 在其中一台插入数据

   ```shell
   INSERT INTO ontime_replica_all SELECT * FROM ontime;
   ```

   

5. 每台机器数据总数一致

   




# 7. IP 访问策略

```shell
iptables -I INPUT -p tcp --dport 9000 -j DROP
iptables -I INPUT -p tcp --dport 8123 -j DROP
iptables -I INPUT -p tcp --dport 9004 -j DROP
iptables -I INPUT -s 180.101.0.0/16 -p tcp --dport 2128 -j ACCEPT
iptables -I INPUT -s 180.101.0.0/16 -p tcp --dport 2888 -j ACCEPT
iptables -I INPUT -s 180.101.0.0/16 -p tcp --dport 3888 -j ACCEPT
iptables -I INPUT -s 180.101.0.0/16 -p tcp --dport 9000 -j ACCEPT
iptables -I INPUT -s 127.0.0.0/16 -p tcp --dport 9000 -j ACCEPT

iptables -I INPUT -s 59.61.78.224/28 -p tcp --dport 8123 -j ACCEPT
iptables -I INPUT -s 59.61.78.128/28 -p tcp --dport 8123 -j ACCEPT
iptables -I INPUT -s 59.61.78.0/24 -p tcp --dport 8123 -j ACCEPT
iptables -I INPUT -s 218.85.123.224/27 -p tcp --dport 8123 -j ACCEPT
iptables -I INPUT -s 218.107.205.208/28 -p tcp --dport 8123 -j ACCEPT
iptables -I INPUT -s 183.253.10.80/28 -p tcp --dport 8123 -j ACCEPT





service iptables save
chkconfig iptables on
```



# 8.ClickHouse 问题

1.磁盘空间不足的情况, 比较笨的办法是手动创建表或者库，软连接到其他盘

2.如何使用多块磁盘，通过DB、或者Table 软连接到其他盘

# 9. 常用命令

```sql
--1.手动创建用户命令
CREATE USER upa_tmp@'0.0.0.0' IDENTIFIED WITH sha256_password BY '123456';
--2.授权
GRANT SELECT(date, imei, idfa, keyword, city, province, mac) ON UPA.XXXX  TO upa_tmp WITH GRANT OPTION;
--3.远程插入
insert into UPA.XXXXX
SELECT *
FROM remote('127.0.0.1', 'UPA.UPA_xxxx', 'user', 'password') insert
into UPA.XXXXX
--4 文件导入数据库
cat $path|grep -v '#' |  clickhouse-client -h 127.0.0.1  -u user --password pwd  --query "insert into UPA.xxx FORMAT TabSeparated"
```





# 10.数据修复

https://clickhouse.com/docs/zh/engines/table-engines/mergetree-family/replication/

参考上诉链接

从另外一个副本拷贝sql数据过来， 在重启
