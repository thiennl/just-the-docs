---
layout: default
title: Zimbra Policyd Mysql
parent: Mail
nav_order: 3
---
@Domain dinguyen.com
@Kiểm tra bind-address
```
[root@mail ~]# vi /opt/zimbra/conf/my.cnf
```

```
[mysqld]

basedir        = /opt/zimbra/mariadb
datadir        = /opt/zimbra/db/data
socket         = /opt/zimbra/db/mysql.sock
pid-file       = /opt/zimbra/db/mysql.pid
bind-address   = 127.0.0.1
port           = 7306
user           = zimbra
tmpdir         = /opt/zimbra/data/tmp
```

@Tạo db và user policyd
```
[root@mail ~]# su - zimbra
```

```
[zimbra@mail ~]$ mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 1644
Server version: 10.0.15-MariaDB-log Zimbra binary distribution

Copyright (c) 2000, 2014, Oracle, SkySQL Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
```

```
MariaDB [(none)]> show databases;
MariaDB [(none)]> create database policyd CHARACTER SET 'UTF8';
MariaDB [(none)]> show databases;
MariaDB [(none)]> create user 'policyd'@'127.0.0.1' identified by 'vNpt_2o2i';
Query OK, 0 rows affected (0.96 sec)

MariaDB [(none)]> grant all privileges on policyd.* to 'policyd'@'127.0.0.1' identified by 'vNpt_2o2i' with grant option;
Query OK, 0 rows affected (0.06 sec)

MariaDB [(none)]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> quit;
Bye
```

@Convert .tsql files to .sql file
```
[zimbra@mail database]$ cd /opt/zimbra/common/share/database/
[zimbra@mail database]$ ll
total 48
-rw-r--r-- 1 root root 1272 Dec 31  2015 access_control.tsql
-rw-r--r-- 1 root root 2755 Dec 31  2015 accounting.tsql
-rw-r--r-- 1 root root 4147 Dec 31  2015 amavis.tsql
-rw-r--r-- 1 root root 3227 Dec 31  2015 checkhelo.tsql
-rw-r--r-- 1 root root 1295 Dec 31  2015 checkspf.tsql
-rwxr-xr-x 1 root root 3116 Dec 31  2015 convert-tsql
-rw-r--r-- 1 root root 4882 Dec 31  2015 core.tsql
-rw-r--r-- 1 root root 4417 Dec 31  2015 greylisting.tsql
-rw-r--r-- 1 root root 3137 Dec 31  2015 quotas.tsql
drwxr-xr-x 2 root root  143 Feb 26  2019 whitelists
```

```
[zimbra@mail database]$POLICYDTABLESSQL="$(mktemp /tmp/policyd-dbtables.XXXXXXXX.sql)"

[zimbra@mail database]$for i in core.tsql access_control.tsql quotas.tsql amavis.tsql checkhelo.tsql checkspf.tsql greylisting.tsql accounting.tsql;
 do  
 ./convert-tsql mysql $i;             
 done > "${POLICYDTABLESSQL}"
```

```
[root@mail tmp]# ll policyd-dbtables.vl6f9V4E.sql 
-rw------- 1 root root 24846 Mar 19 13:26 policyd-dbtables.vl6f9V4E.sql
```
@import db
```
[zimbra@mail ~]$ /opt/zimbra/bin/mysql policyd < /tmp/policyd-dbtables.vl6f9V4E.sql
```
@Kiểm tra db
```
[zimbra@mail ~]$ mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 1782
Server version: 10.0.15-MariaDB-log Zimbra binary distribution

Copyright (c) 2000, 2014, Oracle, SkySQL Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
| mboxgroup98        |
| mboxgroup99        |
| mysql              |
| performance_schema |
| policyd            |
| test               |
| zimbra             |
+--------------------+
106 rows in set (0.00 sec)
MariaDB [(none)]> use policyd;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [policyd]> show tables;
+---------------------------+
| Tables_in_policyd         |
+---------------------------+
| access_control            |
| accounting                |
| accounting_tracking       |
| amavis_rules              |
| checkhelo                 |
| checkhelo_blacklist       |
| checkhelo_tracking        |
| checkhelo_whitelist       |
| checkspf                  |
| greylisting               |
| greylisting_autoblacklist |
| greylisting_autowhitelist |
| greylisting_tracking      |
| greylisting_whitelist     |
| policies                  |
| policy_group_members      |
| policy_groups             |
| policy_members            |
| quotas                    |
| quotas_limits             |
| quotas_tracking           |
| session_tracking          |
+---------------------------+
22 rows in set (0.00 sec)

MariaDB [policyd]> quit
Bye
```
@Chỉnh thông số trong file /opt/zimbra/conf/cbpolicyd.conf.in
```
#Gán user db
[zimbra@mail ~]$ grep -lZr -e ".*sername=.*$" "/opt/zimbra/conf/cbpolicyd.conf.in" | xargs -0 sed -i "s^.*sername=.*$^Username=policyd^g"
#Gán password
[zimbra@mail ~]$ grep -lZr -e ".*assword=.*$" "/opt/zimbra/conf/cbpolicyd.conf.in"  | xargs -0 sed -i "s^.*assword=.*$^Password=vNpt_2o2i^g"
#Gán db
[zimbra@mail ~]$ grep -lZr -e "DSN=.*$" "/opt/zimbra/conf/cbpolicyd.conf.in" | xargs -0 sed -i "s^DSN=.*$^DSN=DBI:mysql:database=policyd;host=127.0.0.1;port=7306^g"
#Kiểm tra lại kết quả
[zimbra@mail ~]$ vi /opt/zimbra/conf/cbpolicyd.conf.in
```

---
Cấu hình các rules
@Xem db policyd tables
```
[zimbra@mail ~]$ mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 1818
Server version: 10.0.15-MariaDB-log Zimbra binary distribution

Copyright (c) 2000, 2014, Oracle, SkySQL Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use policyd;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [policyd]> show tables;
+---------------------------+
| Tables_in_policyd         |
+---------------------------+
| access_control            |
| accounting                |
| accounting_tracking       |
| amavis_rules              |
| checkhelo                 |
| checkhelo_blacklist       |
| checkhelo_tracking        |
| checkhelo_whitelist       |
| checkspf                  |
| greylisting               |
| greylisting_autoblacklist |
| greylisting_autowhitelist |
| greylisting_tracking      |
| greylisting_whitelist     |
| policies                  |
| policy_group_members      |
| policy_groups             |
| policy_members            |
| quotas                    |
| quotas_limits             |
| quotas_tracking           |
| session_tracking          |
+---------------------------+
22 rows in set (0.00 sec)
```
@Sẽ thay đổi nội dung các tables sau:
```
| policies                  |
| policy_group_members      |
| policy_groups             |
| policy_members            |
| quotas                    |
| quotas_limits             |
```

@Xem các policy groups đang có
```
MariaDB [policyd]> select * from policy_groups;
+----+------------------+----------+---------+
| ID | Name             | Disabled | Comment |
+----+------------------+----------+---------+
|  1 | internal_ips     |        0 | NULL    |
|  2 | internal_domains |        0 | NULL    |
+----+------------------+----------+---------+
2 rows in set (0.00 sec)
```
@Thêm Policy group list_domain
```
MariaDB [policyd]> INSERT INTO policy_groups (ID, Name,Disabled,Comment) VALUES(3, 'list_domain', 0, 'NULL');
Query OK, 1 row affected (0.00 sec)
MariaDB [policyd]> select * from policy_groups;
+----+------------------+----------+---------+
| ID | Name             | Disabled | Comment |
+----+------------------+----------+---------+
|  1 | internal_ips     |        0 | NULL    |
|  2 | internal_domains |        0 | NULL    |
|  3 | list_domain      |        0 | NULL    |
+----+------------------+----------+---------+
3 rows in set (0.00 sec)
```
@Kiểm tra các Policy group member
```
MariaDB [policyd]> select * from policy_group_members ;
+----+---------------+--------------+----------+---------+
| ID | PolicyGroupID | Member       | Disabled | Comment |
+----+---------------+--------------+----------+---------+
|  1 |             1 | 10.0.0.0/8   |        0 | NULL    |
|  2 |             2 | @example.org |        0 | NULL    |
|  3 |             2 | @example.com |        0 | NULL    |
+----+---------------+--------------+----------+---------+
3 rows in set (0.00 sec)
```
@Xóa các member và thêm member @dinguyen.com
```
MariaDB [policyd]> DELETE FROM `policy_group_members` WHERE Member='@example.org';
Query OK, 1 row affected (0.00 sec)
MariaDB [policyd]> DELETE FROM `policy_group_members` WHERE Member='@example.com';
Query OK, 1 row affected (0.00 sec)
MariaDB [policyd]> DELETE FROM `policy_group_members` WHERE Member='10.0.0.0/8';
Query OK, 1 row affected (0.00 sec)
MariaDB [policyd]> INSERT INTO `policy_group_members`(ID,PolicyGroupID,Member,Disabled,Comment) VALUES (1,3,'@dinguyen.com',0,'NULL');
Query OK, 1 row affected (0.00 sec)
MariaDB [policyd]> select * from policy_group_members ;
+----+---------------+--------------------+----------+---------+
| ID | PolicyGroupID | Member             | Disabled | Comment |
+----+---------------+--------------------+----------+---------+
|  1 |             3 | @dinguyen.com |        0 | NULL    |
+----+---------------+--------------------+----------+---------+
1 row in set (0.00 sec)
```
#Notes: PolicyGroupID  = 3 --> chính là ID của list_domain trong policy_groups;
@Kiểm tra các polices
```
MariaDB [policyd]> select * from  policies;
+----+------------------+----------+--------------------------------+----------+
| ID | Name             | Priority | Description                    | Disabled |
+----+------------------+----------+--------------------------------+----------+
|  1 | Default          |        0 | Default System Policy          |        0 |
|  2 | Default Outbound |       10 | Default Outbound System Policy |        0 |
|  3 | Default Inbound  |       10 | Default Inbound System Policy  |        0 |
|  4 | Default Internal |       20 | Default Internal System Policy |        0 |
|  5 | Test             |       50 | Test policy                    |        0 |
+----+------------------+----------+--------------------------------+----------+
```
@Kiểm tra các policy members
```
MariaDB [policyd]> select * from  policy_members ;
MariaDB [policyd]> select * from  policy_members ;
+----+----------+-----------------------------------+--------------------+---------+----------+
| ID | PolicyID | Source                            | Destination        | Comment | Disabled |
+----+----------+-----------------------------------+--------------------+---------+----------+
|  1 |        1 | NULL                              | NULL               | NULL    |        0 |
|  2 |        2 | %internal_ips,%internal_domains   | !%internal_domains | NULL    |        0 |
|  3 |        3 | !%internal_ips,!%internal_domains | %internal_domains  | NULL    |        0 |
|  4 |        4 | %internal_ips,%internal_domains   | %internal_domains  | NULL    |        0 |
|  5 |        5 | @example.net                      | NULL               | NULL    |        0 |
+----+----------+-----------------------------------+--------------------+---------+----------+
5 rows in set (0.00 sec)
@Disable tất cả các policy members trên
MariaDB [policyd]> UPDATE policy_members SET Disabled = 1 WHERE PolicyID = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
MariaDB [policyd]> UPDATE policy_members SET Disabled = 1 WHERE PolicyID = 2;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [policyd]> UPDATE policy_members SET Disabled = 1 WHERE PolicyID = 3;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [policyd]> UPDATE policy_members SET Disabled = 1 WHERE PolicyID = 4;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [policyd]> UPDATE policy_members SET Disabled = 1 WHERE PolicyID = 5;
Query OK, 1 row affected (0.29 sec)
Rows matched: 1  Changed: 1  Warnings: 0
MariaDB [policyd]> select * from  policy_members ;
+----+----------+-----------------------------------+--------------------+---------+----------+
| ID | PolicyID | Source                            | Destination        | Comment | Disabled |
+----+----------+-----------------------------------+--------------------+---------+----------+
|  1 |        1 | NULL                              | NULL               | NULL    |        1 |
|  2 |        2 | %internal_ips,%internal_domains   | !%internal_domains | NULL    |        1 |
|  3 |        3 | !%internal_ips,!%internal_domains | %internal_domains  | NULL    |        1 |
|  4 |        4 | %internal_ips,%internal_domains   | %internal_domains  | NULL    |        1 |
|  5 |        5 | @example.net                      | NULL               | NULL    |        1 |
+----+----------+-----------------------------------+--------------------+---------+----------+
5 rows in set (0.00 sec)
```
@Thêm 2 policy members:
From: %list_domain to: !%list_domain 	--> Gửi từ mail có domain trong list_domain đến các mail khác domain
From: !%list_domain  to: any 	--> Gửi từ mail có domain 0 nằm trong list_domain đến các mail khác
```
MariaDB [policyd]> INSERT INTO policy_members (ID,PolicyID,Source,Destination,Comment,Disabled) VALUES(6,1, '%list_domain', '!%list_domain', 'NULL', 0);
Query OK, 1 row affected (0.00 sec)

MariaDB [policyd]> INSERT INTO policy_members (ID,PolicyID,Source,Destination,Comment,Disabled) VALUES(7,1, '!%list_domain', 'any', 'NULL', 0);
Query OK, 1 row affected (0.00 sec)
```

```
MariaDB [policyd]> select * from  policy_members ;
+----+----------+-----------------------------------+--------------------+---------+----------+
| ID | PolicyID | Source                            | Destination        | Comment | Disabled |
+----+----------+-----------------------------------+--------------------+---------+----------+
|  1 |        1 | NULL                              | NULL               | NULL    |        1 |
|  2 |        2 | %internal_ips,%internal_domains   | !%internal_domains | NULL    |        1 |
|  3 |        3 | !%internal_ips,!%internal_domains | %internal_domains  | NULL    |        1 |
|  4 |        4 | %internal_ips,%internal_domains   | %internal_domains  | NULL    |        1 |
|  5 |        5 | @example.net                      | NULL               | NULL    |        1 |
|  6 |        1 | %list_domain                      | !%list_domain      | NULL    |        0 |
|  7 |        1 | !%list_domain                     | any                | NULL    |        0 |
+----+----------+-----------------------------------+--------------------+---------+----------+
```
Note: PolicyID=1 --> là ID của Default trong bảng policies.
@Gán Quotas
#Xem các quotas đang có
```
MariaDB [policyd]> select * from  quotas;
+----+----------+-------------------+-----------------------+--------+---------+------+-----------+---------+----------+
| ID | PolicyID | Name              | Track                 | Period | Verdict | Data | LastQuota | Comment | Disabled |
+----+----------+-------------------+-----------------------+--------+---------+------+-----------+---------+----------+
|  1 |        5 | Recipient quotas  | Recipient:user@domain |   3600 | REJECT  | NULL |         0 | NULL    |        0 |
|  2 |        5 | Quota on all /24s | SenderIP:/24          |   3600 | REJECT  | NULL |         0 | NULL    |        0 |
+----+----------+-------------------+-----------------------+--------+---------+------+-----------+---------+----------+
2 rows in set (0.00 sec)
```
#Thêm quota rate_limit
```
MariaDB [policyd]> INSERT INTO quotas (ID,PolicyID,Name,Track,Period,Verdict,Data,LastQuota,Comment,Disabled) VALUES(3,1,'rate_limit','Sender:user@domain',3600, 'REJECT','NULL',0,'NULL',0);
Query OK, 1 row affected (0.00 sec)
MariaDB [policyd]> select * from  quotas;
+----+----------+-------------------+-----------------------+--------+---------+------+-----------+---------+----------+
| ID | PolicyID | Name              | Track                 | Period | Verdict | Data | LastQuota | Comment | Disabled |
+----+----------+-------------------+-----------------------+--------+---------+------+-----------+---------+----------+
|  1 |        5 | Recipient quotas  | Recipient:user@domain |   3600 | REJECT  | NULL |         0 | NULL    |        0 |
|  2 |        5 | Quota on all /24s | SenderIP:/24          |   3600 | REJECT  | NULL |         0 | NULL    |        0 |
|  3 |        1 | rate_limit        | Sender:user@domain    |   3600 | REJECT  | NULL |         0 | NULL    |        0 |
+----+----------+-------------------+-----------------------+--------+---------+------+-----------+---------+----------+
3 rows in set (0.00 sec)
```

@Disable 2 quota ID=1 và ID=2
```
MariaDB [policyd]> UPDATE quotas SET Disabled = 1 WHERE PolicyID = 5;
Query OK, 2 rows affected (1.10 sec)
Rows matched: 2  Changed: 2  Warnings: 0

MariaDB [policyd]> select * from quotas ;
+----+----------+-------------------+-----------------------+--------+---------+------+-----------+---------+----------+
| ID | PolicyID | Name              | Track                 | Period | Verdict | Data | LastQuota | Comment | Disabled |
+----+----------+-------------------+-----------------------+--------+---------+------+-----------+---------+----------+
|  1 |        5 | Recipient quotas  | Recipient:user@domain |   3600 | REJECT  | NULL |         0 | NULL    |        1 |
|  2 |        5 | Quota on all /24s | SenderIP:/24          |   3600 | REJECT  | NULL |         0 | NULL    |        1 |
|  3 |        1 | rate_limit        | Sender:user@domain    |   3600 | REJECT  | NULL |         0 | NULL    |        0 |
+----+----------+-------------------+-----------------------+--------+---------+------+-----------+---------+----------+
3 rows in set (0.00 sec)
```
@Xem các qouta limit đang có
```
MariaDB [policyd]> select * from quotas_limits ;
+----+----------+-----------------------+--------------+---------+----------+
| ID | QuotasID | Type                  | CounterLimit | Comment | Disabled |
+----+----------+-----------------------+--------------+---------+----------+
|  1 |        1 | MessageCount          |           10 | NULL    |        0 |
|  2 |        1 | MessageCumulativeSize |         8000 | NULL    |        0 |
|  3 |        2 | MessageCount          |           12 | NULL    |        0 |
+----+----------+-----------------------+--------------+---------+----------+
3 rows in set (0.00 sec)
```
@Disable 3 quota limits trên
```
MariaDB [policyd]> UPDATE quotas_limits SET Disabled = 1 WHERE ID = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [policyd]> UPDATE quotas_limits SET Disabled = 1 WHERE ID = 2;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [policyd]> UPDATE quotas_limits SET Disabled = 1 WHERE ID = 3;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [policyd]> select * from quotas_limits ;
+----+----------+-----------------------+--------------+---------+----------+
| ID | QuotasID | Type                  | CounterLimit | Comment | Disabled |
+----+----------+-----------------------+--------------+---------+----------+
|  1 |        1 | MessageCount          |           10 | NULL    |        1 |
|  2 |        1 | MessageCumulativeSize |         8000 | NULL    |        1 |
|  3 |        2 | MessageCount          |           12 | NULL    |        1 |
+----+----------+-----------------------+--------------+---------+----------+
3 rows in set (0.00 sec)
```
@Thêm quota limit count = 160 cho quota rate_limit ( có ID = 3 trong bảng quotas)
```
MariaDB [policyd]> INSERT INTO quotas_limits (ID,QuotasID,Type,CounterLimit,Comment,Disabled) VALUES(4,3,'MessageCount',160,'NULL',0);
Query OK, 1 row affected (0.00 sec)
MariaDB [policyd]> select * from quotas_limits ;
+----+----------+-----------------------+--------------+---------+----------+
| ID | QuotasID | Type                  | CounterLimit | Comment | Disabled |
+----+----------+-----------------------+--------------+---------+----------+
|  1 |        1 | MessageCount          |           10 | NULL    |        1 |
|  2 |        1 | MessageCumulativeSize |         8000 | NULL    |        1 |
|  3 |        2 | MessageCount          |           12 | NULL    |        1 |
|  4 |        3 | MessageCount          |          160 | NULL    |        0 |
+----+----------+-----------------------+--------------+---------+----------+
4 rows in set (0.00 sec)
```
@Enable policyd
```
[root@mail ~]#su - zimbra
[zimbra@mail ~]$zmprov ms `zmhostname` +zimbraServiceEnabled cbpolicyd
[zimbra@mail ~]$zmprov ms `zmhostname` zimbraCBPolicydQuotasEnabled TRUE
[zimbra@mail ~]$zmcontrol restart
```
@Kiểm tra log
```
[zimbra@mail ~]$ tail -f /opt/zimbra/log/cbpolicyd.log
```
