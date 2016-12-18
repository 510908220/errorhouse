# Docker使用

## 安装

docker安装时对系统有要求的:

- Docker only works on a 64-bit Linux installation
- Docker requires version 3.10 or higher of the Linux kernel.
  如果内核版本不符合,可以看官网步骤升级内核

ubuntu的安装步骤,见官网[install-the-latest-version](https://docs.docker.com/engine/installation/linux/ubuntulinux/#/install-the-latest-version)

我用的是ubuntu 12.04,按照官网步骤还是出错:
W: Failed to fetch https://apt.dockerproject.org/repo/dists/ubuntu-precise/main/binary-amd64/Packages  server certificate verification failed. CAfile: /etc/ssl/certs/ca-certificates.crt CRLfile: none
还以为是版本问题,最后换成16.04还一个样，但是错误依旧，找了半天也没合适的方法. 最后看到一个可行的答案[using-a-self-signed-ssl-cert-for-an-https-based-internal-apt-repository](http://serverfault.com/questions/340887/using-a-self-signed-ssl-cert-for-an-https-based-internal-apt-repository):
在目录```/etc/apt/apt.conf.d```随便建一个文件,内容为:
```bash
Acquire::https {
        Verify-Peer "false";
        Verify-Host "false";
}
```
这样在执行```sudo apt-get update```就不会出错,就可以安装docker了

说明:我是用的```vagrant box```镜像，不知道是不是这个原因.

当我执行```docker pull debian```时, 又遇到了这个错误```x509: certificate signed by unknown authority```,这个问题大概是个证书问题,解决方式如下(Ubuntu/Debian),[答案](https://github.com/docker/docker/issues/8849):
- 下载证书: https://curl.haxx.se/docs/caextract.html
- 拷贝到目录: ```/usr/local/share/ca-certificates```
- ```sudo update-ca-certificates```,要是不起作用的话加上```-f```选项
- ```sudo service docker restart```

## 使用

#### docker images
查看系统当前有哪些镜像,刚开始为空:
```bash
root@hzz:/home/vagrant# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```

#### [docker search](https://docs.docker.com/engine/reference/commandline/search/)
参数信息:
```bash
Usage:  docker search [OPTIONS] TERM

Search the Docker Hub for images

Options:
  -f, --filter value   Filter output based on conditions provided (default [])
                       - is-automated=(true|false)
                       - is-official=(true|false)
                       - stars=<number> - image has at least 'number' stars
      --help           Print usage
      --limit int      Max number of search results (default 25)
      --no-trunc       Don't truncate output
```
简单描述一下参数的含义:
- ```--filter```:过滤搜索结果, 条件格式为```key=value```形式，多个条件的话并列写(--filter "foo=bar" --filter "bif=baz"). 比如要搜索不少于500颗星的ubuntu镜像,可以这样写查询:
```bash
root@hzz:/home/vagrant# docker search --filter stars=500  ubuntu 
NAME      DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
ubuntu    Ubuntu is a Debian-based Linux operating s...   5218      [OK]  
```

另外还支持两个条件```is-automated```(是否是自动构建)、```is-official```(是否是官网)

- ```--limit```:控制显示多少条结果
- ```--no-trunc```: 是否显示完整的镜像描述

#### [docker pull](https://docs.docker.com/engine/reference/commandline/pull/)
参数信息:

```bash
Usage:  docker pull [OPTIONS] NAME[:TAG|@DIGEST]

Pull an image or a repository from a registry

Options:
  -a, --all-tags                Download all tagged images in the repository
      --disable-content-trust   Skip image verification (default true)
      --help                    Print usage
```
看几组例子:
1. 下载指定名称的镜像. 如果没指定```tag```,默认会使用```:latest```. 就是说```docker pull ubuntu```和```docker pull ubuntu:latest```是一样的:

   ```bash
   root@hzz:/usr/local/share/ca-certificates# docker pull ubuntu
   Using default tag: latest
   latest: Pulling from library/ubuntu
   b3e1c725a85f: Pull complete 
   4daad8bdde31: Pull complete 
   63fe8c0068a8: Pull complete 
   4a70713c436f: Pull complete 
   bd842a2105a8: Pull complete 
   Digest: sha256:7a64bc9c8843b0a8c8b8a7e4715b7615e4e1b0d8ca3c7e7a76ec8250899c397a
   Status: Downloaded newer image for ubuntu:latest
   ```

   docker镜像是由多层组成的, 上面的例子，镜像包含5层. 层是可以被复用的. 从https://hub.docker.com/_/ubuntu/可以看到```ubuntu```镜像的一些tag, 这些都是16.x版本的:`16.04`, `xenial-20161213`, `xenial`, `latest`,当我安装`xenial`这个镜像时:

   ```bash
   root@hzz:/usr/local/share/ca-certificates# docker pull ubuntu:xenial
   xenial: Pulling from library/ubuntu
   Digest: sha256:7a64bc9c8843b0a8c8b8a7e4715b7615e4e1b0d8ca3c7e7a76ec8250899c397a
   Status: Downloaded newer image for ubuntu:xenial
   ```

   可以看到并未下载,因为```xenial```与```latest```共享所有的层，而```latest```之前已经下载好了. 查看一下镜像:

   ```bash
   root@hzz:/usr/local/share/ca-certificates# docker images
   REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
   ubuntu              latest              104bec311bcd        38 hours ago        129 MB
   ubuntu              xenial              104bec311bcd        38 hours ago        129 MB
   ```

   可以看到```xenial```与```latest```镜像id是一样的,只是名字不同而已.

2. 根据digest 下载镜像.

   通常我们下载镜像的时候,都是最新的. 比如```docker pull ubuntu:14.04```会下载最新的```Ubuntu 14.04 ```

   (PS:我理解是不是同一个tag可以多次提交,这样导致有个提交历史),上面的例子Digest是`sha256:7a64bc9c8843b0a8c8b8a7e4715b7615e4e1b0d8ca3c7e7a76ec8250899c397a`

   下面使用Digest来下载一个镜像:

   ```docker pull ubuntu@sha256:45b23dee08af5e43a7fea6c4cf9c25ccf269ee113168c19722f87876677c5cb2
   docker pull ubuntu@sha256:7a64bc9c8843b0a8c8b8a7e4715b7615e4e1b0d8ca3c7e7a76ec8250899c397a
   ```

   输出:

   ```bash
   root@hzz:/usr/local/share/ca-certificates# docker pull ubuntu@sha256:7a64bc9c8843b0a8c8b8a7e4715b7615e4e1b0d8ca3c7e7a76ec8250899c397a
   sha256:7a64bc9c8843b0a8c8b8a7e4715b7615e4e1b0d8ca3c7e7a76ec8250899c397a: Pulling from library/ubuntu
   Digest: sha256:7a64bc9c8843b0a8c8b8a7e4715b7615e4e1b0d8ca3c7e7a76ec8250899c397a
   Status: Image is up to date for ubuntu@sha256:7a64bc9c8843b0a8c8b8a7e4715b7615e4e1b0d8ca3c7e7a76ec8250899c397a
   ```

   另外Digest也可以使用在Dockerfile里,例如:

   ```bash
   FROM ubuntu@sha256:45b23dee08af5e43a7fea6c4cf9c25ccf269ee113168c19722f87876677c5cb2
   MAINTAINER some maintainer <maintainer@example.com>
   ```

   ​

3. 从指定的镜像服务器下载.

   默认```docker pull```是从[Docker Hub](https://hub.docker.com/)下载镜像的. 当然可以自己指定镜像服务器. 比如下面命令会从```myregistry.local:5000```这个镜像服务器下载```testing/test-image```镜像的.

   ```bash
   docker pull myregistry.local:5000/testing/test-image
   ```

4. 从镜像仓库下载多个镜像

   默认```docker pull```只下载一个镜像. 一个镜像仓库可以包含多个镜像的. 为了下载镜像仓库里所有的镜像,可以使用`-a` (or `--all-tags`) 选项.下面的命令会下载ubuntu仓库里所有镜像:

   ​
#### [docker run](https://docs.docker.com/engine/reference/commandline/run/)
参数信息:
```
Usage:  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

Run a command in a new container

Options:
```
选项内容太多就不列出了. 字面意思看就是在一个新的容器运行一个命令. 下面为官网描述:
`docker run`首先在镜像的基础上创建一个可写的层. 然后使用特定的命令启动它. 也就是说`docker run`等价于下面的API:`/containers/create`和`/containers/(id)/start`
下面看一些例子:
1. 基本用法
   使用命令`docker run --name test -it ubuntu`可以创建一个名为`test`的容器,参数`-it`告诉docker给这个容器分配一个伪终端到stdin和创建一个交互的shell. 如果再次运行这个命令会报错，因为容器名是唯一的.
2. 捕获容器id(–cidfile)
   执行命令`docker run --cidfile /tmp/docker_test.cid ubuntu echo "test"`
   这个命令会创建一个容器并在控制台输出`test`. `cidfile`标记使得docker将容器id写入指定的文件里. 如果这个文件存在,那么命令指定会失败的. docker会在`docker run`退出时关闭文件.
3. 挂载目录
   ```docker run -v /doesnt/exist:/foo -w /foo -i -t ubuntu bash```
   上面的命令挂在本地目录`/doesnt/exist`(目录不存在时会自动创建)到容器目录`/foo`,然后切换到容器目录`/foo`
   默认挂载的路径权限为读写。如果指定为只读可以用：ro
```
docker run -it -v /home/dock/Downloads:/usr/Downloads:ro --name test ubuntu /bin/bash
```
4. 根据容器挂载
   `上面的例子创建了一个挂载信息,当我新创建了一个容器也需要相同的挂载方式,可以使用如下命令:
```
docker run -it --volumes-from test ubuntu /bin/bash
```
这样新的容器也和主机目录共享了. 可以在容器名后加上```:ro or :rw```来控制容器内目录读写权限.
5. 绑定端口到指定接口
   ```docker run -p 127.0.0.1:80:8080 -it ubuntu bash```
   上面命令将容器内的8080端口进行了绑定到主机的80端口.比如我在容器内启动一个简单的web服务```python -m SimpleHTTPServer 8080```,在宿主机打开浏览器输入ip就能看到容器内的文件列表了. 

- 不指定主机端口,那么会随机分配一个端口,例如`docker run -p 127.0.0.1::8080 -it ubuntu bash`;
- 不指定ip,那么则为`0.0.0.0`.
- 不指定主机ip和端口,那么会将容器内的8080端口绑定到主机`0.0.0.0:随机端口上`
默认都是绑定的TCP

docker网络详细可以[查看](http://www.open-open.com/lib/view/open1423703640748.html#articleHeader6)



## Dockerfile
https://docs.docker.com/engine/reference/builder/#/maintainer
http://www.open-open.com/lib/view/open1423703640748.html#articleHeader26
