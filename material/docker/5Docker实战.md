# 一、数据卷

## 1.概述

`数据卷` 是一个可供一个或多个容器使用的特殊目录，它绕过 UFS，可以提供很多有用的特性：

- `数据卷` 可以在容器之间共享和重用
- 对 `数据卷` 的修改会立马生效
- 对 `数据卷` 的更新，不会影响镜像
- `数据卷` 默认会一直存在，即使容器被删除

> 注意：`数据卷` 的使用，类似于 Linux 下对目录或文件进行 mount，镜像中的被指定为挂载点的目录中的文件会隐藏掉，能显示看的是挂载的 `数据卷`。

## 2.选择 -v 还是 -–mount 参数

Docker 新用户应该选择 `--mount` 参数，经验丰富的 Docker 使用者对 `-v` 或者 `--volume` 已经很熟悉了，但是推荐使用 `--mount` 参数。

## 3.创建一个数据卷

```
$ docker volume create my-vol
```

查看所有的 `数据卷`

```
$ docker volume ls

local               my-vol
```

在主机里使用以下命令可以查看指定 `数据卷` 的信息

```
$ docker volume inspect my-vol
[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```

## 4.启动一个挂载数据卷的容器

在用 `docker run` 命令的时候，使用 `--mount` 标记来将 `数据卷` 挂载到容器里。在一次 `docker run`中可以挂载多个 `数据卷`。

下面创建一个名为 `web` 的容器，并加载一个 `数据卷` 到容器的 `/webapp` 目录。

```
$ docker run -d -P \
    --name web \
    # -v my-vol:/wepapp \
    --mount source=my-vol,target=/webapp \
    training/webapp \
    python app.py
```

## 5.查看数据卷的具体信息

在主机里使用以下命令可以查看 `web` 容器的信息

```
$ docker inspect web
```

`数据卷` 信息在 "Mounts" Key 下面

```
"Mounts": [
    {
        "Type": "volume",
        "Name": "my-vol",
        "Source": "/var/lib/docker/volumes/my-vol/_data",
        "Destination": "/app",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    }
],
```

## 6.删除数据卷

```
$ docker volume rm my-vol
```

`数据卷` 是被设计用来持久化数据的，它的生命周期独立于容器，Docker 不会在容器被删除后自动删除 `数据卷`，并且也不存在垃圾回收这样的机制来处理没有任何容器引用的 `数据卷`。如果需要在删除容器的同时移除数据卷。可以在删除容器的时候使用 `docker rm -v` 这个命令。

无主的数据卷可能会占据很多空间，要清理请使用以下命令

```
$ docker volume prune
```

# 二、Docker 构建 Tomcat

## 1.查找 Docker Hub 上的 Tomcat 镜像

```
root@UbuntuBase:/usr/local/docker/tomcat# docker search tomcat
NAME                           DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
tomcat                         Apache Tomcat is an open source implementa...   1550                [OK]                
dordoka/tomcat                 Ubuntu 14.04, Oracle JDK 8 and Tomcat 8 ba...   43                                      [OK]
tomee                          Apache TomEE is an all-Apache Java EE cert...   42                  [OK]                
davidcaste/alpine-tomcat       Apache Tomcat 7/8 using Oracle Java 7/8 wi...   21                                      [OK]
consol/tomcat-7.0              Tomcat 7.0.57, 8080, "admin/admin"              16                                      [OK]
cloudesire/tomcat              Tomcat server, 6/7/8                            15                                      [OK]
maluuba/tomcat7                                                                9                                       [OK]
tutum/tomcat                   Base docker image to run a Tomcat applicat...   8                                       
jeanblanchard/tomcat           Minimal Docker image with Apache Tomcat         8                                       
andreptb/tomcat                Debian Jessie based image with Apache Tomc...   7                                       [OK]
bitnami/tomcat                 Bitnami Tomcat Docker Image                     5                                       [OK]
aallam/tomcat-mysql            Debian, Oracle JDK, Tomcat & MySQL              4                                       [OK]
antoineco/tomcat-mod_cluster   Apache Tomcat with JBoss mod_cluster            1                                       [OK]
maluuba/tomcat7-java8          Tomcat7 with java8.                             1                                       
amd64/tomcat                   Apache Tomcat is an open source implementa...   1                                       
primetoninc/tomcat             Apache tomcat 8.5, 8.0, 7.0                     1                                       [OK]
trollin/tomcat                                                                 0                                       
fabric8/tomcat-8               Fabric8 Tomcat 8 Image                          0                                       [OK]
awscory/tomcat                 tomcat                                          0                                       
oobsri/tomcat8                 Testing CI Jobs with different names.           0                                       
hegand/tomcat                  docker-tomcat                                   0                                       [OK]
s390x/tomcat                   Apache Tomcat is an open source implementa...   0                                       
ppc64le/tomcat                 Apache Tomcat is an open source implementa...   0                                       
99taxis/tomcat7                Tomcat7                                         0                                       [OK]
qminderapp/tomcat7             Tomcat 7                                        0
```

这里我们拉取官方的镜像

```
docker pull tomcat
```

等待下载完成后，我们就可以在本地镜像列表里查到 REPOSITORY 为 tomcat 的镜像。

## 2.运行容器：

```
docker run --name tomcat -p 8080:8080 -v $PWD/test:/usr/local/tomcat/webapps/test -d tomcat
```

命令说明：

- -p 8080:8080：将容器的8080端口映射到主机的8080端口
- -v $PWD/test:/usr/local/tomcat/webapps/test：将主机中当前目录下的test挂载到容器的/test

查看容器启动情况

```
root@UbuntuBase:/usr/local/docker/tomcat/webapps# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
38498e53128c        tomcat              "catalina.sh run"   2 minutes ago       Up 2 minutes        0.0.0.0:8080->8080/tcp   tomcat
```

通过浏览器访问

# 三、Docker 构建 MySQL

## 1.查找 Docker Hub 上的 MySQL 镜像

```
root@UbuntuBase:/usr/local/docker/mysql# docker search mysql
NAME                                                   DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
mysql                                                  MySQL is a widely used, open-source relati...   5177                [OK]                
mariadb                                                MariaDB is a community-developed fork of M...   1602                [OK]                
mysql/mysql-server                                     Optimized MySQL Server Docker images. Crea...   361                                     [OK]
percona                                                Percona Server is a fork of the MySQL rela...   298                 [OK]                
hypriot/rpi-mysql                                      RPi-compatible Docker Image with Mysql          72                                      
zabbix/zabbix-server-mysql                             Zabbix Server with MySQL database support       62                                      [OK]
centurylink/mysql                                      Image containing mysql. Optimized to be li...   53                                      [OK]
sameersbn/mysql                                                                                        48                                      [OK]
zabbix/zabbix-web-nginx-mysql                          Zabbix frontend based on Nginx web-server ...   36                                      [OK]
tutum/mysql                                            Base docker image to run a MySQL database ...   27                                      
1and1internet/ubuntu-16-nginx-php-phpmyadmin-mysql-5   ubuntu-16-nginx-php-phpmyadmin-mysql-5          17                                      [OK]
schickling/mysql-backup-s3                             Backup MySQL to S3 (supports periodic back...   16                                      [OK]
centos/mysql-57-centos7                                MySQL 5.7 SQL database server                   15                                      
linuxserver/mysql                                      A Mysql container, brought to you by Linux...   12                                      
centos/mysql-56-centos7                                MySQL 5.6 SQL database server                   6                                       
openshift/mysql-55-centos7                             DEPRECATED: A Centos7 based MySQL v5.5 ima...   6                                       
frodenas/mysql                                         A Docker Image for MySQL                        3                                       [OK]
dsteinkopf/backup-all-mysql                            backup all DBs in a mysql server                3                                       [OK]
circleci/mysql                                         MySQL is a widely used, open-source relati...   2                                       
cloudposse/mysql                                       Improved `mysql` service with support for ...   0                                       [OK]
astronomerio/mysql-sink                                MySQL sink                                      0                                       [OK]
ansibleplaybookbundle/rhscl-mysql-apb                  An APB which deploys RHSCL MySQL                0                                       [OK]
cloudfoundry/cf-mysql-ci                               Image used in CI of cf-mysql-release            0                                       
astronomerio/mysql-source                              MySQL source                                    0                                       [OK]
jenkler/mysql                                          Docker Mysql package                            0                                       
```

这里我们拉取官方的镜像

```
docker pull mysql
```

等待下载完成后，我们就可以在本地镜像列表里查到 REPOSITORY 为 mysql 的镜像

## 2.运行容器：

```
docker run -p 3306:3306 --name mysql \
-v /usr/local/docker/mysql/conf:/etc/mysql \
-v /usr/local/docker/mysql/logs:/var/log/mysql \
-v /usr/local/docker/mysql/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123 \
-d mysql:5.7.25
```

命令参数：

- `-p 3306:3306`：将容器的3306端口映射到主机的3306端口
- `-v /usr/local/docker/mysql/conf:/etc/mysql`：将主机当前目录下的 conf 挂载到容器的 /etc/mysql
- `-v /usr/local/docker/mysql/logs:/var/log/mysql`：将主机当前目录下的 logs 目录挂载到容器的 /var/log/mysql
- `-v /usr/local/docker/mysql/data:/var/lib/mysql`：将主机当前目录下的 data 目录挂载到容器的 /var/lib/mysql
- `-e MYSQL\_ROOT\_PASSWORD=123456`：初始化root用户的密码

查看容器启动情况

```
root@UbuntuBase:/usr/local/docker/mysql# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
bc49c9de4cdf        mysql:latest        "docker-entrypoint..."   4 minutes ago       Up 4 minutes        0.0.0.0:3306->3306/tcp   mysql
```

使用客户端工具连接 MySQL

# 四、部署项目到容器

# 五、Docker 常用命令

## 1.查看 Docker 版本

```
docker version
```

## 2.从 Docker 文件构建 Docker 映像

```
docker build -t image-name docker-file-location
```

## 3.运行 Docker 映像

```
docker run -d image-name
```

## 4.查看可用的 Docker 映像

```
docker images
```

## 5.查看最近的运行容器

```
docker ps -l
```

## 6.查看所有正在运行的容器

```
docker ps -a
```

## 7.停止运行容器

```
docker stop container_id
```

## 8.删除一个镜像

```
docker rmi image-name
```

## 9.删除所有镜像

```
docker rmi $(docker images -q)
```

## 10.强制删除所有镜像

```
docker rmi -r $(docker images -q)
```

## 11.删除所有虚悬镜像

```
docker rmi $(docker images -q -f dangling=true)
```

## 12.删除所有容器

```
docker rm $(docker ps -a -q)
```

## 13.进入 Docker 容器

```
docker exec -it container-id /bin/bash
```

## 14.查看所有数据卷

```
docker volume ls
```

## 15.删除指定数据卷

```
docker volume rm [volume_name]
```

## 16.删除所有未关联的数据卷

```
docker volume rm $(docker volume ls -qf dangling=true)
```

## 17.从主机复制文件到容器

```
sudo docker cp host_path containerID:container_path
```

## 18.从容器复制文件到主机

```
sudo docker cp containerID:container_path host_path
```