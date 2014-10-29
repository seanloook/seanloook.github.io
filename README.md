seanloook.github.io
===================

My Second Github Blog
本文只记录大部分的使用场景，只做备忘。
参考
http://docker-doc.readthedocs.org/zh_CN/latest/reference/commandline/cli.html


### 1. 列出机器上的镜像 ###
```
# docker images 
REPOSITORY                   TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ubuntu                       14.10               2185fd50e2ca        13 days ago         236.9 MB
…
```
其中我们可以根据REPOSITORY来判断这个镜像是来自哪个服务器，如果没有 / 则表示官方镜像，类似于`username/repos_name`表示Github的个人公共库，类似于`regsistory.example.com:5000/repos_name`则表示的是私服。
IMAGE ID列其实是缩写，要显示完整则带上`--no-trunc`选项

### 2. 在docker index中搜索image ###
`Usage: docker search TERM `
```
# docker search seanlo
NAME                DESCRIPTION           STARS     OFFICIAL   AUTOMATED
seanloook/centos6   sean's docker repos         0
```
搜索的范围是官方镜像和所有个人公共镜像。NAME列的 / 后面是仓库的名字。

### 3. 从docker registry server 中下拉image或repository ###
`Usage: docker pull [OPTIONS] NAME[:TAG]`
```
# docker pull centos
```
上面的命令需要注意，在docker v1.2版本以前，会下载官方镜像的centos仓库里的所有镜像，而从v.13开始官方文档里的说明变了：will pull the centos:latest image, its intermediate layers and any aliases  of the same id，也就是只会下载tag为latest的镜像（以及同一images id的其他tag）。
也可以明确指定具体的镜像：
```
# docker pull centos:centos6
```
当然也可以从某个人的公共仓库（包括自己是私人仓库）拉取，形如`docker pull username/repository<:tag_name>` ：
```
# docker pull seanlook/centos:centos6
```
如果你没有网络，或者从其他私服获取镜像，形如`docker pull registry.domain.com:5000/resps<tag_name>` ：
```
# docker pull dl.dockerpool.com:5000/mongo:latest
```
### 4. 推送一个image或repository到registry ###
与上面的pull对应，可以推送到Docker Hub的Public、Private以及私服，但不能推送到Top Level Repository。
```
# docker push seanlook/mongo
# docker push registry.tp-link.net:5000/mongo:2014-10-27
```
registry.tp-link.net也可以写成IP，172.29.88.222。
在repository不存在的情况下，命令行下push上去的会为我们创建为私有库，然而通过浏览器创建的默认为公共库。

### 5. 从image启动一个container ###
`docker run`命令首先会从特定的image创之上create一层可写的container，然后通过start命令来启动它。停止的container可以重新启动并保留原来的修改。run命令启动参数有很多，以下是一些常规使用说明，更多部分请参考http://www.cnphp6.com/archives/24899
当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：
- 检查本地是否存在指定的镜像，不存在就从公有仓库下载
- 利用镜像创建并启动一个容器
- 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
- 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
- 从地址池配置一个 ip 地址给容器
- 执行用户指定的应用程序
- 执行完毕后容器被终止

`Usage: docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`

#### 5.1 使用image创建container并执行相应命令，然后停止 ####
```
# docker run ubuntu echo "hello world"
hello word
```
这是最简单的方式，跟在本地直接执行`echo 'hello world'` 几乎感觉不出任何区别，而实际上它会从本地ubuntu:latest镜像启动到一个容器，并执行打印命令后退出（`docker ps -l`可查看）。需要注意的是，默认有一个`--rm=true`参数，即完成操作后停止容器并从文件系统移除。因为Docker的容器实在太轻量级了，很多时候用户都是随时删除和新创建容器。
容器启动后会自动随机生成一个`CONTAINER ID`，这个ID在后面commit命令后可以变为`IMAGE ID`
#### 使用image创建container并进入交互模式, login shell是/bin/bash ####
```
# docker run -i -t --name mytest centos:centos6 /bin/bash
bash-4.1#
```
上面的`--name`参数可以指定启动后的容器名字，如果不指定则docker会帮我们取一个名字。镜像`centos:centos6`也可以用`IMAGE ID` (68edf809afe7) 代替），并且会启动一个伪终端，但通过ps或top命令我们却只能看到一两个进程，因为容器的核心是所执行的应用程序，所需要的资源都是应用程序运行所必需的，除此之外，并没有其它的资源，可见Docker对资源的利用率极高。此时使用exit或Ctrl+D退出后，这个容器也就消失了（消失后的容器并没有完全删除？）
（那么多个TAG不同而IMAGE ID相同的的镜像究竟会运行以哪一个TAG启动呢

#### 5.2 运行出一个container放到后台运行 ####
```
# docker run -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 2; done"
ae60c4b642058fefcc61ada85a610914bed9f5df0e2aa147100eab85cea785dc
```
它将直接把启动的container挂起放在后台运行（这才叫saas），并且会输出一个`CONTAINER ID`，通过`docker ps`可以看到这个容器的信息，可在container外面查看它的输出`docker logs ae60c4b64205`，也可以通过`docker attach ae60c4b64205`连接到这个正在运行的终端，此时在`Ctrl+C`退出container就消失了，或`docker stop $CONTAINER_ID`终止。
另外，如果-d启动但后面的命令执行完就结束了，如`/bin/bash`、`echo test`，则container做完该做的时候依然会终止。而且-d不能与--rm同时使用
可以通过这种方式来运行memcached、apache等。

#### 5.3 将container的端口映射到宿主机的端口 ####
`# docker run -i -t -p <host_port:contain_port>`


 
从本地注册服务器下拉
