## 使用 `Docker` 创建 `Mycat` 和 `MySQL` 主从服务器
[官方安装指南](https://github.com/MyCATApache/Mycat-Server/wiki/2.1-docker%E5%AE%89%E8%A3%85Mycat)  
[docker 配置参考指南](https://github.com/liuwel/docker-mycat)   
[双主双从配置指南](https://cloud.tencent.com/developer/article/1623821)   
[docker es+logstash+kibana](https://github.com/deviantony/docker-elk)

### 1. 环境
   
- `Mycat`: 1.6.7.6
- `MySQL`: 8.0.20

### 2. 目录

```
├── compose
│     ├── docker-compose.yml
│     └── mycat
│         └── Dockerfile
├── conf
│     ├── hosts
│     ├── mycat
│     │     └──...
│     ├── mysql-m1
│     │     └── conf.d
│     │         └── docker.cnf
│     ├── mysql-s1
│     │     └── conf.d
│     │         └── docker.cnf
│     └── mysql-s2
│           └── conf.d
│               └── docker.cnf
├── logs
│     ├── mycat
│     │     ├── mycat.log
│     │     ├── mycat.pid
│     │     ├── switch.log
│     │     └── wrapper.log
│     ├── mysql-m1
│     ├── mysql-s1
│     └── mysql-s2
├── mysql
│     ├── mysql-m1
│     │     ├── ...
│     ├── mysql-s1
│     │     ├── ...
│     └── mysql-s2
│     │     ├── ...
└── README.md
```

### 3. `MySQL` 主从服务器结构
  
- `mysql-m1` : 主服务器 `IP:172.18.0.2` 
- `mysql-s1` : 从服务器 `slave1 IP:172.18.0.3`
- `mysql-s2` : 从服务器 `slave2 IP:172.18.0.4`
- `mycat`    : `Mycat` 服务器 `IP:172.18.0.5`  

### 4. `hosts` 文件添加解析
  
```shell
172.18.0.2      dbm1
172.18.0.3      dbs1
172.18.0.4      dbs2
172.18.0.5      Python
127.0.0.1       localhost
```

### 5. `docker-compose.yml` 配置文件
   

### 6. 启动容器
构建镜像
```shell
[root@192 docker-Python]# sudo docker-compose build m1 s1 s2
```

运行 
```shell
[root@192 docker-Python]# docker-compose up -d dbm1 dbs1 dbs2
```

### 7. `MySQL` 主从配置

#### 7.1. 配置 `dbm1` 主服务器
```shell
[root@192 docker-Python]# docker exec -it dbm1 /bin/bash
root@dbm1:/# mysql -uroot -p
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.20 MySQL Community Server - GPL

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
mysql>
```

创建用于主从复制的用户 slave，并给 slave 用户授予 slave 的权限
   
`mysql5.7` 写法是 
```
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'172.18.0.%' IDENTIFIED BY '123456';
```    
   
`mysql8` 已经将创建账户和赋予权限的方式分开   
加上 `mysql_native_password` 是因为 `mysql8` 以上版本使用 `caching_sha2_password` 加密方式
```
mysql> CREATE USER 'slave'@'172.18.0.%' IDENTIFIED WITH mysql_native_password BY '123456';
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'slave'@'172.18.0.%';
Query OK, 0 rows affected (0.00 sec)
```

查看 `binlog` 状态 记录 `File` 和 `Position` 状态稍后从库配置的时候会用
```shell
mysql>  show master status;
+-------------------+----------+--------------+------------------+-------------------+
| File              | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-------------------+----------+--------------+------------------+-------------------+
| master-bin.000004 |      710 |              |                  |                   |
+-------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

#### 7.2. 配置一主多从
从库
进入 dbs1 
```shell
[root@192 docker-Python]# docker exec -it dbs1 /bin/sh
root@dbs1:/# mysql -uroot -p
mysql> change master to master_host='dbm1',master_port=3306,master_user='slave',master_password='123456',master_log_file='master-bin.000004',master_log_pos=710;
Query OK, 0 rows affected, 2 warnings (0.05 sec)
mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
```

进入 dbs2
```shell
sudo docker exec -it dbs2 /bin/bash                                                            
root@s2:/# mysql -uroot -p
mysql> change master to master_host='dbm1',master_port=3306,master_user='slave',master_password='123456',master_log_file='master-bin.000004',master_log_pos=710;
Query OK, 0 rows affected, 2 warnings (0.03 sec)

mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
```

### 8. 测试主从
  
登陆主数据库 创建 test_db 数据库 (这个数据库名在稍后的 `mycat` 里面会用到)
```shell
[root@192 docker-Python]# mysql -h dbm1 -uroot -p
MySQL [(none)]> create database test_db;
Query OK, 1 row affected (0.01 sec)
```

进入从库看看数据库是否创建
```shell
[root@192 docker-Python]# mysql -uroot -p123456 -h dbs1
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test_db          |
+--------------------+
5 rows in set (0.00 sec)
```
可以看到从库也已经创建成功了 到这里 mysql 的主从已经配置完成了

### 9. 安装 `mycat` 

#### 9.1. 配置 `mycat`
**重要说明**：`mycat` 在启动时依赖 `/usr/local/mycat/conf` 中的配置文件，
因此需要提前将配置文件放到挂载目录 `./conf/mycat`，
并修改好 `server.xml`，`schema.xml` 中的参数。

看下 `schama.xml` 配置文件
```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
   <!-- 配置1个逻辑库-->
   <schema name="test_db" checkSQLschema="false" sqlMaxLimit="100" dataNode="masterDN" />
   <!-- 逻辑库对应的真实数据库-->
   <dataNode name="masterDN" dataHost="masterDH" database="test_db" />
   <!--真实数据库所在的服务器地址，这里配置了1主2从。主服务器(hostM1)宕机会自动切换到(hostS1) -->
   <dataHost name="masterDH" maxCon="1000" minCon="10" balance="1"
             writeType="0" dbType="mysql" dbDriver="native" switchType="-1" slaveThreshold="100">
      <heartbeat>select user()</heartbeat>
      <writeHost host="dbm1" url="172.18.0.2:3306" user="root" password="123456">
         <readHost host="dbs1" url="172.18.0.3:3306" user="root" password="123456" />
         <readHost host="dbs2" url="172.18.0.4:3306" user="root" password="s2test" />
      </writeHost>
   </dataHost>
</mycat:schema>
```

`server.xml` 配置文件 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
   <user name="mycat" defaultAccount="true">
      <property name="password">123456</property>
      <property name="schemas">test_db</property>
      <property name="defaultSchema">test_db</property>
   </user>
</mycat:server>

```

#### 9.2. 启动 `mycat`
```shell
% cd ~/docker-Python/compose
% sudo docker-compose up -d Python
```

#### 9.3. 测试 `Mycat`
> 进入 `Mycat`
```shell
[root@localhost docker-Python]# docker exec -it dbm1 /bin/sh
# mysql -h Python -umycat -P8066 -p --default_auth=mysql_native_password
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 7
Server version: 5.6.29-Python-1.6.7.6-release-20211118155357 MyCat Server (OpenCloudDB)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+----------+
| DATABASE |
+----------+
| test_db  |
+----------+
1 row in set (0.00 sec)
```

> 多次执行下面的 `sql`，观察 `hostname` 的变化
```shell
mysql> select @@hostname;
+------------+
| @@hostname |
+------------+
| dbs1       |
+------------+
1 row in set (0.00 sec)

mysql> select @@hostname;
+------------+
| @@hostname |
+------------+
| dbs2       |
+------------+
1 row in set (0.00 sec)
```

测试数据
```shell
mysql> CREATE TABLE `test_table`( 
    `id` int(11) NOT NULL AUTO_INCREMENT PRIMARY KEY COMMENT '', 
    `tittle` varchar(255) DEFAULT NULL COMMENT ''
  );
Query OK, 0 rows affected, 1 warning (0.09 sec)                                             
                                                                                     
mysql> show tables;
+-------------------+
| Tables_in_test_db |
+-------------------+
| test_table        |
+-------------------+
1 row in set (0.01 sec)                                                          
                                                                                     
mysql> INSERT INTO `test_table` VALUES (1, 'title1');
Query OK, 1 row affected (0.05 sec)

mysql> INSERT INTO `test_table` VALUES (2, 'title2');
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO `test_table` VALUES (3, 'title3');
Query OK, 1 row affected (0.00 sec)

mysql> select * from test_table;
+----+--------+
| id | title  |
+----+--------+
|  1 | title1 |
|  2 | title2 |
|  3 | title3 |
+----+--------+
3 rows in set (0.01 sec)                                                 
```

### 10. 问题
1. `Mycat` 连接 `dbm1` 连接失败
`mycat.log`:
```shell
2018-07-02 16:31:21.173 INFO [$_NIOREACTOR-25-RW] (io.mycat.sqlengine.SQLJob.connectionError(SQLJob.java:117)) - can't get connection for sql :select user()
2018-07-02 16:31:21.173 WARN [$_NIOREACTOR-27-RW] (io.Python.backend.mysql.nio.MySQLConnectionAuthenticator.handle(MySQLConnectionAuthenticator.java:91)) - can't connect to mysql server ,errmsg:Client does not support authentication protocol requested by server; consider upgrading MySQL client MySQLConnection
```
解决方式：
```shell
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
FLUSH PRIVILEGES;
ALTER USER 'root'@'%' IDENTIFIED BY '123456'; ## 必须执行这个变更密码,否则会报错找不到caching_sha2_password模块
FLUSH PRIVILEGES;
```

### 11. 配置读写分离
删掉刚刚在三个主机数据库中创建的所有数据库，关闭 `dbs2` 的主从复制

#### 11.1. 配置 `Mycat` 的 `schema.xml`
对刚才的文件 `schema.xml` 稍作修改
```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
	<!-- 配置1个逻辑库-->
	<schema name="test_db" checkSQLschema="false" sqlMaxLimit="100">
		<!-- 将 employee 分片，结点跟 host IP 最后一个数字相同 -->
		<table name="employee" primaryKey="ID" dataNode="dn2,dn4" rule="sharding-by-intfile" />
	</schema>
	<!-- 逻辑库对应的真实数据库-->
	<dataNode name="dn2" dataHost="host2" database="test_db" />
	<dataNode name="dn4" dataHost="host4" database="test_db" />
	<!--真实数据库所在的服务器地址，这里配置了1主2从。主服务器(dbm1)宕机会自动切换到(dbs2) -->
	<dataHost name="host2" maxCon="1000" minCon="10" balance="1"
			  writeType="0" dbType="mysql" dbDriver="native" switchType="-1" slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="dbm1" url="172.18.0.2:3306" user="root" password="123456">
			<readHost host="dbs1" url="172.18.0.3:3306" user="root" password="123456" />
			<!--<readHost host="dbs2" url="172.18.0.4:3306" user="root" password="123456" />-->
		</writeHost>
	</dataHost>
	<!-- 将此主机作为切片 分表操作 balance=0表示关闭负载均衡 -->
	<dataHost name="host4" maxCon="1000" minCon="10" balance="0"
			  writeType="0" dbType="mysql" dbDriver="native" switchType="-1" slaveThreshold="100">
		<heartbeat>select user()</heartbeat>
		<writeHost host="dbs2" url="172.18.0.4:3306" user="root" password="123456" />
	</dataHost>
</mycat:schema>
```
因为只有三个主机数据库，所以我将 `dbs2` 从从库中移了出来，作为一个分片数据库。 
表 `employee` 根据字段 `sharding_id` 来分片 
使用的分片规则是 `sharding-by-intfile`，`sharding_id` 做了一定的限制：
```xml
<function name="hash-int" class="io.mycat.route.function.PartitionByFileMap">
    <property name="mapFile">partition-hash-int.txt</property>
</function>
```
可以看出，`hash_int` 规则引用了文件 `partition-hash-int.txt`   
我们来看看这个文件写了什么：
```
10000=0
10010=1
```
也就是说我们的 `sharding_id` 只能是这两个值

#### 11.2. 测试
##### 11.2.1 创建数据库
先在 `dbm1` 和 `dbs2` 各创建一个数据库 `test_db` 和 表 `employee`
```
mysql> create database test_db;
mysql> create table employee(id int(11) auto_increment primary key,`name` varchar(100) NOT NULL DEFAULT '',sharding_id int(11) not null);
```
由于 `dbm1` 和 `dbs1`开启了主从，`dbs1` 会自动创建数据库和数据表   

##### 11.2.3 创建测试数据
进入 `Mycat`，向 `employee` 表添加数据
```sql
insert into employee(name,sharding_id) values('北京', 10000);
insert into employee(name,sharding_id) values('天津', 10010);
insert into employee(name,sharding_id) values('河北', 10000);
insert into employee(name,sharding_id) values('辽宁', 10010);
insert into employee(name,sharding_id) values('吉林', 10000);
insert into employee(name,sharding_id) values('黑龙江', 10010);
insert into employee(name,sharding_id) values('山东', 10000);
insert into employee(name,sharding_id) values('江苏', 10010);
insert into employee(name,sharding_id) values('上海', 10000);
insert into employee(name,sharding_id) values('浙江', 10010);
insert into employee(name,sharding_id) values('安徽', 10000);
insert into employee(name,sharding_id) values('福建', 10010);
insert into employee(name,sharding_id) values('江西', 10000);
insert into employee(name,sharding_id) values('广东', 10010);
insert into employee(name,sharding_id) values('广西', 10000);
```
查询 `dbm1`:
```sql
mysql> select * from employee;
+----+--------+-------------+
| id | name   | sharding_id |
+----+--------+-------------+
|  1 | 北京 |       10000 |
|  2 | 河北 |       10000 |
|  3 | 吉林 |       10000 |
|  4 | 山东 |       10000 |
|  5 | 上海 |       10000 |
|  6 | 安徽 |       10000 |
|  7 | 江西 |       10000 |
|  8 | 广西 |       10000 |
+----+--------+-------------+
8 rows in set (0.00 sec)
```
查询切片 `dbs2`:
```sql
mysql> select * from employee;
+----+-----------+-------------+
| id | name      | sharding_id |
+----+-----------+-------------+
|  1 | 天津    |       10010 |
|  2 | 辽宁    |       10010 |
|  3 | 黑龙江 |       10010 |
|  4 | 江苏    |       10010 |
|  5 | 浙江    |       10010 |
|  6 | 福建    |       10010 |
|  7 | 广东    |       10010 |
+----+-----------+-------------+
7 rows in set (0.03 sec)
```
查询 `dbs1`:
```sql
mysql> select * from employee;
+----+--------+-------------+
| id | name   | sharding_id |
+----+--------+-------------+
|  1 | 北京 |       10000 |
|  2 | 河北 |       10000 |
|  3 | 吉林 |       10000 |
|  4 | 山东 |       10000 |
|  5 | 上海 |       10000 |
|  6 | 安徽 |       10000 |
|  7 | 江西 |       10000 |
|  8 | 广西 |       10000 |
+----+--------+-------------+
8 rows in set (0.00 sec)
```

从查询结果可以看出，`dbm1` 和 `dbs1` 相同，`dbs2` 和 `dbm` 分别按分片规则做了数据存储。


