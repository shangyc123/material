# SpringBoot整合单机Redis

## 引入依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```



## 配置文件 application.yml

```yaml
spring:
  redis:
    host: 192.168.2.147
    port: 6379
    password: java1902
    jedis:
      pool:
        max-active: 100
```

