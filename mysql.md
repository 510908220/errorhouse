# 记录mysql使用过程的一些错误

## ERROR 2003 (HY000): Can't connect to MySQL server on '192.168.56.200' (111)
连接mysql服务器报的错误. 首先查看mysql配置文件(my.conf). 有如下配置:
```
# Instead of skip-networking the default is now to listen only on
# localhost which is more compatible and is not less secure.
bind-address            = 127.0.0.1
```
可以看这只允许本地访问mysql数据库. 为了使其他机器可以连接需要注释掉```bind-address ```,保存并重启服务.

但是接着又遇到了这错误:
```
ERROR 1045 (28000): Access denied for user 'root'@'192.168.56.100' (using password: YES)
```
在mysql服务器执行```SELECT host FROM mysql.user WHERE User = 'root';```输出:
```
+-----------+
| host      |
+-----------+
| 127.0.0.1 |
| ::1       |
| db        |
| localhost |
+-----------+
```
因为host并没有我们发起连接的主机ip,所以连接不了. 如下方式可以使连接的ip访问:
```
CREATE USER 'root'@'ip_address' IDENTIFIED BY 'some_pass';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'ip_address';
```
这里的意思即使允许ip_address使用root账号,some_pass密码登陆mysql.如果机器比较多的话每次添加都很麻烦,这时可以使用如下方式:
```
CREATE USER 'root'@'%' IDENTIFIED BY 'some_pass';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
```
这允许任何host以root去登陆到mysql了.
最后需要执行```FLUSH PRIVILEGES;```使上述操作生效.


参考:
- [cant-connect-to-mysql-server-error-111](http://stackoverflow.com/questions/1420839/cant-connect-to-mysql-server-error-111)
- [error-1130-hy000-host-is-not-allowed-to-connect-to-this-mysql-server](http://stackoverflow.com/questions/19101243/error-1130-hy000-host-is-not-allowed-to-connect-to-this-mysql-server)
