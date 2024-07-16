---
title: springcloud sentinel
date: 2024-07-07
authors: [刘耀文]
categories:
  - 备忘笔记

---
# seata部署
## docker-compose db nacos 模式
### 准备数据库表
数据库表: https://github.com/apache/incubator-seata/tree/develop/script/server/db
#### 准备compose.yaml文件
<!-- more -->
```yaml
version: "3.1"
services:
  seata-server:
    image: seataio/seata-server:1.5.2
    ports:
      - "7091:7091"
      - "8091:8091"
    environment:
      - STORE_MODE=db
      # 以SEATA_IP作为host注册seata server
      - SEATA_IP=192.168.208.128
      - SEATA_PORT=8091
    volumes:
      - "/usr/share/zoneinfo/Asia/Shanghai:/etc/localtime"        #设置系统时区
      - "/usr/share/zoneinfo/Asia/Shanghai:/etc/timezone"  #设置时区
      # 假设我们通过docker cp命令把资源文件拷贝到相对路径`./seata-server/resources`中
      # docker cp 容器id:/seata-server/resources ./resource 这步不能少
      # 如有问题，请阅读上面的[注意事项]以及[使用自定义配置文件]
      - "./resources:/seata-server/resources"
    networks:
    # 外部db数据库网络，需要先启动mysql容器，可以单独布置，或者放在这里一起布置
      - hm-net
          
networks:
  hm-net:
    external: true # 来自外部
```
#### 准备application.yaml文件

https://github.com/apache/incubator-seata/blob/develop/server/src/main/resources/application.example.yml

**application 例子**
注意，将/seata-server/resources拷贝出来后需要将里面的application替换
```yaml
server:
  port: 7091

spring:
  application:
    name: seata-server
    
console:
  user:
    username: liuyaowen
    password: 123123

logging:
  config: classpath:logback-spring.xml
  file:
    path: ${log.home:${user.home}/logs/seata}
  extend:
    logstash-appender:
      destination: 127.0.0.1:4560
    kafka-appender:
      bootstrap-servers: 127.0.0.1:9092
      topic: logback_to_logstash

seata:
  config:
    # support: nacos 、 consul 、 apollo 、 zk  、 etcd3
    type: nacos
    nacos:
      server-addr: 192.168.208.128:8848
      namespace: 9194952e-02a9-4737-89c2-1f3dee3317f0
      group: SEATA_GROUP
      username:
      password:
      context-path:
      ##if use MSE Nacos with auth, mutex with username/password attribute
      #access-key:
      #secret-key:
      data-id: seataServer.properties
  registry:
    # support: nacos 、 eureka 、 redis 、 zk  、 consul 、 etcd3 、 sofa
    type: nacos
    preferred-networks: 30.240.*
    nacos:
      application: seata-server
      server-addr: 192.168.208.128:8848
      group: SEATA_GROUP
      namespace: 9194952e-02a9-4737-89c2-1f3dee3317f0
      cluster: default
      username:
      password:
      context-path:
      ##if use MSE Nacos with auth, mutex with username/password attribute
      #access-key:
      #secret-key:
  security:
    secretKey: SeataSecretKey0c382ef121d778043159209298fd40bf3850a017
    tokenValidityInMilliseconds: 1800000
    ignore:
      urls: /,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/api/v1/auth/login

  metrics:
    enabled: false
    registry-type: compact
    exporter-list: prometheus
    exporter-prometheus-port: 9898
  transport:
    rpc-tc-request-timeout: 15000
    enable-tc-server-batch-send-response: false
    shutdown:
      wait: 3
    thread-factory:
      boss-thread-prefix: NettyBoss
      worker-thread-prefix: NettyServerNIOWorker
      boss-thread-size: 1
```
之后还需要在nacos中配置好seata的配置
**nacos 配置例子**
```java
store.mode=db
#-----db-----
store.db.datasource=druid
store.db.dbType=mysql
# 需要根据mysql的版本调整driverClassName
# mysql8及以上版本对应的driver：com.mysql.cj.jdbc.Driver
# mysql8以下版本的driver：com.mysql.jdbc.Driver
store.db.driverClassName=com.mysql.cj.jdbc.Driver
store.db.url=jdbc:mysql://xxxx:xxxx/{数据库中seata的表名字}?useUnicode=true&characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false
store.db.user=root
store.db.password=mysql_lyw
# 数据库初始连接数
store.db.minConn=1
# 数据库最大连接数
store.db.maxConn=20
# 获取连接时最大等待时间 默认5000，单位毫秒
store.db.maxWait=5000
# 全局事务表名 默认global_table
store.db.globalTable=global_table
# 分支事务表名 默认branch_table
store.db.branchTable=branch_table
# 全局锁表名 默认lock_table
store.db.lockTable=lock_table
store.db.distributedLockTable=distributed_lock
# 查询全局事务一次的最大条数 默认100
store.db.queryLimit=100


# undo保留天数 默认7天,log_status=1（附录3）和未正常清理的undo
server.undo.logSaveDays=7
# undo清理线程间隔时间 默认86400000，单位毫秒
server.undo.logDeletePeriod=86400000
# 二阶段提交重试超时时长 单位ms,s,m,h,d,对应毫秒,秒,分,小时,天,默认毫秒。默认值-1表示无限重试
# 公式: timeout>=now-globalTransactionBeginTime,true表示超时则不再重试
# 注: 达到超时时间后将不会做任何重试,有数据不一致风险,除非业务自行可校准数据,否者慎用
server.maxCommitRetryTimeout=-1
# 二阶段回滚重试超时时长
server.maxRollbackRetryTimeout=-1
# 二阶段提交未完成状态全局事务重试提交线程间隔时间 默认1000，单位毫秒
server.recovery.committingRetryPeriod=1000
# 二阶段异步提交状态重试提交线程间隔时间 默认1000，单位毫秒
server.recovery.asynCommittingRetryPeriod=1000
# 二阶段回滚状态重试回滚线程间隔时间  默认1000，单位毫秒
server.recovery.rollbackingRetryPeriod=1000
# 超时状态检测重试线程间隔时间 默认1000，单位毫秒，检测出超时将全局事务置入回滚会话管理器
server.recovery.timeoutRetryPeriod=1000

console.user.username=liuyaowen
console.user.password=123123
```

### 拉起服务
```java

docker-compose -f docker-compose.yaml up

```

### 文件建构

```xml
├── docker-compose.yaml
├── resources
│   ├── application.yaml
│   ├── banner.txt
│   ├── io
│   ├── logback
│   ├── logback-spring.xml
│   ├── lua
│   ├── META-INF
│   ├── README.md
│   ├── README-zh.md
```

## 简单使用
### 引入id
```xml
当然，discovery,springcloud,springcloudAlibaba,都要提前准备
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

### 配置
```xml
seata:
    registry:
        type: nacos
        nacos:
            application: seata-server
            server-addr: 192.168.208.128:8848
            group: SEATA_GROUP
            namespace: 9194952e-02a9-4737-89c2-1f3dee3317f0
            username:
            password:
    tx-service-group: hmall
    service: 
        vgroup-mapping:
        <!-- 对应事务组的集群位置 -->
            hmall: "default"

```

### AT 模式

```sql
-- 注意此处0.7.0+ 增加字段 context
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```
