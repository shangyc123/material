# 一、什么是 Docker Compose

`Docker Compose` 是 Docker 官方编排（Orchestration）项目之一，负责快速的部署分布式应用。

## 1.概述

`Compose` 项目是 Docker 官方的开源项目，负责实现对 Docker 容器集群的快速编排。从功能上看，跟 `OpenStack` 中的 `Heat` 十分类似。

其代码目前在 <https://github.com/docker/compose> 上开源。

`Compose` 定位是 「定义和运行多个 Docker 容器的应用（Defining and running multi-container Docker applications）」，其前身是开源项目 Fig。

通过第一部分中的介绍，我们知道使用一个 `Dockerfile` 模板文件，可以让用户很方便的定义一个单独的应用容器。然而，在日常工作中，经常会碰到需要多个容器相互配合来完成某项任务的情况。例如要实现一个 Web 项目，除了 Web 服务容器本身，往往还需要再加上后端的数据库服务容器，甚至还包括负载均衡容器等。

`Compose` 恰好满足了这样的需求。它允许用户通过一个单独的 `docker-compose.yml` 模板文件（YAML 格式）来定义一组相关联的应用容器为一个项目（project）。

`Compose` 中有两个重要的概念：

- 服务 (`service`)：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。
- 项目 (`project`)：由一组关联的应用容器组成的一个完整业务单元，在 `docker-compose.yml` 文件中定义。

`Compose` 的默认管理对象是项目，通过子命令对项目中的一组容器进行便捷地生命周期管理。

`Compose` 项目由 Python 编写，实现上调用了 Docker 服务提供的 API 来对容器进行管理。因此，只要所操作的平台支持 Docker API，就可以在其上利用 `Compose` 来进行编排管理。



# 二、Docker Compose 安装与卸载

`Compose` 支持 Linux、macOS、Windows 10 三大平台。

`Compose` 可以通过 Python 的包管理工具 `pip` 进行安装，也可以直接下载编译好的二进制文件使用，甚至能够直接在 Docker 容器中运行。

前两种方式是传统方式，适合本地环境下安装使用；最后一种方式则不破坏系统环境，更适合云计算场景。

`Docker for Mac` 、`Docker for Windows` 自带 `docker-compose` 二进制文件，安装 Docker 之后可以直接使用。

```
$ docker-compose --version

docker-compose version 1.17.1, build 6d101fb
```

Linux 系统请使用以下介绍的方法安装。

## 1.二进制包

在 Linux 上的也安装十分简单，从 [官方 GitHub Release](https://github.com/docker/compose/releases) 处直接下载编译好的二进制文件即可。

例如，在 Linux 64 位系统上直接下载对应的二进制包。

```
$ sudo curl -L https://github.com/docker/compose/releases/download/1.24.0-rc1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```

## 2.PIP 安装

*注：* `x86_64` 架构的 Linux 建议按照上边的方法下载二进制包进行安装，如果您计算机的架构是 `ARM`(例如，树莓派)，再使用 `pip` 安装。

这种方式是将 Compose 当作一个 Python 应用来从 pip 源中安装。

执行安装命令：

```
$ sudo pip install -U docker-compose
```

可以看到类似如下输出，说明安装成功。

```
Collecting docker-compose
  Downloading docker-compose-1.17.1.tar.gz (149kB): 149kB downloaded
...
Successfully installed docker-compose cached-property requests texttable websocket-client docker-py dockerpty six enum34 backports.ssl-match-hostname ipaddress
```

## 3.bash 补全命令

```
$ curl -L https://raw.githubusercontent.com/docker/compose/1.8.0/contrib/completion/bash/docker-compose > /etc/bash_completion.d/docker-compose
```

## 4.容器中执行

Compose 既然是一个 Python 应用，自然也可以直接用容器来执行它。

```
$ curl -L https://github.com/docker/compose/releases/download/1.8.0/run.sh > /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```

实际上，查看下载的 `run.sh` 脚本内容，如下

```
set -e

VERSION="1.8.0"
IMAGE="docker/compose:$VERSION"


# Setup options for connecting to docker host
if [ -z "$DOCKER_HOST" ]; then
    DOCKER_HOST="/var/run/docker.sock"
fi
if [ -S "$DOCKER_HOST" ]; then
    DOCKER_ADDR="-v $DOCKER_HOST:$DOCKER_HOST -e DOCKER_HOST"
else
    DOCKER_ADDR="-e DOCKER_HOST -e DOCKER_TLS_VERIFY -e DOCKER_CERT_PATH"
fi


# Setup volume mounts for compose config and context
if [ "$(pwd)" != '/' ]; then
    VOLUMES="-v $(pwd):$(pwd)"
fi
if [ -n "$COMPOSE_FILE" ]; then
    compose_dir=$(dirname $COMPOSE_FILE)
fi
# TODO: also check --file argument
if [ -n "$compose_dir" ]; then
    VOLUMES="$VOLUMES -v $compose_dir:$compose_dir"
fi
if [ -n "$HOME" ]; then
    VOLUMES="$VOLUMES -v $HOME:$HOME -v $HOME:/root" # mount $HOME in /root to share docker.config
fi

# Only allocate tty if we detect one
if [ -t 1 ]; then
    DOCKER_RUN_OPTIONS="-t"
fi
if [ -t 0 ]; then
    DOCKER_RUN_OPTIONS="$DOCKER_RUN_OPTIONS -i"
fi

exec docker run --rm $DOCKER_RUN_OPTIONS $DOCKER_ADDR $COMPOSE_OPTIONS $VOLUMES -w "$(pwd)" $IMAGE "$@"
```

可以看到，它其实是下载了 `docker/compose` 镜像并运行。

## 5.卸载

如果是二进制包方式安装的，删除二进制文件即可。

```
$ sudo rm /usr/local/bin/docker-compose
```

如果是通过 `pip` 安装的，则执行如下命令即可删除。

```
$ sudo pip uninstall docker-compose
```

# 三、Docker Compose 使用

## 1.术语

首先介绍几个术语。

- 服务 (`service`)：一个应用容器，实际上可以运行多个相同镜像的实例。
- 项目 (`project`)：由一组关联的应用容器组成的一个完整业务单元。

可见，一个项目可以由多个服务（容器）关联而成，`Compose` 面向项目进行管理。

## 2.场景

最常见的项目是 web 网站，该项目应该包含 web 应用和缓存。

下面我们用 `Python` 来建立一个能够记录页面访问次数的 web 网站。

### 1）web 应用

新建文件夹，在该目录中编写 `app.py` 文件

```
from flask import Flask
from redis import Redis

app = Flask(__name__)
redis = Redis(host='redis', port=6379)

@app.route('/')
def hello():
    count = redis.incr('hits')
    return 'Hello World! 该页面已被访问 {} 次。\n'.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```

### 2）Dockerfile

编写 `Dockerfile` 文件，内容为

```
FROM python:3.6-alpine
ADD . /code
WORKDIR /code
RUN pip install redis flask
CMD ["python", "app.py"]
```

### 3）docker-compose.yml

编写 `docker-compose.yml` 文件，这个是 Compose 使用的主模板文件。

```
version: '3'
services:

  web:
    build: .
    ports:
     - "5000:5000"
     
  redis:
    image: "redis:alpine"
```

### 4）运行 compose 项目

```
$ docker-compose up
```

此时访问本地 `5000` 端口，每次刷新页面，计数就会加 1。

# 四、Docker Compose 命令说明

## 1.命令对象与格式

对于 Compose 来说，大部分命令的对象既可以是项目本身，也可以指定为项目中的服务或者容器。如果没有特别的说明，命令对象将是项目，这意味着项目中所有的服务都会受到命令影响。

执行 `docker-compose [COMMAND] --help` 或者 `docker-compose help [COMMAND]` 可以查看具体某个命令的使用格式。

`docker-compose` 命令的基本的使用格式是

```
docker-compose [-f=<arg>...] [options] [COMMAND] [ARGS...]
```

## 2.命令选项

- `-f, --file FILE` 指定使用的 Compose 模板文件，默认为 `docker-compose.yml`，可以多次指定。
- `-p, --project-name NAME` 指定项目名称，默认将使用所在目录名称作为项目名。
- `--x-networking` 使用 Docker 的可拔插网络后端特性
- `--x-network-driver DRIVER` 指定网络后端的驱动，默认为 `bridge`
- `--verbose` 输出更多调试信息。
- `-v, --version` 打印版本并退出。

## 3.`build`

格式为 `docker-compose build [options] [SERVICE...]`。

构建（重新构建）项目中的服务容器。

服务容器一旦构建后，将会带上一个标记名，例如对于 web 项目中的一个 db 容器，可能是 web_db。

可以随时在项目目录下运行 `docker-compose build` 来重新构建服务。

选项包括：

- `--force-rm` 删除构建过程中的临时容器。
- `--no-cache` 构建镜像过程中不使用 cache（这将加长构建过程）。
- `--pull` 始终尝试通过 pull 来获取更新版本的镜像。

## 4.`config`

验证 Compose 文件格式是否正确，若正确则显示配置，若格式错误显示错误原因。

## 5.`down`

此命令将会停止 `up` 命令所启动的容器，并移除网络

## 6.`exec`

进入指定的容器。

## 7.`help`

获得一个命令的帮助。

## 8.`images`

列出 Compose 文件中包含的镜像。

## 9.`kill`

格式为 `docker-compose kill [options] [SERVICE...]`。

通过发送 `SIGKILL` 信号来强制停止服务容器。

支持通过 `-s` 参数来指定发送的信号，例如通过如下指令发送 `SIGINT` 信号。

```
$ docker-compose kill -s SIGINT
```

## 10.`logs`

格式为 `docker-compose logs [options] [SERVICE...]`。

查看服务容器的输出。默认情况下，docker-compose 将对不同的服务输出使用不同的颜色来区分。可以通过 `--no-color` 来关闭颜色。

该命令在调试问题的时候十分有用。

## 11.`pause`

格式为 `docker-compose pause [SERVICE...]`。

暂停一个服务容器。

## 12.`port`

格式为 `docker-compose port [options] SERVICE PRIVATE_PORT`。

打印某个容器端口所映射的公共端口。

选项：

- `--protocol=proto` 指定端口协议，tcp（默认值）或者 udp。
- `--index=index` 如果同一服务存在多个容器，指定命令对象容器的序号（默认为 1）。

## 13.`ps`

格式为 `docker-compose ps [options] [SERVICE...]`。

列出项目中目前的所有容器。

选项：

- `-q` 只打印容器的 ID 信息。

## 14.`pull`

格式为 `docker-compose pull [options] [SERVICE...]`。

拉取服务依赖的镜像。

选项：

- `--ignore-pull-failures` 忽略拉取镜像过程中的错误。

## 15.`push`

推送服务依赖的镜像到 Docker 镜像仓库。

## 16.`restart`

格式为 `docker-compose restart [options] [SERVICE...]`。

重启项目中的服务。

选项：

- `-t, --timeout TIMEOUT` 指定重启前停止容器的超时（默认为 10 秒）。

## 17.`rm`

格式为 `docker-compose rm [options] [SERVICE...]`。

删除所有（停止状态的）服务容器。推荐先执行 `docker-compose stop` 命令来停止容器。

选项：

- `-f, --force` 强制直接删除，包括非停止状态的容器。一般尽量不要使用该选项。
- `-v` 删除容器所挂载的数据卷。

## 18.`run`

格式为 `docker-compose run [options] [-p PORT...] [-e KEY=VAL...] SERVICE [COMMAND] [ARGS...]`。

在指定服务上执行一个命令。

例如：

```
$ docker-compose run ubuntu ping docker.com
```

将会启动一个 ubuntu 服务容器，并执行 `ping docker.com` 命令。

默认情况下，如果存在关联，则所有关联的服务将会自动被启动，除非这些服务已经在运行中。

该命令类似启动容器后运行指定的命令，相关卷、链接等等都将会按照配置自动创建。

两个不同点：

- 给定命令将会覆盖原有的自动运行命令；
- 不会自动创建端口，以避免冲突。

如果不希望自动启动关联的容器，可以使用 `--no-deps` 选项，例如

```
$ docker-compose run --no-deps web python manage.py shell
```

将不会启动 web 容器所关联的其它容器。

选项：

- `-d` 后台运行容器。
- `--name NAME` 为容器指定一个名字。
- `--entrypoint CMD` 覆盖默认的容器启动指令。
- `-e KEY=VAL` 设置环境变量值，可多次使用选项来设置多个环境变量。
- `-u, --user=""` 指定运行容器的用户名或者 uid。
- `--no-deps` 不自动启动关联的服务容器。
- `--rm` 运行命令后自动删除容器，`d` 模式下将忽略。
- `-p, --publish=[]` 映射容器端口到本地主机。
- `--service-ports` 配置服务端口并映射到本地主机。
- `-T` 不分配伪 tty，意味着依赖 tty 的指令将无法运行。

## 19.`scale`

格式为 `docker-compose scale [options] [SERVICE=NUM...]`。

设置指定服务运行的容器个数。

通过 `service=num` 的参数来设置数量。例如：

```
$ docker-compose scale web=3 db=2
```

将启动 3 个容器运行 web 服务，2 个容器运行 db 服务。

一般的，当指定数目多于该服务当前实际运行容器，将新创建并启动容器；反之，将停止容器。

选项：

- `-t, --timeout TIMEOUT` 停止容器时候的超时（默认为 10 秒）。

## 20.`start`

格式为 `docker-compose start [SERVICE...]`。

启动已经存在的服务容器。

## 21.`stop`

格式为 `docker-compose stop [options] [SERVICE...]`。

停止已经处于运行状态的容器，但不删除它。通过 `docker-compose start` 可以再次启动这些容器。

选项：

- `-t, --timeout TIMEOUT` 停止容器时候的超时（默认为 10 秒）。

## 22.`top`

查看各个服务容器内运行的进程。

## 23.`unpause`

格式为 `docker-compose unpause [SERVICE...]`。

恢复处于暂停状态中的服务。

## 24.`up`

格式为 `docker-compose up [options] [SERVICE...]`。

该命令十分强大，它将尝试自动完成包括构建镜像，（重新）创建服务，启动服务，并关联服务相关容器的一系列操作。

链接的服务都将会被自动启动，除非已经处于运行状态。

可以说，大部分时候都可以直接通过该命令来启动一个项目。

默认情况，`docker-compose up` 启动的容器都在前台，控制台将会同时打印所有容器的输出信息，可以很方便进行调试。

当通过 `Ctrl-C` 停止命令时，所有容器将会停止。

如果使用 `docker-compose up -d`，将会在后台启动并运行所有的容器。一般推荐生产环境下使用该选项。

默认情况，如果服务容器已经存在，`docker-compose up` 将会尝试停止容器，然后重新创建（保持使用 `volumes-from` 挂载的卷），以保证新启动的服务匹配 `docker-compose.yml` 文件的最新内容。如果用户不希望容器被停止并重新创建，可以使用 `docker-compose up --no-recreate`。这样将只会启动处于停止状态的容器，而忽略已经运行的服务。如果用户只想重新部署某个服务，可以使用 `docker-compose up --no-deps -d <SERVICE_NAME>` 来重新创建服务并后台停止旧服务，启动新服务，并不会影响到其所依赖的服务。

选项：

- `-d` 在后台运行服务容器。
- `--no-color` 不使用颜色来区分不同的服务的控制台输出。
- `--no-deps` 不启动服务所链接的容器。
- `--force-recreate` 强制重新创建容器，不能与 `--no-recreate` 同时使用。
- `--no-recreate` 如果容器已经存在了，则不重新创建，不能与 `--force-recreate` 同时使用。
- `--no-build` 不自动构建缺失的服务镜像。
- `-t, --timeout TIMEOUT` 停止容器时候的超时（默认为 10 秒）。

## 25.`version`

格式为 `docker-compose version`。

打印版本信息。

# 五、Docker Compose 模板文件

模板文件是使用 `Compose` 的核心，涉及到的指令关键字也比较多。但大家不用担心，这里面大部分指令跟 `docker run` 相关参数的含义都是类似的。

默认的模板文件名称为 `docker-compose.yml`，格式为 YAML 格式。

```
version: "3"

services:
  webapp:
    image: examples/web
    ports:
      - "80:80"
    volumes:
      - "/data"
```

注意每个服务都必须通过 `image` 指令指定镜像或 `build` 指令（需要 Dockerfile）等来自动构建生成镜像。

如果使用 `build` 指令，在 `Dockerfile` 中设置的选项(例如：`CMD`, `EXPOSE`, `VOLUME`, `ENV` 等) 将会自动被获取，无需在 `docker-compose.yml` 中再次设置。

下面分别介绍各个指令的用法。

## 1.`build`

指定 `Dockerfile` 所在文件夹的路径（可以是绝对路径，或者相对 docker-compose.yml 文件的路径）。 `Compose` 将会利用它自动构建这个镜像，然后使用这个镜像。

```
version: '3'
services:

  webapp:
    build: ./dir
```

你也可以使用 `context` 指令指定 `Dockerfile` 所在文件夹的路径。

使用 `dockerfile` 指令指定 `Dockerfile` 文件名。

使用 `arg` 指令指定构建镜像时的变量。

```
version: '3'
services:

  webapp:
    build:
      context: ./dir
      dockerfile: Dockerfile-alternate
      args:
        buildno: 1
```

使用 `cache_from` 指定构建镜像的缓存

```
build:
  context: .
  cache_from:
    - alpine:latest
    - corp/web_app:3.14
```

## 2.`cap_add, cap_drop`

指定容器的内核能力（capacity）分配。

例如，让容器拥有所有能力可以指定为：

```
cap_add:
  - ALL
```

去掉 NET_ADMIN 能力可以指定为：

```
cap_drop:
  - NET_ADMIN
```

## 3.`command`

覆盖容器启动后默认执行的命令。

```
command: echo "hello world"
```

## 4.`configs`

仅用于 `Swarm mode`

## 5.`cgroup_parent`

指定父 `cgroup` 组，意味着将继承该组的资源限制。

例如，创建了一个 cgroup 组名称为 `cgroups_1`。

```
cgroup_parent: cgroups_1
```

## 6.`container_name`

指定容器名称。默认将会使用 `项目名称_服务名称_序号` 这样的格式。

```
container_name: docker-web-container
```

> 注意: 指定容器名称后，该服务将无法进行扩展（scale），因为 Docker 不允许多个容器具有相同的名称。

## 7.`deploy`

仅用于 `Swarm mode`

## 8.`devices`

指定设备映射关系。

```
devices:
  - "/dev/ttyUSB1:/dev/ttyUSB0"
```

## 9.`depends_on`

解决容器的依赖、启动先后的问题。以下例子中会先启动 `redis` `db` 再启动 `web`

```
version: '3'

services:
  web:
    build: .
    depends_on:
      - db
      - redis

  redis:
    image: redis

  db:
    image: postgres
```

> 注意：`web` 服务不会等待 `redis` `db` 「完全启动」之后才启动。

## 10.`dns`

自定义 `DNS` 服务器。可以是一个值，也可以是一个列表。

```
dns: 8.8.8.8

dns:
  - 8.8.8.8
  - 114.114.114.114
```

## 11.`dns_search`

配置 `DNS` 搜索域。可以是一个值，也可以是一个列表。

```
dns_search: example.com

dns_search:
  - domain1.example.com
  - domain2.example.com
```

## 12.`tmpfs`

挂载一个 tmpfs 文件系统到容器。

```
tmpfs: /run
tmpfs:
  - /run
  - /tmp
```

## 13.`env_file`

从文件中获取环境变量，可以为单独的文件路径或列表。

如果通过 `docker-compose -f FILE` 方式来指定 Compose 模板文件，则 `env_file` 中变量的路径会基于模板文件路径。

如果有变量名称与 `environment` 指令冲突，则按照惯例，以后者为准。

```
env_file: .env

env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/secrets.env
```

环境变量文件中每一行必须符合格式，支持 `#` 开头的注释行。

```
# common.env: Set development environment
PROG_ENV=development
```

## 14.`environment`

设置环境变量。你可以使用数组或字典两种格式。

只给定名称的变量会自动获取运行 Compose 主机上对应变量的值，可以用来防止泄露不必要的数据。

```
environment:
  RACK_ENV: development
  SESSION_SECRET:

environment:
  - RACK_ENV=development
  - SESSION_SECRET
```

如果变量名称或者值中用到 `true|false，yes|no` 等表达 [布尔](http://yaml.org/type/bool.html) 含义的词汇，最好放到引号里，避免 YAML 自动解析某些内容为对应的布尔语义。这些特定词汇，包括

```
y|Y|yes|Yes|YES|n|N|no|No|NO|true|True|TRUE|false|False|FALSE|on|On|ON|off|Off|OFF
```

## 15.`expose`

暴露端口，但不映射到宿主机，只被连接的服务访问。

仅可以指定内部端口为参数

```
expose:
 - "3000"
 - "8000"
```

## 16.`external_links`

> 注意：不建议使用该指令。

链接到 `docker-compose.yml` 外部的容器，甚至并非 `Compose` 管理的外部容器。

```
external_links:
 - redis_1
 - project_db_1:mysql
 - project_db_1:postgresql
```

## 17.`extra_hosts`

类似 Docker 中的 `--add-host` 参数，指定额外的 host 名称映射信息。

```
extra_hosts:
 - "googledns:8.8.8.8"
 - "dockerhub:52.1.157.61"
```

会在启动后的服务容器中 `/etc/hosts` 文件中添加如下两条条目。

```
8.8.8.8 googledns
52.1.157.61 dockerhub
```

## 18.`healthcheck`

通过命令检查容器是否健康运行。

```
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
```

## 19.`image`

指定为镜像名称或镜像 ID。如果镜像在本地不存在，`Compose` 将会尝试拉取这个镜像。

```
image: ubuntu
image: orchardup/postgresql
image: a4bc65fd
```

## 20.`labels`

为容器添加 Docker 元数据（metadata）信息。例如可以为容器添加辅助说明信息。

```
labels:
  com.startupteam.description: "webapp for a startup team"
  com.startupteam.department: "devops department"
  com.startupteam.release: "rc3 for v1.0"
```

## 21.`links`

> 注意：不推荐使用该指令。

## 22.`logging`

配置日志选项。

```
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.0.42:123"
```

目前支持三种日志驱动类型。

```
driver: "json-file"
driver: "syslog"
driver: "none"
```

`options` 配置日志驱动的相关参数。

```
options:
  max-size: "200k"
  max-file: "10"
```

## 23.`network_mode`

设置网络模式。使用和 `docker run` 的 `--network` 参数一样的值。

```
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
```

## 24.`networks`

配置容器连接的网络。

```
version: "3"
services:

  some-service:
    networks:
     - some-network
     - other-network

networks:
  some-network:
  other-network:

```

## 25.`pid`

跟主机系统共享进程命名空间。打开该选项的容器之间，以及容器和宿主机系统之间可以通过进程 ID 来相互访问和操作。

```
pid: "host"
```

## 26.`ports`

暴露端口信息。

使用宿主端口：容器端口 `(HOST:CONTAINER)` 格式，或者仅仅指定容器的端口（宿主将会随机选择端口）都可以。

```
ports:
 - "3000"
 - "8000:8000"
 - "49100:22"
 - "127.0.0.1:8001:8001"
```

*注意：当使用 HOST:CONTAINER 格式来映射端口时，如果你使用的容器端口小于 60 并且没放到引号里，可能会得到错误结果，因为 YAML 会自动解析 xx:yy 这种数字格式为 60 进制。为避免出现这种问题，建议数字串都采用引号包括起来的字符串格式。*

## 27.`secrets`

存储敏感数据，例如 `mysql` 服务密码。

```
version: "3.1"
services:

mysql:
  image: mysql
  environment:
    MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
  secrets:
    - db_root_password
    - my_other_secret

secrets:
  my_secret:
    file: ./my_secret.txt
  my_other_secret:
    external: true
```

## 28.`security_opt`

指定容器模板标签（label）机制的默认属性（用户、角色、类型、级别等）。例如配置标签的用户名和角色名。

```
security_opt:
    - label:user:USER
    - label:role:ROLE
```

## 29.`stop_signal`

设置另一个信号来停止容器。在默认情况下使用的是 SIGTERM 停止容器。

```
stop_signal: SIGUSR1
```

## 30.`sysctls`

配置容器内核参数。

```
sysctls:
  net.core.somaxconn: 1024
  net.ipv4.tcp_syncookies: 0

sysctls:
  - net.core.somaxconn=1024
  - net.ipv4.tcp_syncookies=0
```

## 31.`ulimits`

指定容器的 ulimits 限制值。

例如，指定最大进程数为 65535，指定文件句柄数为 20000（软限制，应用可以随时修改，不能超过硬限制） 和 40000（系统硬限制，只能 root 用户提高）。

```
  ulimits:
    nproc: 65535
    nofile:
      soft: 20000
      hard: 40000
```

## 32.`volumes`

数据卷所挂载路径设置。可以设置宿主机路径 （`HOST:CONTAINER`） 或加上访问模式 （`HOST:CONTAINER:ro`）。

该指令中路径支持相对路径。

```
volumes:
 - /var/lib/mysql
 - cache/:/tmp/cache
 - ~/configs:/etc/configs/:ro
```

## 33.其它指令

此外，还有包括 `domainname, entrypoint, hostname, ipc, mac_address, privileged, read_only, shm_size, restart, stdin_open, tty, user, working_dir` 等指令，基本跟 `docker run` 中对应参数的功能一致。

指定服务容器启动后执行的入口文件。

```
entrypoint: /code/entrypoint.sh
```

指定容器中运行应用的用户名。

```
user: nginx
```

指定容器中工作目录。

```
working_dir: /code
```

指定容器中搜索域名、主机名、mac 地址等。

```
domainname: your_website.com
hostname: test
mac_address: 08-00-27-00-0C-0A
```

允许容器中运行一些特权命令。

```
privileged: true
```

指定容器退出后的重启策略为始终重启。该命令对保持服务始终运行十分有效，在生产环境中推荐配置为 `always` 或者 `unless-stopped`。

```
restart: always
```

以只读模式挂载容器的 root 文件系统，意味着不能对容器内容进行修改。

```
read_only: true
```

打开标准输入，可以接受外部输入。

```
stdin_open: true
```

模拟一个伪终端。

```
tty: true
```

## 34.读取变量

Compose 模板文件支持动态读取主机的系统环境变量和当前目录下的 `.env` 文件中的变量。

例如，下面的 Compose 文件将从运行它的环境中读取变量 `${MONGO_VERSION}` 的值，并写入执行的指令中。

```
version: "3"
services:

db:
  image: "mongo:${MONGO_VERSION}"
```

如果执行 `MONGO_VERSION=3.2 docker-compose up` 则会启动一个 `mongo:3.2` 镜像的容器；如果执行 `MONGO_VERSION=2.8 docker-compose up` 则会启动一个 `mongo:2.8` 镜像的容器。

若当前目录存在 `.env` 文件，执行 `docker-compose` 命令时将从该文件中读取变量。

在当前目录新建 `.env` 文件并写入以下内容。

```
# 支持 # 号注释
MONGO_VERSION=3.6
```

执行 `docker-compose up` 则会启动一个 `mongo:3.6` 镜像的容器。

# 六、Docker Compose 实战 Tomcat

```
version: '3.1'
services:
  tomcat:
    restart: always
    image: tomcat
    container_name: tomcat
    ports:
      - 8080:8080
    volumes:
      - /usr/local/docker/tomcat/webapps/test:/usr/local/tomcat/webapps/test
    environment:
      TZ: Asia/Shanghai
```

# 七、Docker Compose 实战 MySQL

## 1.MySQL5

```
version: '3.1'
services:
  mysql:
    restart: always
    image: mysql:5.7.25
    container_name: mysql
    ports:
      - 3306:3306
    environment:
      TZ: Asia/Shanghai
      MYSQL_ROOT_PASSWORD: 123
    command:
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
      --max_allowed_packet=128M
      --sql-mode="STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO"
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:
```

## 2.MySQL8

```
version: '3.1'
services:
  db:
    image: mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 123456
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
    ports:
      - 3306:3306
    volumes:
      - ./data:/var/lib/mysql

  adminer:
    image: adminer
    restart: always
    ports:
      - 8080:8080
```

# 八、Docker Compose 部署项目到容器

# 九、Docker Compose 常用命令

## 1.前台运行

```
docker-compose up
```

## 2.后台运行

```
docker-compose up -d
```

## 3.启动

```
docker-compose start
```

## 4.停止

```
docker-compose stop
```

## 5.停止并移除容器

```
docker-compose down
```

# 十、YAML 配置文件语言

## 1.简介

YAML 是专门用来写配置文件的语言   ，非常简洁和强大，远比 JSON 格式方便。

YAML 语言的设计目标，就是方便人类读写。它实质上是一种通用的数据串行化格式。它的基本语法规则如下：

- 大小写敏感
- 使用缩进表示层级关系
- 缩进时不允许使用Tab键，只允许使用空格。
- 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可

`#` 表示注释，从这个字符一直到行尾，都会被解析器忽略。

YAML 支持的数据结构有三种：

- 对象：键值对的集合，又称为映射（mapping）/ 哈希（hashes） / 字典（dictionary）
- 数组：一组按次序排列的值，又称为序列（sequence） / 列表（list）
- 纯量（scalars）：单个的、不可再分的值

## 2.对象

对象的一组键值对，使用冒号结构表示

```
animal: pets
```

## 3.数组

一组连词线开头的行，构成一个数组

```
- Cat
- Dog
- Goldfish
```

数据结构的子成员是一个数组，则可以在该项下面缩进一个空格

```
- Array
 - Cat
 - Dog
 - Goldfish
```

## 4.复合结构

对象和数组可以结合使用，形成复合结构

```
languages:
 - Ruby
 - Perl
 - Python 
websites:
 YAML: yaml.org 
 Ruby: ruby-lang.org 
 Python: python.org 
 Perl: use.perl.org 
```

## 5.纯量

纯量是最基本的、不可再分的值。以下数据类型都属于 JavaScript 的纯量

- 字符串
- 布尔值
- 整数
- 浮点数
- Null
- 时间
- 日期

# 附：为什么说 JSON 不适合做配置文件？

> 很多项目使用 JSON 作为配置文件，最明显的例子就是 npm 和 yarn 使用的 package.json 文件。当然，还有很多其他文件，例如 CloudFormation（最初只有 JSON，但现在也支持 YAML）和 composer（PHP）。

但是，JSON 实际上是一种非常糟糕的配置语言。别误会我的意思，我其实是喜欢 JSON 的。它是一种相对灵活的文本格式，对于机器和人类来说都很容易阅读，而且是一种非常好的数据交换和存储格式。但作为一种配置语言，它有它的不足。

## 1.为什么流行使用 JSON 作为配置语言？

将 JSON 用作配置文件有几个方面的原因，其中最大的原因可能是它很容易实现。很多编程语言的标准库都支持 JSON，开发人员或用户可能已经很熟悉 JSON，所以不需要学习新的配置格式就可以使用那些产品。现在几乎所有的工具都提供 JSON 支持，包括语法突出显示、自动格式化、验证工具等。

这些都是很好的理由，但这种无处不在的格式其实不适合用作配置。

## 2.JSON 的问题 

### 1）缺乏注释

注释对于配置语言而言绝对是一个重要的功能。注释可用于标注不同的配置选项、解释为什么要配置成特定的值，更重要的是，在使用不同的配置进行测试和调试时需要临时注释掉部分配置。当然，如果只是把 JSON 当作是一种数据交换格式，那么就不需要用到注释。

我们可以通过一些方法给 JSON 添加注释。一种常见的方法是在对象中使用特殊的键作为注释，例如“//”或“__comment”。但是，这种语法的可读性不高，并且为了在单个对象中包含多个注释，需要为每个注释使用唯一的键。David Crockford（JSON 的发明者）建议使用预处理器来删除注释。如果你的应用程序需要使用 JSON 作为配置，那么完全没问题，不过这确实带来了一些额外的工作量。

一些 JSON 库允许将注释作为输入。例如，Ruby 的 JSON 模块和启用了 JsonParser.Feature.ALLOW_COMMENTS 功能的 Java Jackson 库可以处理 JavaScript 风格的注释。但是，这不是标准的方式，而且很多编辑器无法正确处理 JSON 文件中的注释，这让编辑它们变得更加困难。

### 2）过于严格

JSON 规范非常严格，这也是为什么实现 JSON 解析器会这么简单，但在我看来，它还会影响可读性，并且在较小程度上会影响可写性。

### 3）低信噪比

与其他配置语言相比，JSON 显得非常嘈杂。JSON 的很多标点符号对可读性毫无帮助，况且，对象中的键几乎都是标识符，所以键的引号其实是多余的。

此外，JSON 需要使用花括号将整个文档包围起来，所以 JSON 是 JavaScript 的子集，并在流中发送多个对象时用于界定不同的对象。但是，对于配置文件来说，最外面的大括号其实没有任何用处。在配置文件中，键值对之间的逗号也是没有必要的。通常情况下，每行只有一个键值对，所以使用换行作为分隔符更有意义。

说到逗号，JSON 居然不允许在结尾出现逗号。如果你需要在每个键值对之后使用逗号，那么至少应该接受结尾的逗号，因为有了结尾的逗号，在添加新条目时会更容易，而且在进行 commit diff 时也更清晰。

### 4）长字符串

JSON 作为配置格式的另一个问题是，它不支持多行字符串。如果你想在字符串中换行，必须使用 “\n” 进行转义，更糟糕的是，如果你想要一个字符串在文件中另起一行显示，那就彻底没办法了。如果你的配置项里没有很长的字符串，那就不是问题。但是，如果你的配置项里包括了长字符串，例如项目描述或 GPG 密钥，你可能不希望只是使用 “\n” 来转义而不是使用真实的换行符。

### 5）数字

此外，在某些情况下，JSON 对数字的定义可能会有问题。JSON 规范中将数字定义成使用十进制表示的任意精度有限浮点数。对于大多数应用程序来说，这没有问题。但是，如果你需要使用十六进制表示法或表示无穷大或 NaN 等值时，那么 TOML 或 YAML 将能够更好地处理它们。

```
{

  "name": "example",

  "description": "A really long description that needs multiple lines.\nThis is a sample project to illustrate why JSON is not a good configuration format. This description is pretty long, but it doesn't have any way to go onto multiple lines.",

  "version": "0.0.1",

  "main": "index.js",

  "//": "This is as close to a comment as you are going to get",

  "keywords": ["example", "config"],

  "scripts": {

    "test": "./test.sh",

    "do_stuff": "./do_stuff.sh"

  },

  "bugs": {

    "url": "https://example.com/bugs"

  },

  "contributors": [{

    "name": "John Doe",

    "email": "johndoe@example.com"

  }, {

    "name": "Ivy Lane",

    "url": "https://example.com/ivylane"

  }],

  "dependencies": {

    "dep1": "^1.0.0",

    "dep2": "3.40",

    "dep3": "6.7"

  }

}
```



## 2.JSON 的替代方案

选择哪一种配置语言取决于你的应用程序。每种语言都有各自的优缺点，下面列出了一些可以考虑的选项。它们都是为配置而设计的语言，每一种都比 JSON 这样的数据语言更好。

```
name = "example"

description = """

A really long description that needs multiple lines.

This is a sample project to illustrate why JSON is not a \

good configuration format. This description is pretty long, \

but it doesn't have any way to go onto multiple lines."""



version = "0.0.1"

main = "index.js"

# This is a comment

keywords = ["example", "config"]



[bugs]

url = "https://example.com/bugs"



[scripts]



test = "./test.sh"

do_stuff = "./do_stuff.sh"



[[contributors]]

name = "John Doe"

email = "johndow@example.com"



[[contributors]]

name = "Ivy Lane"

url = "https://example.com/ivylane"



[dependencies]



dep1 = "^1.0.0"

# Why we depend on dep2

dep2 = "3.40"

dep3 = "6.7"
```

### 1）HJSON

HJSON 是一种基于 JSON 的格式，但具有更大的灵活性，可读性也更强。它支持注释、多行字符串、不带引号的键和字符串，以及可选的逗号。如果你想要 JSON 结构的简单性，同时对配置文件更友好，那么可以考虑 HJSON。有一些可以将 HJSON 转换为 JSON 的命令行工具，如果你使用的工具是基于 JSON 的，可以先用 HJSON 编写配置，然后再转换成 JSON。JSON5 是另一个与 HJSON 非常相似的配置语言。

```
{

  name: example

  description: '''

  A really long description that needs multiple lines.

  This is a sample project to illustrate why JSON is 

  not a good configuration format.  This description 

  is pretty long, but it doesn't have any way to go 

  onto multiple lines.

  '''

  version: 0.0.1

  main: index.js

  # This is a a comment

  keywords: ["example", "config"]

  scripts: {

    test: ./test.sh

    do_stuff: ./do_stuff.sh

  }

  bugs: {

    url: https://example.com/bugs

  }

  contributors: [{

    name: John Doe

    email: johndoe@example.com

  } {

    name: Ivy Lane

    url: https://example.com/ivylane

  }]

  dependencies: {

    dep1: ^1.0.0

    # Why we have this dependency

    dep2: "3.40"

    dep3: "6.7"

  }

}
```

### 2）HOCON

HOCON 是为 Play 框架设计的配置格式，在 Scala 项目中非常流行。它是 JSON 的超集，因此可以使用现有的 JSON 文件。除了注释、可选逗号和多行字符串这些标准特性外，HOCON 还支持从其他文件导入和引用其他值的键，避免重复代码，并使用以点作为分隔符的键来指定值的路径，因此用户可以不必将所有值直接放在花括号对象中。

```
name = example

description = """

A really long description that needs multiple lines.



This is a sample project to illustrate why JSON is 

not a good configuration format.  This description 

is pretty long, but it doesn't have any way to go 

onto multiple lines.

"""

version = 0.0.1

main = index.js

# This is a a comment

keywords = ["example", "config"]

scripts {

  test = ./test.sh

  do_stuff = ./do_stuff.sh

}

bugs.url = "https://example.com/bugs"

contributors = [

  {

    name = John Doe

    email = johndoe@example.com

  }

  {

    name = Ivy Lane

    url = "https://example.com/ivylane"

  }

]

dependencies {

  dep1 = ^1.0.0

  # Why we have this dependency

  dep2 = "3.40"

  dep3 = "6.7"

}
```

### 3）YAML

YAML（YAML 不是标记语言）是一种非常灵活的格式，几乎是 JSON 的超集，已经被用在一些著名的项目中，如 Travis CI、Circle CI 和 AWS CloudFormation。YAML 的库几乎和 JSON 一样无处不在。除了支持注释、换行符分隔、多行字符串、裸字符串和更灵活的类型系统之外，YAML 也支持引用文件，以避免重复代码。

YAML 的主要缺点是规范非常复杂，不同的实现之间可能存在不一致的情况。它将缩进视为严格语法的一部分（类似于 Python），有些人喜欢，有些人不喜欢。这会让复制和粘贴变得很麻烦。

### 4）脚本语言

如果你的应用程序是使用 Python 或 Ruby 等脚本语言开发的，并且你知道配置的来源是可靠的，那么最好的选择可能就是使用这些语言进行配置。如果你需要一个真正灵活的配置选项，也可以在编译语言中嵌入诸如 Lua 之类的脚本语言。这样可以获得脚本语言的灵活性，而且比使用不同的配置语言更容易实现。使用脚本语言的缺点是它可能过于强大，当然，如果配置来源是不受信任的，可能会引入严重的安全问题。

### 5）自定义配置格式

如果由于某种原因，键值配置格式不能满足你的要求，并且由于性能或大小限制而无法使用脚本语言，那么可以考虑自定义配置格式。如果是这种情况，那么在做出选择之前要想清楚，因为你不仅要编写和维护一个解析器，还要让你的用户熟悉另一种配置格式。

## 结论

有了这么多更好的配置语言，没有理由还要使用 JSON。如果要创建需要用到配置的新应用程序、框架或库，请选择 JSON 以外的其他选项。

英文原文：https://www.lucidchart.com/techblog/2018/07/16/why-json-isnt-a-good-configuration-language/