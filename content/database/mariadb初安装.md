---
title: "Mariadb初安装"
date: 2020-12-20T11:27:39+08:00
draft: false
tags: ["mariadb"]
toc: true
---

# 登录

mariadb初次安装后，默认是只有一个root用户的，且只能从本机"以linux系统用户"登录，需要切换到系统root用户

```bash
zk@ubuntu0001:~ » su
密码：
root@ubuntu0001:/home/zk# mariadb
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 35
Server version: 10.1.44-MariaDB-0ubuntu0.18.04.1 Ubuntu 18.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use mysql
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [mysql]> select Host, User from user;
+-----------+------+
| Host      | User |
+-----------+------+
| localhost | root |
+-----------+------+
1 row in set (0.00 sec)

MariaDB [mysql]>
```

# 改密码

```bash
# 切换到系统root用户执行
mysqladmin -u root password [new_password]
```

# 设置可以从其他机器使用密码登录root用户

## 修改bind-address

初次安装后按默认配置启动，3306端口是监听在本地“127.0.0.1“”的，首先需要修改的就是改为监听在“0.0.0.0”

找到“server.cnf”，对于ubuntu,路径是“/etc/mysql/mariadb.conf.d/50-server.cnf”, centos的话在/etc下找到对应的即可。将“bind-address = 127.0.0.1”修改为“bind-address = 0.0.0.0”

# 修改mysql.user Host字段
database名：mysql

表名：user

root用户的Host字段默认是“localhost”，可以改为运行的ip段，使用“%”表示所有
```
use mysql;
update user set host="%" where user="root";
```

# 修改mysql.user plugin字段

plugin字段决定用户可以以什么方式登录,初次安装mysql后，root用户的plugin字段默认是`unix_socket`，所以就只能从本机通过unix_socket方式登录了，需要修改为`mysql_native_password`以支持从它机验证密码登录。

```
MariaDB [mysql]> select User, plugin from user;
+------+-------------+
| User | plugin      |
+------+-------------+
| root | unix_socket |
+------+-------------+
1 row in set (0.00 sec)

MariaDB [mysql]> update user set plugin = "mysql_native_password" where User = "root";
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [mysql]> select User, plugin from user;
+------+-----------------------+
| User | plugin                |
+------+-----------------------+
| root | mysql_native_password |
+------+-----------------------+
1 row in set (0.00 sec)

```

最后重启：`service mysql restart`

重启之后就可以登录了：

```
mysql -h [ip] -u root -p
# 输入密码
```

修改Host字段和修改plugin字段可以用一条sql同时执行，上面是分开说明而已。

但是一般不建议这样，会导致root用户的安全性降低。最好还是维持原样，让root用户只能在本机通过“unix_socket”，方式登录。 要远程登录的话就新建新的用户。

# 新建用戶

(@符号2边无空格)

```sql
-- 本地
CREATE USER "test"@"localhost" IDENTIFIED BY "test";
-- 远程
CREATE USER "test"@"%" IDENTIFIED BY "test";
```

- 为新建用于赋予权限

```sql
grant all privileges on testDB.* to 'test'@'localhost' identified by 'mypassword';
-- 刷新
flush privileges;
```