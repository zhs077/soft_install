---
title: Elasticsearch 部署
date: 2020-03-01 18:34:28
tags:
categories:

- 部署
- 数据库
---
# 1.前置条件

   jdk版本：jdk1.8,  elasticsearch 6.4.2, kibana 6.4.2

​	操作系统：centos7.6

1. 安装：

   `rpm -U jdk-8u152-linux-x64.rpm` 

2. 如果安装机器版本不一致需要卸载旧版本

   `rpm -e java-1.7.0-openjdk java-1.7.0-openjdk-headless`


3. 添加JAVA_HOME

```shell
echo "export JAVA_HOME=/usr/java/jdk1.8.0_152/" >> /etc/profile
echo "export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar" >> /etc/profile
echo "export PATH=$JAVA_HOME/bin:$PATH" >> /etc/profile
source /etc/profile
```

4. 检测环境变量是否生效

```shell
[root@PSjssqsjzxtw71 ~]# echo $JAVA_HOME
/usr/java/jdk1.8.0_152/
```

# 2.安装elasticsearch

1. 解锁用户

   `chattr -i /etc/group;chattr -i /etc/gshadow;chattr -i /etc/passwd;chattr -i /etc/shadow`

2. 安装rpm

   `rpm -U elasticsearch-6.4.2.rpm`

3. 加锁

   chattr +i /etc/group;chattr +i /etc/gshadow;chattr +i /etc/passwd;chattr +i /etc/shadow

4. 设置存储盘

   ```
   mkdir -p /cache3/elasticsearch_data;chown -R elasticsearch:elasticsearch /cache3/elasticsearch_data/
   mkdir -p /cache4/elasticsearch_data;chown -R elasticsearch:elasticsearch /cache4/elasticsearch_data/
   mkdir -p /cache7/elasticsearch_data;chown -R elasticsearch:elasticsearch /cache7/elasticsearch_data/
   mkdir -p /cache8/elasticsearch_data;chown -R elasticsearch:elasticsearch /cache8/elasticsearch_data/
   mkdir -p /cache9/elasticsearch_data;chown -R elasticsearch:elasticsearch /cache9/elasticsearch_data/
   
   ```

   

5. 设置jvm内存

   /etc/elasticsearch/jvm.options

   ```shell
   # Xms represents the initial size of total heap space
   # Xmx represents the maximum size of total heap space
   
   -Xms25g
   -Xmx25g
   ```

   

6. 尽量不使用机器交换区

   ```shell
   echo 'vm.swappiness=0'>> /etc/sysctl.conf
   sysctl -p
   ```

   

7. 主配置文件设置

   /etc/elasticsearch/elasticsearch.yml

   ```
   ##集群名称整个集群唯一
   cluster.name: upa-cluster-s1 
   #节点名称，不能重复
   node.name: node-127.0.0.1
   #是否是主节点
   node.master: true 
   #如果是主节点，就不要设置数据节点
   node.data: false
   path.data:["/cache3/elasticsearch_data","/cache4/elasticsearch_data","/cache7/elasticsearch_data","/cache8/elasticsearch_data","/cache9/elasticsearch_data"]
   path.logs: /var/log/elasticsearch
   network.host:127.0.0.1
   http.port:48841
   transport.tcp.port:48862
   discovery.zen.ping.unicast.hosts:
   xpack.security.enabled: true
   xpack.security.transport.ssl.enabled: true
   xpack.security.transport.ssl.verification_mode: certificate 
   xpack.security.transport.ssl.keystore.path: /etc/elasticsearch/cert/upa-cluster-certificates.p12
   xpack.security.transport.ssl.truststore.path: /etc/elasticsearch/cert/upa-cluster-certificates.p12
   
   ```

   

8. 生成证书(第一台机器部署时需要)

    /etc/elasticsearch/cert/upa-cluster-certificates.p12

   ```
   cd /etc/elasticsearch/cert
   /usr/share/elasticsearch/bin/elasticsearch-certutil ca 
   
   生成 upa-cluster-certificates.p12
   修改权限: chmod 644 upa-cluster-certificates.p12 
   ```

   

9. SSL证书部署

   `chown -R elasticsearch:elasticsearch /etc/elasticsearch/cert/`

10. 每台机器设置http+tcp ssl通信密码

    ```shell
    /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
    
    /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
    
    /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.keystore.secure_password
    
    /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.http.ssl.truststore.secure_password
    
    备注：密码为生成证书输入的密码
    ```

    

11. 添加证书license.json(第一台机器部署时需要)

    ```shell
    curl -XPUT 'http://180.101.24.71:48841/_xpack/license?pretty'  -H 'Content-Type: application/json' -u 'elastic:changeme' -d @/CNCLog/UserInfoMerge/ES/license.json
    ```

    备注：elastic:changeme 为新集群的初始用户名和密码

    license.json 为x-pack 权限破解后，需要向es集群注册，注册成功后，es集群的x-pack 权限功能将变成铂金版；

    

12. 为新集群创建密码（自动生成方式，需自己保存）(第一台机器部署时需要)

    ```
    /usr/share/elasticsearch/bin/elasticsearch-setup-passwords auto
    备注：需要有数据节点，否则会超时失败
    ```

    

13. ES创建完毕

    ```shell
    curl "http://127.0.0.1:48841/_cluster/stats" -u 'elastic:xxx'
    ```

    

    

# 3.安装kibana

1. 解锁

   `chattr -i /etc/group;chattr -i /etc/gshadow;chattr -i /etc/passwd;chattr -i /etc/shadow`

2. 安装rpm

   `rpm -U kibana-6.4.2-x86_64.rpm` 

3. 解锁

   `chattr +i /etc/group;chattr +i /etc/gshadow;chattr +i /etc/passwd;chattr +i /etc/shadow`

4. 配置文件

   /etc/kibana/kibana.yml

   ```shell
   server.port: 48866
   xpack.monitoring.enabled: true
   server.host: xx
   xpack.security.enabled: true
   elasticsearch.username: "kibana"
   elasticsearch.password: "xx"
   elasticsearch.url: "http://xx:48841"
   
   备注：
   elasticsearch.username: "xx" #配置成kibana 角色用户名即可
   elasticsearch.password: "xx"  #配置成kibana 角色密码即可
   
   ```

5. 启动服务

   `service  kibana start`

   

# 4.安装sentinl

1. 安装插件

   `/usr/share/kibana/bin/kibana-plugin install  https://github.com/sirensolutions/sentinl/releases/download/tag-6.4.2-0/sentinl-v6.4.2.zip`

2. 



# 5.问题

1. 如果修改集群密码

   先生成一个超级用户，在通过超级用户去修改密码

   ```shell
   service elasticsearch stop
   /usr/share/elasticsearch/bin/elasticsearch-users  useradd my_admin -p my_password -r superuser
   curl -u my_admin -XPUT 'http://XXXX:48841/_xpack/security/user/elastic/_password?pretty' -H 'Content-Type: application/json' -d'{"password" : "xxxx"}'
   curl -u elastic 'http://XXXX:48841/_xpack/security/_authenticate?pretty'
   
   ```

2. 文件权限问题

   /etc/elasticsearch/elasticsearch.yml

    /etc/elasticsearch/jvm.options 

   

3. 创建新用户以及权限

   需要用户权限控制的时候，需要创建角色，并赋予角色一定的权限控制

   ```shell
   curl -XPOST -u 'elastic:xx' 'http://xxx.xx:48841/_xpack/security/role/upa_admin' -H "Content-Type: application/json" -d '{
     "indices" : [
       {
         "names" : [ "upa*" ],
         "privileges" : [ "all" ]
       },
       {
         "names" : [ ".kibana*" ],
         "privileges" : [ "manage", "read", "index" ]
       }
     ]
   }'
   
   ```

   说明：创建角色upa_admin，并赋予upa*开头的index_name全部权限

   ```shell
   curl -X POST "http://xxx.xx:48841/_xpack/security/user/wsupa" -H 'Content-Type: application/json' -u 'elastic:xx -d'
   {
     "password" : "xx",
     "roles" : [ "upa_admin" ]
   }
   '
   说明：给用户名为wsupa的用户，赋予角色upa_admin权限，并创建wsupa的用户名密码
   
   ```

   

4. 删除角色

   curl -XDELETE -u elastic:xx 'xxxxx:48841/_xpack/security/role/upa_admin'

5. Kibana 6.4.2 plugin:security Privileges are missing and can’t be removed, currently

   ```shell
    curl -XDELETE "http://180.xxx.xxxx:48841/_xpack/security/privilege/kibana-.kibana/space_all"  -u 'elastic:xxx' 
    curl -XDELETE "http://180.xxx:48841/_xpack/security/privilege/kibana-.kibana/space_read"  -u 'elastic:xxx' 
   ```

   

6. 





# 6.常用操作

## 1.数据迁移

```shell
elasticdump \
  --input=http://user:pwd@oriIP:48841/dmp_ctrip_hotel_info \
  --output=http://user:pwd@destIP:48841/dmp_ctrip_hotel_info  \
  --type=data
```



## 2.数据dumper

```shell
elasticdump \
  --input=http://user:pwd@oriIP:48841/dmp_ctrip_hotel_info \
  --output=/data/my_index.json \
  --type=data
```





