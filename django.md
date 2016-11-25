# 记录django的一些错误

## Error loading MySQLdb module: No module named MySQLdb
这是由于没装```MySQL-python```包. 使用```pip install MySQL-python```来安装.
这一可能会遇到如下错误:
```
EnvironmentError: mysql_config not found
```
这时需要执行如下命令:
```
apt-get install python-mysqldb
```
