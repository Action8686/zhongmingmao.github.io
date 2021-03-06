---
title: MySQL -- 基于Docker搭建主从集群
date: 2019-02-23 16:43:10
categories:
    - Storage
    - MySQL
tags:
    - Storage
    - MySQL
---

## 目录结构
```
$ tree
.
├── master
│   ├── data
│   └── master.cnf
└── slave
    ├── data
    └── slave.cnf
```

<!-- more -->

### master.cnf
```
[mysqld]
pid-file    = /var/run/mysqld/mysqld.pid
socket      = /var/run/mysqld/mysqld.sock
datadir     = /var/lib/mysql
server-id=1
log-bin=master-bin
gtid_mode=on
enforce_gtid_consistency=on
```

### slave.cnf
```
[mysqld]
pid-file    = /var/run/mysqld/mysqld.pid
socket      = /var/run/mysqld/mysqld.sock
datadir     = /var/lib/mysql
server-id=2
log-bin=slave-bin
read-only=1
relay_log=relay-bin
log-slave-updates=1
gtid_mode=on
enforce_gtid_consistency=on
```

## 启动容器
```
$ docker run --name mysql_master -d -e MYSQL_ROOT_PASSWORD=123456 -v ~/mysql/master/data:/var/lib/mysql -v ~/mysql/master/master.cnf:/etc/mysql/mysql.conf.d/master.cnf mysql:5.7
519a9c6d2d6bb03916fb7659d3a5be86a179e6569d6545215517720665da23f0

$ docker run --name mysql_slave -d -e MYSQL_ROOT_PASSWORD=123456 -v ~/mysql/slave/data:/var/lib/mysql -v ~/mysql/slave/slave.cnf:/etc/mysql/mysql.conf.d/slave.cnf mysql:5.7
d087a00e01a93a816b023c4f2d87a15cc475446e8822031ae70e59d654cf651e

$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                 NAMES
d087a00e01a9        mysql:5.7           "docker-entrypoint.s…"   9 seconds ago       Up 8 seconds        3306/tcp, 33060/tcp   mysql_slave
519a9c6d2d6b        mysql:5.7           "docker-entrypoint.s…"   24 seconds ago      Up 23 seconds       3306/tcp, 33060/tcp   mysql_master

$ docker inspect --format='{{.NetworkSettings.IPAddress}}' mysql_master mysql_slave
172.17.0.2
172.17.0.3
```

## 主库

### 添加复制账号
```sql
$ docker exec -it mysql_master bash
root@519a9c6d2d6b:/# mysql -uroot -p123456

mysql> GRANT REPLICATION SLAVE ON *.* to 'replication'@'%' IDENTIFIED BY '123456';
Query OK, 0 rows affected, 1 warning (0.03 sec)

mysql> SHOW WARNINGS;
+---------+------+------------------------------------------------------------------------------------------------------------------------------------+
| Level   | Code | Message                                                                                                                            |
+---------+------+------------------------------------------------------------------------------------------------------------------------------------+
| Warning | 1287 | Using GRANT for creating new user is deprecated and will be removed in future release. Create new user with CREATE USER statement. |
+---------+------+------------------------------------------------------------------------------------------------------------------------------------+
```

### 查看binlog位置
```sql
mysql> SHOW MASTER STATUS;
+-------------------+----------+--------------+------------------+------------------------------------------+
| File              | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                        |
+-------------------+----------+--------------+------------------+------------------------------------------+
| master-bin.000003 |      484 |              |                  | b0bda503-3cf1-11e9-8c3a-0242ac110002:1-6 |
+-------------------+----------+--------------+------------------+------------------------------------------+
```

## 从库

### 配置同步信息
```sql
$ docker exec -it mysql_slave bash
root@d087a00e01a9:/# mysql -uroot -p123456

-- 基于位点
-- mysql> CHANGE MASTER TO master_host='172.17.0.2',master_user='replication',master_password='123456',master_log_file='master-bin.000003',master_log_pos=484,master_port=3306;

-- 基于GTID
mysql> CHANGE MASTER TO master_host='172.17.0.2',master_user='replication',master_password='123456',master_auto_position=1,master_port=3306;
Query OK, 0 rows affected, 2 warnings (0.06 sec)

-- Slave_IO_Running=No, Slave_SQL_Running=No
mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: 172.17.0.2
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File:
          Read_Master_Log_Pos: 4
               Relay_Log_File: relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File:
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 0
              Relay_Log_Space: 154
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 0
                  Master_UUID:
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set: ba0b2f12-3cf1-11e9-9c40-0242ac110003:1-5
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
```

### 开启同步
```sql
mysql> START SLAVE;
Query OK, 0 rows affected (0.01 sec)

-- Slave_IO_Running=Yes, Slave_SQL_Running=Yes, Seconds_Behind_Master=0
mysql> SHOW SLAVE STATUS\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.2
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000003
          Read_Master_Log_Pos: 484
               Relay_Log_File: relay-bin.000003
                Relay_Log_Pos: 699
        Relay_Master_Log_File: master-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 484
              Relay_Log_Space: 3058612
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: b0bda503-3cf1-11e9-8c3a-0242ac110002
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: b0bda503-3cf1-11e9-8c3a-0242ac110002:1-6
            Executed_Gtid_Set: b0bda503-3cf1-11e9-8c3a-0242ac110002:1-6,
ba0b2f12-3cf1-11e9-9c40-0242ac110003:1-5
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
```

## 验证
```
$ docker exec mysql_slave mysql -uroot -p123456 -e "SHOW DATABASES"
Database
information_schema
mysql
performance_schema
sys

$ docker exec mysql_master mysql -uroot -p123456 -e "CREATE DATABASE test"

$ docker exec mysql_slave mysql -uroot -p123456 -e "SHOW DATABASES"
Database
information_schema
mysql
performance_schema
sys
test
```

<!-- indicate-the-source -->
