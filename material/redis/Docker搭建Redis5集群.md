# 概述



### Author  July



2018年十月 Redis 发布了稳定版本的 5.0 版本，推出了各种新特性，其中一点是放弃 Ruby的集群方式，改为 使用 C语言编写的 redis-cli的方式，使集群的构建方式复杂度大大降低。关于集群的更新可以在 Redis5 的版本说明中看到，如下：

> The cluster manager was ported from Ruby (redis-trib.rb) to C code inside redis-cli. check `redis-cli --cluster help ` for more info.

可以查看Redis官网查看集群搭建方式，连接如下

https://redis.io/topics/cluster-tutorial

接下来是使用Docker搭建有6个节点（三主三从）的 Redis集群。







# 集群Redis

## 环境

192.168.2.148
192.168.2.149
192.168.2.149

## 创建文件夹

三台分别执行

```shell
mkdir /usr/local/docker/redis-cluster/6379/data

mkdir /usr/local/docker/redis-cluster/6380/data

mkdir /usr/local/docker/redis-cluster/6379/conf

mkdir /usr/local/docker/redis-cluster/6380/conf
```



## 准备redis.conf

三台分别执行



### vim 6379/redis.conf

```shell
protected-mode no #非保护模式

port 6379 #端口

pidfile /var/run/redis_6379.pid #需要修改为 reids_{port}.pid 的形式

appendonly yes #开启AOF日志 指定持久化方式

cluster-enabled yes #开启集群

cluster-config-file nodes-6379.conf #集群的配置文件 nodes_{port}.conf的形式

cluster-node-timeout 5000 #超时时间

```



### vim 6380/redis.conf

```shell

protected-mode no #非保护模式

port 6380 #端口

pidfile /var/run/redis_6380.pid #需要修改为 reids_{port}.pid 的形式

appendonly yes #开启AOF日志 指定持久化方式

cluster-enabled yes #开启集群

cluster-config-file nodes-6380.conf #集群的配置文件 nodes_{port}.conf的形式

cluster-node-timeout 5000 #超时时间

```



## 创建并启动容器

### 主节点6379：

```shell
docker run -p 6379:6379 -p 16379:16379 -v /usr/local/docker/redis-cluster/6379/conf/redis.conf:/usr/local/etc/redis/redis.conf -v /usr/local/docker/redis-cluster/6379/data:/data --privileged=true --name redis1 -d redis redis-server /usr/local/etc/redis/redis.conf
```

注意要在主节点上映射16379端口，16379是集群的总线端口，需要映射出来。

- redis集群总线端口为redis客户端端口加上10000，比如说你的redis 6379端口为客户端通讯端口，那么16379端口为集群总线端口

![img](D:\2019\1902\第四阶段\day15Redis\资料\pic\1.jpg) 

### 从节点6380：

```shell
docker run -p 6380:6380 -v /usr/local/docker/redis-cluster/6380/conf/redis.conf:/usr/local/etc/redis/redis.conf -v /usr/local/docker/redis-cluster/6380/data:/data --privileged=true --name redis2 -d redis redis-server /usr/local/etc/redis/redis.conf
```

从节点不需要映射16379端口



## 查看是否启动成功

```
docker ps
```



## 创建集群

选择一台redis服务器来创建集群

```shell
docker exec -it myredis bash
```

进入容器

```shell
cd /usr/local/bin
```

执行命令

```shell
./redis-cli --cluster create 192.168.2.148:6379 192.168.2.148:6380 192.168.2.149:6379 192.168.2.149:6380 192.168.2.150:6379 192.168.2.150:6380 --cluster-replicas 1
```

看到如下界面，输入yes，完成创建

![1564661526863](D:\2019\1902\第四阶段\day15Redis\资料\pic\2.png)



## 连接

```shell
./redis-cli -c -h 192.168.0.148 -p 6379
```



### 相关命令

```shell
cluster info
cluster nodes
```





### 使用docker连接

```
docker exec -it redis1 redis-cli -p 6379
```



### 关闭开启redis集群

三台虚拟机分别执行

关闭

```
docker stop redis1 redis2 
```


开启

```
docker start redis1 redis2 
```





# spring-data-redis中配置redis集群

## spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">


    <!-- 配置redis连接池对象 -->
    <bean id="poolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <!-- 最大空闲数 -->
        <property name="maxIdle" value="50" />
        <!-- 最大连接数 -->
        <property name="maxTotal" value="100" />
        <!-- 最大等待时间 -->
        <property name="maxWaitMillis" value="20000" />
    </bean>

    <!-- Redis集群配置 -->
    <bean id="redisClusterConfiguration"
          class="org.springframework.data.redis.connection.RedisClusterConfiguration">
        <property name="clusterNodes">
            <set>
                <bean class="org.springframework.data.redis.connection.RedisNode">
                    <constructor-arg name="host" value="192.168.2.148"/>
                    <constructor-arg name="port" value="6379"/>
                </bean>
                <bean class="org.springframework.data.redis.connection.RedisNode">
                    <constructor-arg name="host" value="192.168.2.148"/>
                    <constructor-arg name="port" value="6380"/>
                </bean>
                <bean class="org.springframework.data.redis.connection.RedisNode">
                    <constructor-arg name="host" value="192.168.2.149"/>
                    <constructor-arg name="port" value="6379"/>
                </bean>
                <bean class="org.springframework.data.redis.connection.RedisNode">
                    <constructor-arg name="host" value="192.168.2.149"/>
                    <constructor-arg name="port" value="6380"/>
                </bean>
                <bean class="org.springframework.data.redis.connection.RedisNode">
                    <constructor-arg name="host" value="192.168.2.150"/>
                    <constructor-arg name="port" value="6379"/>
                </bean>
                <bean class="org.springframework.data.redis.connection.RedisNode">
                    <constructor-arg name="host" value="192.168.2.150"/>
                    <constructor-arg name="port" value="6380"/>
                </bean>
            </set>
        </property>
    </bean>


    <!-- 配置redis连接工厂 -->
    <bean id="connectionFactory"
          class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
        <!--&lt;!&ndash; 配置哨兵 &ndash;&gt;-->
        <!--<constructor-arg name="sentinelConfig" ref="sentinelConfig"/>-->

        <!--集群配置-->
        <constructor-arg name="clusterConfig" ref="redisClusterConfiguration"/>

        <!-- 连接池配置 -->
        <property name="poolConfig" ref="poolConfig" />

    </bean>

    <bean id="stringRedisSerializer" class="org.springframework.data.redis.serializer.StringRedisSerializer"></bean>


    <!-- 配置redis模板对象 -->
    <bean class="org.springframework.data.redis.core.RedisTemplate">
        <!-- 配置连接工厂 -->
        <property name="connectionFactory" ref="connectionFactory" />

        <property name="keySerializer" ref="stringRedisSerializer"/>

        <property name="valueSerializer" ref="stringRedisSerializer"/>

        <property name="hashKeySerializer" ref="stringRedisSerializer"/>

        <property name="hashValueSerializer" ref="stringRedisSerializer" />

    </bean>

  
</beans>

```





# springboot整合redis集群

```yaml
spring:
  redis:
    cluster:
      nodes: 192.168.2.148:6379,192.168.2.148:6380,192.168.2.149:6379,192.168.2.149:6380,192.168.2.150:6379,192.168.2.150:6380
    jedis:
      pool:
        max-active: 100
```





