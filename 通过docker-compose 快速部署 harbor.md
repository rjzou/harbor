[TOC]
通过docker-compose 快速部署 harbor



# 一、概述
Harbor是一个开源的企业级Docker Registry管理工具，它提供了一个安全、可靠、可扩展的平台，用于存储、管理和分发Docker镜像。Harbor可以帮助组织和团队更好地管理Docker镜像，并提高应用程序构建和部署的效率。
以下是Harbor的一些主要特点：


安全：Harbor提供了完整的认证和授权机制，支持LDAP、AD等集成方式，可以让用户更加安全地管理和使用Docker镜像。


可靠：Harbor提供了多个镜像仓库，支持复制和高可用性，确保应用程序的部署和升级是平滑和无缝的。


可扩展：Harbor是一个可扩展的平台，可以支持数千个并发构建和部署，从而满足高流量的应用程序部署需求。


用户友好：Harbor提供了一个直观、易于使用的Web界面，让用户可以轻松地查找、上传和下载Docker镜像。


灵活性：Harbor提供了一个可定制的平台，可以根据组织和团队的需求进行自定义配置。


Harbor的部署和使用非常简单，可以使用Docker Compose轻松地在本地环境中部署。Harbor还提供了API和CLI工具，可以方便地与其他DevOps工具集成。Harbor已被广泛采用，并被认为是企业级Docker Registry管理工具的首选之一。
# 二、Harbor 架构

如上图所描述，Harbor由6个大的模块所组成：


Proxy: Harbor的registry、UI、token services等组件，都处在一个反向代理后边。该代理将来自浏览器、docker clients的请求转发到后端服务上。


Registry: 负责存储Docker镜像，以及处理Docker push/pull请求。因为Harbor强制要求对镜像的访问做权限控制， 在每一次push/pull请求时，Registry会强制要求客户端从token service那里获得一个有效的token。


Core services: Harbor的核心功能，主要包括如下3个服务:


UI：图形界面


WebHook：及时获取registry上image状态变化情况，在registry上配置 webhook，把状态变化传递给UI模块。


Token服务：负责根据用户权限给每个docker push/pull命令签发token。Docker 客户端向Registry服务发起的请求,如果不包含token，会被重定向到这里，获得token后再重新向Registry进行请求。


Database：为core services提供数据库服务，负责储存用户权限、审计日志、Docker image分组信息等数据。


Job services: 主要用于镜像复制，本地镜像可以被同步到远程Harbor实例上。


Log collector: 负责收集其他组件的日志到一个地方。

# 第一种方法

# 三、前期准备
## 1）部署 docker
# 安装yum-config-manager配置工具
```
yum -y install yum-utils

```

## 建议使用阿里云yum源：（推荐）
```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
安装docker-ce版本

```
yum install -y docker-ce

```

### 启动并开机启动
```
systemctl enable --now docker

docker --version

```



## 2）部署 docker-compose

```
curl -SL https://github.com/docker/compose/releases/download/v2.16.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
```
```
chmod +x /usr/local/bin/docker-compose
docker-compose --version
```


# 四、开始部署 Harbor
GitHub地址：[github.com/goharbor/ha…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fgoharbor%2Fharbor%2Freleases%2F)

## 1）下载Harbor的Docker Compose文件
在GitHub上获取Harbor的Docker Compose文件，其中包含了必需的服务和配置文件。您可以使用以下命令在终端中进行下载：
```

export HARBOR_VERSION=2.5.6
wget https://github.com/goharbor/harbor/releases/download/v${HARBOR_VERSION}/harbor-offline-installer-v${HARBOR_VERSION}.tgz
tar xvf harbor-offline-installer-v${HARBOR_VERSION}.tgz
cd harbor
```

## 2）修改配置
1、hostname改为本机hostname或者本机IP
2、把http的端口改为自己想要映射的端口，
3、去掉https

```
cp harbor.yml.tmpl harbor.yml
```
```
# vim harbor.yml
hostname: {自己服务器的ip 内网外网都可以}

# http related config
http:
# port for http, default is 80. If htps enabled, this port will redirect to htps port
port: {自定义端口}

# https related config
#https:
  # https port for harbor, default is 443
#  port: 443
  # The path of cert and key files for nginx
#  certificate: /your/certificate/path
#  private_key: /your/private/key/path
```

## 3）开始安装
下载镜像，生成配置文件（可选）
```
bash prepare
```
通过上面的命令生成的配置文件就可以通过 docker-compose up -d 启动服务
开始安装，包含了生成配置、下载镜像、和启动服务
```
bash install.sh
```

## 4）安装完成后会在当前目录自动生成docker-compose.yml文件
```
docker-compose ps
```

```
# 再次安装，就可以执行以下命令
docker-compose up -d
```

```
# 或者执行下面这句
docker-compose up -f docker-compose.yml -d
```
```
# 停止
docker-compose down
```

访问：http://ip:port
账号/密码：admin/Harbor12345

## 5）客户端docker配置私有镜像仓库
1、配置
修改/etc/docker/daemon.json文件，添加下面一行:
bash复制代码# "insecure-registries":["ip:port"],
# 示例配置如下：
```
{
  "registry-mirrors": ["https://hccwwfjl.mirror.aliyuncs.com"],
  "insecure-registries": ["local-168-182-110:80"]
}
```

重启容器
```
systemctl daemon-reload
systemctl restart docker
```
2、推送和拉取镜像常用操作
1、先登录harbor
```
docker login ip:port -u xxx -p xxx
```

```
docker login 192.168.182.110:80 -u admin
Harbor12345
```

2、会提示登录成功，然后进行tag
docker tag image_id(本地需要push的镜像) ip:port/项目名/保存的镜像名(例如：xxx:version1)
```
docker tag goharbor/harbor-exporter:v2.5.6 local-168-182-110:80/library/goharbor/harbor-exporter:v2.5.6
```
3、推送
docker push ip:port/项目名/保存的镜像名(例如：xxx:version1)
```
docker push local-168-182-110:80/library/goharbor/harbor-exporter:v2.5.6
```
这样就可以在harbor上看到对应项目下会多出一个xxx:version1镜像了
4、拉取镜像

```

docker pull ip:port/项目名/保存的镜像名(例如：xxx:version1)
docker pull local-168-182-110:80/library/goharbor/harbor-exporter:v2.5.6

```
通过docker-compose 快速部署 harbor讲解就先到这里了.


# 第二种方法

## 正确安装方式（推荐）
整个安装流程清晰，可以兼容mac、win、linux等操作系统

你完全可以不采用命令的方式进行以下操作，比如下载解压文件等

// 下载bitnami官方压缩包
wget https://github.com/bitnami/containers/archive/main.tar.gz 

// 解压
tar zxvf main.tar.gz 

// 将harbor-portal目录移动到我们的当前目录
 mv containers-main/bitnami/harbor-portal .
 
 cd harbor-portal
 
 // 启动portal
 docker-compose up
执行完上面的命令，不出意外你将成功启动一套harbor系统。

## harbor默认账号密码
采用本文方法安装的harbor环境，账号密码等信息其实都在harbor-portal/docker-compose.yml文件中，你可以自行查看，比如web面板账号密码如下：

http://localhost/account/sign-in

用户名：admin
密码：bitnami