# 记录一些在bash上的问题
## -bash: /usr/bin/supervisorctl: No such file or directory
> 问题是这样的,一开始我使用apt装了supervisor,后来使用pip升级了一下,新的路径变为了```/usr/local/bin/supervisorctl```. 但是我在终端
执行```supervisorctl```就出现了那个错误了. 使用的路径是一个旧的,应该是在哪缓存的了.

bash 会对命令以及路径进行缓存. 可以使用```type```查看相应的信息:
```
root@master:/vagrant_data/pamc_monitor# type supervisorctl
supervisorctl is hashed (/usr/bin/supervisorctl)
```
可以通过如下命令清除整个缓存:
```
hash -r
```
或者只清理某个命令的缓存:
```
hash -d supervisorctl
```
看下效果:
```
root@master:/vagrant_data/pamc_monitor# hash -r supervisorctl
root@master:/vagrant_data/pamc_monitor# type supervisorctl
supervisorctl is hashed (/usr/local/bin/supervisorctl)
```

参考:
- [how-do-i-clear-bashs-cache-of-paths-to-executables](http://unix.stackexchange.com/questions/5609/how-do-i-clear-bashs-cache-of-paths-to-executables)
