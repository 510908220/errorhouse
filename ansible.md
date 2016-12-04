# 记录Ansible使用的一些错误


## SSH Error: muxserver_listen bind(): Operation not permitted while connecting to xx.xx.xx.xx
ansible 1.9

**问题产生**:
我是使用了一个```python manage.py ecs```，里面会调用```ansible api```. ```ecs```这个命令是使用```supervisor```管理的. 如图:

```
[program:ecs]
command=python manage.py ecs
directory=/vagrant_data/pamc_monitor
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
startretries=3
stderr_logfile_maxbytes=1MB
stdout_logfile=/var/log/ecs.log
stdout_logfile_maxbytes=1MB
user=root
redirect_stderr=true
```

当我在```supervisorctl```里启动```ecs```的话,最后```ansible api```执行结果会报错:
```
PLAY [all] ********************************************************************

TASK: [Execute Script] ********************************************************
fatal: [192.168.56.103] => SSH Error: muxserver_listen bind(): Operation not permitted
    while connecting to 192.168.56.103:22
It is sometimes useful to re-run the command using -vvvv, which prints SSH debug output to help diagnose the issue.
```
但是我手动运行```python manage.py ecs```却是好的.

**问题追踪**:
可以在命令行上调用playbook的yaml文件,加上```-vvvv```参数,比如我这里命令是:
```
ansible-playbook -i /vagrant_data/pamc_monitor/ansible_cloud/ecs/hosts.py  /vagrant_data/pamc_monitor/ansible_cloud/ecs.yml  -vvvv
```
可以看到如下信息:
```
<192.168.56.103> EXEC ssh -C -tt -vvv -o ControlMaster=auto -o ControlPersist=60s -o ControlPath="/vagrant_data/pamc_monitor/$HOME/.ansible/cp/ansible-ssh-%h-%p-%r" -o StrictHostKeyChecking=no -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o ConnectTimeout=10 192.168.56.103 /bin/sh -c 'mkdir -p $HOME/.ansible/tmp/ansible-tmp-1480661526.44-28318449944065 && echo $HOME/.ansible/tmp/ansible-tmp-1480661526.44-28318449944065'
fatal: [192.168.56.103] => SSH Error: muxserver_listen bind(): Operation not permitted
    while connecting to 192.168.56.103:22
It is sometimes useful to re-run the command using -vvvv, which prints SSH debug output to help diagnose the issue.
```

`ControlPath`这个路径是`/vagrant_data/pamc_monitor/$HOME/.ansible/cp/ansible-ssh-%h-%p-%r`, 这个路径最后ssh会识别为`/vagrant_data/pamc_monitor//root/.ansible/cp/ansible-ssh-xxxx`看到"//"了吗，这路径肯定不存在. 错误原因大概就是这个. 但是为什么会这样了?

`ControlPath`这个路径是由`ansible`下的`ssh`模块设置的. 看代码可以知道默认值是`$HOME/.ansible/cp`, 大概代码:

```python
# 文件:ansible/constants.py
ANSIBLE_SSH_CONTROL_PATH       = get_config(p, 'ssh_connection', 'control_path', 'ANSIBLE_SSH_CONTROL_PATH', "%(directory)s/ansible-ssh-%%h-%%p-%%r")

# 文件:ansible/utils/__init__.py
def prepare_writeable_dir(tree,mode=0777):
    ''' make sure a directory exists and is writeable '''

# 文件/ansible/runner/connection_plugins/ssh.py
self.cp_dir = utils.prepare_writeable_dir('$HOME/.ansible/cp',mode=0700)

if cp_in_use and not cp_path_set:
            self.common_args += ["-o", "ControlPath=\"%s\"" % (C.ANSIBLE_SSH_CONTROL_PATH % dict(directory=self.cp_dir))]
```
其中```prepare_writeable_dir```会调用```os.path.realpath```,简单描述一下效果(假如当前在`/root/.ansible/cp`目录):
```python
>>> p = '$HOME/.ansible/cp'
>>> os.path.realpath(p)
'/root/.ansible/cp/$HOME/.ansible/cp'
```
然后bash里最后识别为:
```
root@master:~# echo "/root/.ansible/cp/$HOME/.ansible/cp"
/root/.ansible/cp//root/.ansible/cp
```
