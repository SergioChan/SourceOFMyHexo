title: Ubuntu下重置MySQL的root密码
date: 2014-12-12 10:10:45
categories: Linux服务器笔记
author: Sergio Chan
tags: [Ubuntu, MySQL]
---

sudo vi /etc/mysql/my.cnf，在[mysqld]段中加入一行“skip-grant-tables”
具体环境中可能my.cnf已经存在且在其他目录，记得要找到有效的my.cnf配置文件路径

sudo service mysql restart，重启mySQL服务
具体参考实际的重启MySQL的命令

sudo mysql -u root -p mysql，用空密码进入mysql管理命令行

(进入mysql,或者用use mysql指令)

update user set password=PASSWORD("123") where user='root';，把密码重置为123

(注意，如果是表中没有的用户名，使用insert)

quit，退出数据库管理

sudo vim /etc/mysql/my.cnf，把刚才加入的那一行“skip-grant-tables”注释或删除

sudo service mysql restart，OK，搞定！