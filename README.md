# docker4gis

**本文为asdawn原创，转载请保留此说明**


## 目录

+ [Docker安装配置](#Docker安装配置)

+ [Docker私有仓库安装配置](#Docker私有仓库安装配置)

+ [Docker镜像制作方法](#Docker镜像制作方法)

+ [集成PostgreSQL数据库的Docker镜像](#集成PostgreSQL数据库的Docker镜像)

+ [常见问题](#常见问题)

## Docker安装配置

+ Windows版

Windows 10系统请安装最新版的Windows版[Docker Desktop](https://www.docker.com/products/docker-desktop)。依赖于WSL2（需要升级到较新的版本，然后安装WSL2补丁），Docker安装时会有相应的说明。

旧版本Windows可以使用基于虚拟机的Windows版Docker。

+ MacOS版

MacOS系统请安装最新版的[Docker Desktop](https://www.docker.com/products/docker-desktop)。

+ Linux版

不同的Linux发行版Docker引擎安装方式不同，可以参考[菜鸟教程](https://www.runoob.com/docker/ubuntu-docker-install.html)上的说明，必要时使用bing国际版用英文检索问题解决方法。

+ Docker配置方法

Docker Desktop提供基于图形界面的设置页面，Docker引擎参数设置在设置页面（settings）的Docker引擎（Docker Engine）选项卡中，里边的JSON就是配置参数。修改后会提示重启Docker以使变更生效。

Linux下的Docker配置文件为`/etc/docker/daemon.json`，可能需要手工创建。更新后要手动重启docker引擎使变化生效（不同系统下docker服务名称可能有轻微差异）：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Docker私有仓库安装配置

这里选择目前在持续维护中的官方提供的Docker Registry和第三方的Nexus3。假设已经装好了Docker。建议使用Linux系统的服务器运行Docker仓库。

### Docker Registry

Docker Registry是官方的镜像仓库工具，资源消耗很低，有ARM和X84、X64版的Docker镜像，能够在很多支持Docker的NAS上运行。

首先是下载镜像：

`docker pull registry`

然后找一个空间足够的分区创建dockerhub目录（**请根据实际情况修改路径**），例如:

`mkdir /mnt/hddb/dockerhub`

然后启动dockerhub服务（容器命名为dockerhub），数据目录使用/mnt/hddb/dockerhub，端口33321（将Docker Registry镜像内默认使用的5000端口映射到宿主机的33321端口）:

`docker run --name dockerhub -d -p 33321:5000 -v /mnt/hddb/dockerhub:/var/lib/registry registry`

防火墙开放对应端口:

```
firewall-cmd --permanent --add-port=33321/tcp
firewall-cmd --reload
```

接下来测试连接是否正常。查看名为dockerhub的容器的状态：

`docker logs dockerhub`

然后连接下网页版试试看（请替换为实际使用的IP地址）：

*IP地址*:33321/v2/_catalog，打开后显示repositories为空。

推一个镜像上去，过程为下载镜像、给镜像加本地空间的标签，然后上传到本地。不过没有启用https，所以要先在Docker的配置文件（Docker桌面找设置里的Docker Engine选项卡，Linux版Docker修改/etc/docker/daemon.json并重启docker），在其中的"insecure-registries"中添加该机器，例如：
```JSON
{
  ......
  "insecure-registries": [
    "IP地址:33321",
  ]
  ......
}

```

这样才能正常访问。接下来下载镜像并推送：

```
docker pull centos:8
docker tag centos:8 ip地址:33321/centos:8
docker push ip地址:33321/centos:8
```

现在可以查看一下本地目录：

`ls /mnt/hddb/dockerhub`

一级一级查看，可以看到已经在该文件夹下面创建了centos:8镜像的存储目录`/mnt/hddb/dockerhub/docker/registry/v2/repositories/centos/_manifests/tags/8`。
网页`IP地址:33321/v2/_catalog`的内容也发生了相应的改变。

如果需要关闭服务，执行：

`docker container stop dockerhub`

设置为自启动（只要Docker启动就运行registry），则docker run命令要添加一个`--restart=always`参数。启动命令改为：

`docker run --name dockerhub -d --restart=always -p 33321:5000 -v /mnt/hddb/dockerhub:/var/lib/registry registry`

Docker Registry可以通过htpasswd认证机制来进行账户限制，但是配置稍嫌复杂。需要账号及权限管理时建议使用Nexus3。

### Nexus3

Nexus3是目前尚在定期更新的最知名的仓库工具之一（portus已经两年不更新了），支持docker、git、maven、apt等多种仓库。同样是直接使用官方的Docker镜像进行部署。

首先下载镜像：

`docker pull sonatype/nexus3`

然后创建本地目录并修改访问权限（路径请根据需要自行修改，一般要找一个容量比较大的分区）：
```bash
mkdir /mnt/hddb/nexus3
chown -R 200 /mnt/hddb/nexus3
```

启动nexes3（容器名称指定为nexus3。nexus3的主控Web页面端口是8081，Docker服务我们设置为8082，然后映射到宿主机的33321和33322）：

`docker run -d -p 33321:8081 -p 33322:8082 --name nexus3 -v /mnt/hddb/nexus3:/nexus-data sonatype/nexus3`

如果需要始终运行（Docker服务正常运行的前提下）为：
`docker run -d --restart=always -p 33321:8081 -p 33322:8082 --name nexus3 -v/mnt/hddb/nexus3:/nexus-data sonatype/nexus3`

docker镜像使用的是8081（配置时要使用8082），为防止占用影响其他程序，启动docker时将他们映射到33321和33322。

nexus3启动时间较长，请耐心等待几分钟。如果不放心可以`docker logs nexus3`查看其日志，确定是否正常运行。

为了让其他机器能够访问，还要开放对应端口：

```bash
firewall-cmd --permanent --add-port=33321/tcp
firewall-cmd --permanent --add-port=33322/tcp
firewall-cmd --reload
```
 
停止运行的命令为：
`docker stop nexus3`

启动完毕后可以开始进行访问权限配置，Web管理页面的URL为*IP:33321*。管理员账号admin，密码在容器的`/nexus-data/admin.password`里，也就是`/mnt/hddb/nexus3/admin.password`，注意根据实际情况修改路径。查看密码：

`cat /mnt/hddb/nexus3/admin.password`

登录完成后就是安装向导，在里边更新密码、关闭匿名访问。

接下来创建Docker仓库。可以针对不同的项目或群组创建多个。建议清理掉系统预装的其他仓库。设置里新建仓库，有三种类型：

项目	| 具体说明
----- | ------
hosted	| 本地存储。像官方仓库一样提供本地私库功能
proxy	| 提供代理其它仓库的类型
group	| 组类型，能够组合多个仓库为一个地址提供服务

局域网私有库就选hosted，然后设置一个http端口（docker引擎的配置里要添加相应的非可信源），这里假设是8082。如果申请了证书还可以启用https端口。

权限管理通过（仓库访问）权限、角色和用户实现。

+ nexus会将各个仓库的权限拆分成细节选项，一般可以直接使用，有必要再自行编辑。

+ 角色是权限的组合，可以对用户进行分类，设置多个角色分开管理。

+ 用户具有账号密码和角色，登陆后可以享有角色对应的访问权限。

一般还要关闭匿名访问，强制要求登录。到这里就完成了初步的配置，可以开始测试。

连接下网页版试试看（请替换为实际使用的IP地址）：

*IP地址*:33321/v2/_catalog，打开后显示repositories为空。

推一个镜像上去，过程为下载镜像、给镜像加本地空间的标签，然后上传到本地。不过没有启用https，所以要先在Docker的配置文件（Docker桌面找设置里的Docker Engine选项卡，Linux版Docker修改/etc/docker/daemon.json并重启docker），在其中的"insecure-registries"中添加该机器，例如：
```JSON
{
  ......
  "insecure-registries": [
    "IP地址:33322",
  ]
  ......
}

```

这样才能正常访问。接下来下载镜像并推送：

```
docker pull centos:8
docker tag centos:8 ip地址:33322/centos:8
docker push ip地址:33322/centos:8
```

然后在Nexus3的Web管理页面中可以查看有哪些镜像并进行管理。当然也可以使用Docker仓库的http接口。


## Docker镜像制作方法

### 入门级做法

pull一个Docker镜像，挂载执行其中的bash（建议docker run带上 --name=名称），然后手动安装配置所需的软件与应用，执行完毕后退出。然后对该容器进行commit，得到新的镜像。

`docker commit 容器id或者name 新镜像的标签`

这种方式相对简单，适合在实验阶段使用，但基础镜像是什么、进行了哪些操作不透明，对于用户来说具有一定的风险（例如镜像被嵌入木马或病毒）。

### 正规做法

一般是通过Dockerfile来编译镜像的，Dockerfile由以下几个部分组成：
 
#### a. 注释
 
 `#`开头的行是注释行
 
### b. 基础镜像（必选）
 
 `FROM 镜像标签`
 
#### c. 镜像的Label
 
 `LABEL 标签名="标签值"`

比较新的Dockerfile规范中很多信息统一采用LABEL来添加，详见[官方文档](https://docs.docker.com/engine/reference/builder/)

#### d. 环境变量设置

`ENV 环境变量 取值`

·ARG 环境变量 取值·

前者设置的是镜像加载后的环境变量，后者是设置镜像编译过程中的环境变量。

#### e. 暴露的端口

`EXPOSE 端口号`

有了ENV和EXPOSE，Docker容器管理工具就能自动识别需要设置哪些变量、重定向哪些端口，有利于自动化执行。

#### f. 执行命令

`RUN 操作内容`

这里操作内容就是普通的bash命令。每条RUN提交一层镜像保存其修改，如果没有特殊需求，可以将多个命令用 ` && `拼接到一行以减少非必要的镜像层以节约资源。此外对于一些有前后依赖的命令采用这种方式也很有用处，例如启动数据库、导入数据、关闭数据库这个操作序列，如果写在多个RUN里就会出错，用 && 连接执行就正常了。

由于基础镜像都相对精简，一些常见软件包可能尚未安装（尤其是高度精简的Alpine Linux），所以必要时需要先执行安装操作。可以先以交互的形式运行一次试试看。

由于使用的基础镜像是什么、执行了哪些操作都可以直接看到，部署的应用及其执行方式也比较明确。很多面向一般公众的应用选择将代码发布到github上，从而保证其Docker镜像只使用官方基础镜像、官方软件包以及源代码。

#### g. 拷贝文件到镜像内

`COPY 外部路径 镜像内路径`

#### h. 自启动

`ENTRYPOINT 命令或脚本`

镜像每次启动时都会执行该命令或脚本。只允许一个，所以必要时请写一个脚本。

更多关于Dockerfile格式的说明请参见中文的[菜鸟教程](https://www.runoob.com/docker/docker-dockerfile.html)；比较新的细节还是要参阅[官方文档](https://docs.docker.com/engine/reference/builder/)

一般来说，除非依赖于本地文件，有了Dockerfile文件就可以编译出对应的镜像。因此很多开源软件选择github存放源代码，私有代码也可以使用私有的git仓库，这样就可以避免使用本地文件。再次强调，本地文件存在被恶意使用的可能性，风险大于直接使用源代码。对于敏感度极高的应用，请谨慎处理。

`docker build -t 新镜像标签 Dockerfile所在目录`

Dockerfile文件名要求是**Dockerfile**。

## 集成PostgreSQL数据库的Docker镜像

+ postgres

+ PostGIS

+ PgRouting

+ 自定义镜像


`pg_ctl -D 


/usr/lib/postgresql/11/bin/pg_ctl -D /postgres -l /postgres/logfile stop

## 常见问题

+ 下载慢怎么办

不论是Linux发行版还是Docker官方仓库都有国内镜像，设为国内镜像后下载速度通常会比较快。例如在Docker配置文件（具体位置请参见私有库搭建部分的说明）的"registry-mirrors"项目中添加163的源"https://hub-mirror.c.163.com"：
```JSON
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com"
  ]
 ......
 }
 ```

由于Docker官方仓库提供免费的不限量的上传服务，国内镜像仓库的同步机制是缓存被申请过的镜像。因此对于比较新或者比较小众的镜像，启用国内源时第一次下载可能也是很慢，但是之后就快了。

Linux软件包比较标准化，一般不存在这类问题。不同的发行版的国内镜像仓库设置方法不同，可参阅[清华大学国内源](https://mirrors.tuna.tsinghua.edu.cn/help/)对应项目的说明。对于Ubuntu和Alpine，可以从官方仓库pull设置好国内源的asdawn/cnubuntu:20.10和asdawn/cnalpine:3.12，或者自行编译，Dockerfile见对应的文件夹。

+ 怎样查找已有的Docker镜像

[Docker官方仓库](https://hub.docker.com/)提供查询用的Web界面；命令行下也可以使用`docker search 关键字`来进行检索。

私有仓库需查询镜像列表就不能通过`docker search`来进行了，需要通过仓库标准的http接口来进行查询（所有Docker仓库工具都要实现这些接口）。例如：

查看私有仓库有哪些镜像的URL是`私有仓库URL/v2/_catalog`，查看指定镜像有哪些版本是`私有仓库URL/v2/镜像名/tags/list`。这些URL可以直接通过浏览器打开（可能需要登录），也可以通过命令行的`curl`命令打开，例如查询192.168.3.33:33333上的app1有哪些版本的命令为：

`curl -u 账号:密码 -XGET http://192.168.3.33:33333/v2/app1/tags/list`

Nexus3提供了基于浏览器的镜像浏览页面和镜像管理页面，比直接使用标准接口更加友好一些。
