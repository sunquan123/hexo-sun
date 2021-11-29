---
layout: shoka
title: Seata笔记
date: 2021-11-03 11:34:39
tags: ['Seata']
---

## 分布式事务问题

分布式前

- 单机单库没这个问题

- 从 1:1 -> 1:N -> N:N
  
  单体应用被拆分成微服务应用，原来的三个模块被拆分成三个独立的应用，分别使用三个独立的数据源，业务操作需要调用三个服务来完成。此时每个服务内部的数据一致性由本地事务来保证， 但是全局的数据一致性问题没法保证。
  
  ![img](seata架构.png)

一句话：一次业务操作需要跨多个数据源或需要跨多个系统进行远程调用，就会产生分布式事务问题。

## Seata简介

### 是什么？

Seata 是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。

[官方地址](http://seata.io/zh-cn/index.html)

### 1 XID+3 组件模型

##### TC (Transaction Coordinator) - 事务协调者

维护全局和分支事务的状态，驱动全局事务提交或回滚。

##### TM (Transaction Manager) - 事务管理器

定义全局事务的范围：开始全局事务、提交或回滚全局事务。

##### RM (Resource Manager) - 资源管理器

管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

##### XID(Transaction ID) - 全局事务ID

### 处理过程

1. TM 向 TC 申请开启一个全局事务，全局事务创建成功并生成一个全局唯一的 XID； 
2. XID 在微服务调用链路的上下文中传播； 
3. RM 向 TC 注册分支事务，将其纳入 XID 对应全局事务的管辖； 
4. TM 向 TC 发起针对 XID 的全局提交或回滚决议； 
5. TC 调度 XID 下管辖的全部分支事务完成提交或回滚请求。

![img](seata模型流程.png)

## Seata实战

### 安装Seata server

下载地址：https://github.com/seata/seata/releases

下载版本 - 1.4.2

将seata-server-1.4.2.tar.gz解压到指定目录并修改 conf 目录下的 file.conf 配置文件

将原始 file.conf 文件备份成 file.conf.bak

#### 修改Seata配置文件内容

store模块：修改mode为db模式，并增加数据库的相关配置

```yaml
store {
  ## store mode: file、db、redis
  mode = "db"
  ## rsa decryption public key
  publicKey = ""
  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    maxBranchSessionSize = 16384
    # globe session size , if exceeded throws exceptions
    maxGlobalSessionSize = 512
    # file buffer size , if exceeded allocate new buffer
    fileWriteBufferCacheSize = 16384
    # when recover batch read size
    sessionReloadReadSize = 100
    # async, sync
    flushDiskMode = async
  }

    ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
    datasource = "dbcp"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "mysql"
    driverClassName = "com.mysql.cj.jdbc.Driver"
    ## if using mysql to store the data, recommend add rewriteBatchedStatements=true in jdbc connection param
    url = "jdbc:mysql://127.0.0.1:3306/seata?rewriteBatchedStatements=true"
    user = "root"
    password = "123456"
    minConn = 5
    maxConn = 100
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }
}
```

### 建立Seata数据库

mysql数据库新建库 seata，在 seata 库里建表

```sql
-- the table to store GlobalSession data
drop table if exists `global_table`;
create table `global_table` (
  `xid` varchar(128)  not null,
  `transaction_id` bigint,
  `status` tinyint not null,
  `application_id` varchar(32),
  `transaction_service_group` varchar(32),
  `transaction_name` varchar(128),
  `timeout` int,
  `begin_time` bigint,
  `application_data` varchar(2000),
  `gmt_create` datetime,
  `gmt_modified` datetime,
  primary key (`xid`),
  key `idx_gmt_modified_status` (`gmt_modified`, `status`),
  key `idx_transaction_id` (`transaction_id`)
);

-- the table to store BranchSession data
drop table if exists `branch_table`;
create table `branch_table` (
  `branch_id` bigint not null,
  `xid` varchar(128) not null,
  `transaction_id` bigint ,
  `resource_group_id` varchar(32),
  `resource_id` varchar(256) ,
  `lock_key` varchar(128) ,
  `branch_type` varchar(8) ,
  `status` tinyint,
  `client_id` varchar(64),
  `application_data` varchar(2000),
  `gmt_create` datetime,
  `gmt_modified` datetime,
  primary key (`branch_id`),
  key `idx_xid` (`xid`)
);

-- the table to store lock data
drop table if exists `lock_table`;
create table `lock_table` (
  `row_key` varchar(128) not null,
  `xid` varchar(96),
  `transaction_id` long ,
  `branch_id` long,
  `resource_id` varchar(256) ,
  `table_name` varchar(32) ,
  `pk` varchar(36) ,
  `gmt_create` datetime ,
  `gmt_modified` datetime,
  primary key(`row_key`)
);
```

### 连接Nacos

修改conf 目录下的 registry.conf 配置文件

```yaml
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  # 设置注册中心为nacos
  type = "nacos"

  nacos {
    ## 设置nacos服务地址
    serverAddr = "localhost:8848"
    namespace = ""
    cluster = "default"
  }
}
```

### 建立Seata业务数据库

#### 业务流程

浏览器模拟用户下单操作：

1. 调用订单服务增加订单记录
2. 调用库存服务减少相应库存
3. 调用账号服务减少账号存款
4. 调用订单服务调整订单状态为已支付

#### 业务数据库

- seata_order：存储订单的数据库
- seata_storage：存储库存的数据库
- seata_account：存储账户信息的数据库

#### 建库语句

```sql
CREATE DATABASE seata_order;
CREATE DATABASE seata_storage;
CREATE DATABASE seata_account;
```

#### 建表语句

seata_order 库下建 t_order 表

```sql
CREATE TABLE t_order (
    `id` BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `user_id` BIGINT(11) DEFAULT NULL COMMENT '用户id',
    `product_id` BIGINT(11) DEFAULT NULL COMMENT '产品id',
    `count` INT(11) DEFAULT NULL COMMENT '数量',
    `money` DECIMAL(11,0) DEFAULT NULL COMMENT'金额',
    `status` INT(1) DEFAULT NULL COMMENT '订单状态: 0:创建中; 1:已完结'
) ENGINE=INNODB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;
```

seata_storage 库下建 t_storage 表

```sql
CREATE TABLE t_storage (
        `id` BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
        `product_id` BIGINT(11) DEFAULT NULL COMMENT '产品id',
        `total` INT(11) DEFAULT NULL COMMENT '总库存',
        `used` INT(11) DEFAULT NULL COMMENT '已用库存',
        `residue` INT(11) DEFAULT NULL COMMENT '剩余库存'
) ENGINE=INNODB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

# 初始化一条产品的库存记录
INSERT INTO seata_storage.t_storage(`id`, `product_id`, `total`, `used`, `residue`)
VALUES ('1', '1', '100', '0','100');
```

seata_account 库下建 t_account 表

```sql
CREATE TABLE t_account(
    `id` BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY COMMENT 'id',
    `user_id` BIGINT(11) DEFAULT NULL COMMENT '用户id',
    `total` DECIMAL(10,0) DEFAULT NULL COMMENT '总额度',
    `used` DECIMAL(10,0) DEFAULT NULL COMMENT '已用余额',
    `residue` DECIMAL(10,0) DEFAULT '0' COMMENT '剩余可用额度'
) ENGINE=INNODB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

# 初始化一条用户记录
INSERT INTO seata_account.t_account(`id`, `user_id`, `total`, `used`, `residue`)
VALUES ('1', '1', '1000', '0', '1000');
```

seata_order、seata_storage、seata_account库下建各自的回滚日志表

```sql
drop table `undo_log`;
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```

### Order-Module 模块搭建

模块名：seata-order-service2001

职责：提供订单服务，下订单 -> 减库存 -> 扣余额 -> 改订单状态

#### 配置部分

POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <parent>
    <artifactId>cloud2020</artifactId>
    <groupId>com.atguigu.springcloud</groupId>
    <version>1.0-SNAPSHOT</version>
  </parent>
  <modelVersion>4.0.0</modelVersion>

  <artifactId>seata-order-service2001</artifactId>

  <properties>
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
  </properties>

  <dependencies>

    <!--nacos-->
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!--seata-->
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
      <exclusions>
        <exclusion>
          <artifactId>seata-all</artifactId>
          <groupId>io.seata</groupId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>io.seata</groupId>
      <artifactId>seata-all</artifactId>
      <version>0.9.0</version>
    </dependency>

    <!--feign-->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>

    <!--web-actuator-->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!--mysql-druid-->
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
<!--      <version>5.1.37</version>-->
    </dependency>

    <dependency>
      <groupId>com.alibaba</groupId>
      <artifactId>druid-spring-boot-starter</artifactId>
      <version>1.1.10</version>
    </dependency>

    <dependency>
      <groupId>org.mybatis.spring.boot</groupId>
      <artifactId>mybatis-spring-boot-starter</artifactId>
      <version>2.0.0</version>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>

    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <optional>true</optional>
    </dependency>

    <dependency>
      <groupId>com.atguigu.springcloud</groupId>
      <artifactId>cloudapicommons</artifactId>
      <version>${project.version}</version>
    </dependency>

  </dependencies>
</project>
```

application.yml

```yaml
server:
  port: 2001

spring:
  application:
    name: seata-order-service
  cloud:
    alibaba:
      seata:
        #自定义事务组名称需要与 seata-server 中的对应
        tx-service-group: fsp_tx_group
    nacos:
      discovery:
        server-addr: localhost:15003
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/seata_order
    username: root
    password: 123456

feign:
  hystrix:
    enabled: false

logging:
  level:
    io:
      seata: info

mybatis:
  mapperLocations: classpath:mapper/*.xml
```

file.conf

```yaml
transport {
  # tcp udt unix-domain-socket
  type = "TCP"
  #NIO NATIVE
  server = "NIO"
  #enable heartbeat
  heartbeat = true
  #thread factory for netty
  thread-factory {
    boss-thread-prefix = "NettyBoss"
    worker-thread-prefix = "NettyServerNIOWorker"
    server-executor-thread-prefix = "NettyServerBizHandler"
    share-boss-worker = false
    client-selector-thread-prefix = "NettyClientSelector"
    client-selector-thread-size = 1
    client-worker-thread-prefix = "NettyClientWorkerThread"
    # netty boss thread size,will not be used for UDT
    boss-thread-size = 1
    #auto default pin or 8
    worker-thread-size = 8
  }
  shutdown {
    # when destroy server, wait seconds
    wait = 3
  }
  serialization = "seata"
  compressor = "none"
}

service {

  vgroup_mapping.fsp_tx_group = "default" #修改自定义事务组名称

  default.grouplist = "127.0.0.1:8091"
  enableDegrade = false
  disable = false
  max.commit.retry.timeout = "-1"
  max.rollback.retry.timeout = "-1"
  disableGlobalTransaction = false
}


client {
  async.commit.buffer.limit = 10000
  lock {
    retry.internal = 10
    retry.times = 30
  }
  report.retry.count = 5
  tm.commit.retry.count = 1
  tm.rollback.retry.count = 1
}

## transaction log store
store {
  ## store mode: file、db
  mode = "db"

  ## file store
  file {
    dir = "sessionStore"

    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    max-branch-session-size = 16384
    # globe session size , if exceeded throws exceptions
    max-global-session-size = 512
    # file buffer size , if exceeded allocate new buffer
    file-write-buffer-cache-size = 16384
    # when recover batch read size
    session.reload.read_size = 100
    # async, sync
    flush-disk-mode = async
  }

  ## database store
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp) etc.
    datasource = "dbcp"
    ## mysql/oracle/h2/oceanbase etc.
    db-type = "mysql"
    driver-class-name = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/seata"
    user = "root"
    password = "123456"
    min-conn = 1
    max-conn = 3
    global.table = "global_table"
    branch.table = "branch_table"
    lock-table = "lock_table"
    query-limit = 100
  }
}
lock {
  ## the lock store mode: local、remote
  mode = "remote"

  local {
    ## store locks in user's database
  }

  remote {
    ## store locks in the seata's server
  }
}
recovery {
  #schedule committing retry period in milliseconds
  committing-retry-period = 1000
  #schedule asyn committing retry period in milliseconds
  asyn-committing-retry-period = 1000
  #schedule rollbacking retry period in milliseconds
  rollbacking-retry-period = 1000
  #schedule timeout retry period in milliseconds
  timeout-retry-period = 1000
}

transaction {
  undo.data.validation = true
  undo.log.serialization = "jackson"
  undo.log.save.days = 7
  #schedule delete expired undo_log in milliseconds
  undo.log.delete.period = 86400000
  undo.log.table = "undo_log"
}

## metrics settings
metrics {
  enabled = false
  registry-type = "compact"
  # multi exporters use comma divided
  exporter-list = "prometheus"
  exporter-prometheus-port = 9898
}

support {
  ## spring
  spring {
    # auto proxy the DataSource bean
    datasource.autoproxy = false
  }
}
```

registry.conf

```yaml
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    serverAddr = "localhost:15003"
    namespace = ""
    cluster = "default"
  }
  eureka {
    serviceUrl = "http://localhost:8761/eureka"
    application = "default"
    weight = "1"
  }
  redis {
    serverAddr = "localhost:6379"
    db = "0"
  }
  zk {
    cluster = "default"
    serverAddr = "127.0.0.1:2181"
    session.timeout = 6000
    connect.timeout = 2000
  }
  consul {
    cluster = "default"
    serverAddr = "127.0.0.1:8500"
  }
  etcd3 {
    cluster = "default"
    serverAddr = "http://localhost:2379"
  }
  sofa {
    serverAddr = "127.0.0.1:9603"
    application = "default"
    region = "DEFAULT_ZONE"
    datacenter = "DefaultDataCenter"
    cluster = "default"
    group = "SEATA_GROUP"
    addressWaitTime = "3000"
  }
  file {
    name = "file.conf"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file"

  nacos {
    serverAddr = "localhost"
    namespace = ""
  }
  consul {
    serverAddr = "127.0.0.1:8500"
  }
  apollo {
    app.id = "seata-server"
    apollo.meta = "http://192.168.1.204:8801"
  }
  zk {
    serverAddr = "127.0.0.1:2181"
    session.timeout = 6000
    connect.timeout = 2000
  }
  etcd3 {
    serverAddr = "http://localhost:2379"
  }
  file {
    name = "file.conf"
  }
}
```

#### 代码部分

##### 主启动

SeataOrderMainApp2001

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@EnableDiscoveryClient
@EnableFeignClients
// 取消数据源的自动创建，而是使用自己定义的
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class SeataOrderMainApp2001 {

  public static void main(String[] args) {
    SpringApplication.run(SeataOrderMainApp2001.class, args);
  }
}
```

##### Domain层

Order

```java
import java.math.BigDecimal;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class Order {
  private Long id;
  private Long userId;
  private Long productId;
  private Integer count;
  private BigDecimal money;
  private Integer status; // 订单状态：0：创建中；1：已完结
}
```

OrderDao

```java
import com.atguigu.springcloud.alibaba.domain.Order;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

@Mapper
public interface OrderDao {
  // 1 新建订单
  void create(Order order);
  // 2 修改订单状态，从零改为 1
  void update(@Param("userId") Long userId, @Param("status") Integer status);
}
```

OrderMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.atguigu.springcloud.alibaba.dao.OrderDao">
  <resultMap id="BaseResultMap" type="com.atguigu.springcloud.alibaba.domain.Order">
    <id column="id" property="id" jdbcType="BIGINT"/>
    <result column="user_id" property="userId" jdbcType="BIGINT"/>
    <result column="product_id" property="productId" jdbcType="BIGINT"/>
    <result column="count" property="count" jdbcType="INTEGER"/>
    <result column="money" property="money" jdbcType="DECIMAL"/>
    <result column="status" property="status" jdbcType="INTEGER"/>
  </resultMap>
  <insert id="create">
    insert into t_order (id,user_id,product_id,count,money,status)
    values (null,#{userId},#{productId},#{count},#{money},0);
  </insert>
  <update id="update">
    update t_order set status = 1
    where user_id=#{userId} and status = #{status};
  </update>
</mapper>
```

##### Service层

OrderService

```java
import com.atguigu.springcloud.alibaba.domain.Order;

public interface OrderService {
  void create(Order order);
}
```

AccountService

```java
import com.atguigu.springcloud.alibaba.domain.CommonResult;
import java.math.BigDecimal;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(value = "seata-account-service")
public interface AccountService {
  @PostMapping(value = "/account/decrease")
  CommonResult decrease(
      @RequestParam("userId") Long userId, @RequestParam("money") BigDecimal money);
}
```

StorageService

```java
import com.atguigu.springcloud.alibaba.domain.CommonResult;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(value = "seata-storage-service")
public interface StorageService {
  @PostMapping(value = "/storage/decrease")
  CommonResult decrease(
      @RequestParam("productId") Long productId, @RequestParam("count") Integer count);
}
```

OrderServiceImpl

```java
import com.atguigu.springcloud.alibaba.dao.OrderDao;
import com.atguigu.springcloud.alibaba.domain.Order;
import io.seata.spring.annotation.GlobalTransactional;
import javax.annotation.Resource;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class OrderServiceImpl implements OrderService {
  @Resource private OrderDao orderDao;
  @Resource private StorageService storageService;
  @Resource private AccountService accountService;

  /** 创建订单 -> 调用库存服务扣减库存 -> 调用账户服务扣减账户余额 -> 修改订单状态 简单说：下订单 -> 扣库存 -> 减余额 -> 改状态 */
  @Override
  public void create(Order order) {
    log.info("----->开始新建订单");
    // 1 新建订单
    orderDao.create(order);
    // 2 扣减库存
    log.info("----->订单微服务开始调用库存，做扣减Count");
    storageService.decrease(order.getProductId(), order.getCount());
    log.info("----->订单微服务开始调用库存，做扣减end");
    // 3 扣减账户
    log.info("----->订单微服务开始调用账户，做扣减Money");
    accountService.decrease(order.getUserId(), order.getMoney());
    log.info("----->订单微服务开始调用账户，做扣减end");
    // 4 修改订单状态，从零到 1,1 代表已经完成
    log.info("----->修改订单状态开始");
    orderDao.update(order.getUserId(), 0);
    log.info("----->修改订单状态结束");
    log.info("----->下订单结束了，O(∩_∩)O哈哈~");
  }
}
```

##### Config层

DataSourceProxyConfig

```java
import com.alibaba.druid.pool.DruidDataSource;
import io.seata.rm.datasource.DataSourceProxy;
import javax.sql.DataSource;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.transaction.SpringManagedTransactionFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
/** 使用 Seata 对数据源进行代理 */
@Configuration
public class DataSourceProxyConfig {
  @Value("${mybatis.mapperLocations}")
  private String mapperLocations;

  @Bean
  @ConfigurationProperties(prefix = "spring.datasource")
  public DataSource druidDataSource() {
    return new DruidDataSource();
  }

  @Bean
  public DataSourceProxy dataSourceProxy(DataSource dataSource) {
    return new DataSourceProxy(dataSource);
  }

  @Bean
  public SqlSessionFactory sqlSessionFactoryBean(DataSourceProxy dataSourceProxy) throws Exception {
    SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
    sqlSessionFactoryBean.setDataSource(dataSourceProxy);
    sqlSessionFactoryBean.setMapperLocations(
        new PathMatchingResourcePatternResolver().getResources(mapperLocations));
    sqlSessionFactoryBean.setTransactionFactory(new SpringManagedTransactionFactory());
    return sqlSessionFactoryBean.getObject();
  }
}
```

MyBatisConfig

```java
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@MapperScan({"com.atguigu.springcloud.alibaba.dao"})
public class MyBatisConfig {}
```

##### Controller层

OrderController

```java
import com.atguigu.springcloud.alibaba.domain.CommonResult;
import com.atguigu.springcloud.alibaba.domain.Order;
import com.atguigu.springcloud.alibaba.service.OrderService;
import javax.annotation.Resource;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class OrderController {
  @Resource private OrderService orderService;

  @GetMapping("/order/create")
  public CommonResult create(Order order) {
    orderService.create(order);
    return new CommonResult(200, "订单创建成功");
  }
}
```

### Storage-Module 模块搭建

模块名：seata-Storage-service2002 

职责：提供库存服务

#### 配置部分

POM

（与 seata-order-service2001 模块大致相同）

application.yml

```yaml
server:
  port: 2002

spring:
  application:
    name: seata-storage-service
  cloud:
    alibaba:
      seata:
        #自定义事务组名称需要与 seata-server 中的对应
        tx-service-group: fsp_tx_group
    nacos:
      discovery:
        server-addr: localhost:15003
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/seata_storage
    username: root
    password: 123456

feign:
  hystrix:
    enabled: false

logging:
  level:
    io:
      seata: info

mybatis:
  mapperLocations: classpath:mapper/*.xml
```

file.conf

（与 seata-order-service2001 模块大致相同）

registry.conf

（与 seata-order-service2001 模块大致相同）

#### 代码部分

##### 主启动

SeataStorageMainApp2002

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@EnableDiscoveryClient
@EnableFeignClients
// 取消数据源的自动创建，而是使用自己定义的
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class SeataStorageMainApp2002 {

  public static void main(String[] args) {
    SpringApplication.run(SeataStorageMainApp2002.class, args);
  }
}
```

##### Domain层

Storage

```java
import lombok.Data;

@Data
public class Storage {

  private Long id;

  /** 产品 id */
  private Long productId;

  /** 总库存 */
  private Integer total;

  /** 已用库存 */
  private Integer used;

  /** 剩余库存 */
  private Integer residue;
}
```

StorageDao

```java
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

@Mapper
public interface StorageDao {
  // 扣减库存
  void decrease(@Param("productId") Long productId, @Param("count") Integer count);
}
```

StorageMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.atguigu.springcloud.alibaba.dao.StorageDao">
  <resultMap id="BaseResultMap" type="com.atguigu.springcloud.alibaba.domain.Storage">
    <id column="id" property="id" jdbcType="BIGINT"/>
    <result column="product_id" property="productId" jdbcType="BIGINT"/>
    <result column="total" property="total" jdbcType="INTEGER"/>
    <result column="used" property="used" jdbcType="INTEGER"/>
    <result column="residue" property="residue" jdbcType="INTEGER"/>
  </resultMap>
  <update id="decrease">
    UPDATE
    t_storage
    SET
    used = used + #{count},residue = residue - #{count}
    WHERE
    product_id = #{productId}
  </update>
</mapper>
```

##### Service层

StorageService

```java
public interface StorageService {
  /** 扣减库存 */
  void decrease(Long productId, Integer count);
}
```

StorageServiceImpl

```java
import com.atguigu.springcloud.alibaba.dao.StorageDao;
import javax.annotation.Resource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class StorageServiceImpl implements StorageService {
  private static final Logger LOGGER = LoggerFactory.getLogger(StorageServiceImpl.class);
  @Resource private StorageDao storageDao;
  /** 扣减库存 */
  @Override
  public void decrease(Long productId, Integer count) {
    LOGGER.info("------->storage-service中扣减库存开始");
    storageDao.decrease(productId, count);
    LOGGER.info("------->storage-service中扣减库存结束");
  }
}
```

##### Config层

（与 seata-order-service2001 模块大致相同）

##### Controller层

StorageController

```java
import com.atguigu.springcloud.alibaba.domain.CommonResult;
import com.atguigu.springcloud.alibaba.service.StorageService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class StorageController {

  @Autowired private StorageService storageService;

  /** 扣减库存 */
  @RequestMapping("/storage/decrease")
  public CommonResult decrease(Long productId, Integer count) {
    storageService.decrease(productId, count);
    return new CommonResult(200, "扣减库存成功！");
  }
}
```

### Account-Module 模块搭建

模块名：seata-Account-service2003 

职责：提供账号服务

#### 配置部分

POM

（与 seata-order-service2001 模块大致相同）

application.yml

```yaml
server:
  port: 2003

spring:
  application:
    name: seata-account-service
  cloud:
    alibaba:
      seata:
        #自定义事务组名称需要与 seata-server 中的对应
        tx-service-group: fsp_tx_group
    nacos:
      discovery:
        server-addr: localhost:15003
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/seata_account
    username: root
    password: 123456

feign:
  hystrix:
    enabled: false

logging:
  level:
    io:
      seata: info

mybatis:
  mapperLocations: classpath:mapper/*.xml
```

file.conf

（与 seata-order-service2001 模块大致相同）

registry.conf

（与 seata-order-service2001 模块大致相同）

#### 代码部分

##### 主启动

SeataAccountMainApp2003

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@EnableDiscoveryClient
@EnableFeignClients
// 取消数据源的自动创建，而是使用自己定义的
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class SeataAccountMainApp2003 {

  public static void main(String[] args) {
    SpringApplication.run(SeataAccountMainApp2003.class, args);
  }
}
```

##### Domain层

Account

```java
import java.math.BigDecimal;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class Account {

  private Long id;

  /** 用户 id */
  private Long userId;

  /** 总额度 */
  private BigDecimal total;

  /** 已用额度 */
  private BigDecimal used;

  /** 剩余额度 */
  private BigDecimal residue;
}
```

AccountDao

```java
import java.math.BigDecimal;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;

@Mapper
public interface AccountDao {
  /** 扣减账户余额 */
  void decrease(@Param("userId") Long userId, @Param("money") BigDecimal money);
}
```

AccountMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
  <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.atguigu.springcloud.alibaba.dao.AccountDao">
  <resultMap id="BaseResultMap" type="com.atguigu.springcloud.alibaba.domain.Account">
    <id column="id" property="id" jdbcType="BIGINT"/>
    <result column="user_id" property="userId" jdbcType="BIGINT"/>
    <result column="total" property="total" jdbcType="DECIMAL"/>
    <result column="used" property="used" jdbcType="DECIMAL"/>
    <result column="residue" property="residue" jdbcType="DECIMAL"/>
  </resultMap>
  <update id="decrease">
    UPDATE t_account
    SET
    residue = residue - #{money},used = used + #{money}
    WHERE
    user_id = #{userId};
  </update>
</mapper>
```

##### Service层

AccountService

```java
import java.math.BigDecimal;
import org.springframework.web.bind.annotation.RequestParam;

public interface AccountService {
  /**
   * 扣减账户余额
   *
   * @param userId 用户 id
   * @param money 金额
   */
  void decrease(@RequestParam("userId") Long userId, @RequestParam("money") BigDecimal money);
}
```

AccountServiceImpl

```java
import com.atguigu.springcloud.alibaba.dao.AccountDao;
import java.math.BigDecimal;
import java.util.concurrent.TimeUnit;
import javax.annotation.Resource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
/** */
@Service
public class AccountServiceImpl implements AccountService {
  private static final Logger LOGGER = LoggerFactory.getLogger(AccountServiceImpl.class);
  @Resource AccountDao accountDao;

  /** 扣减账户余额 */
  @Override
  public void decrease(Long userId, BigDecimal money) {
    LOGGER.info("------->account-service中扣减账户余额开始");
    accountDao.decrease(userId, money);
    LOGGER.info("------->account-service中扣减账户余额结束");
  }
}
```

##### Config层

（与 seata-order-service2001 模块大致相同）

##### Controller层

AccountController

```java
import com.atguigu.springcloud.alibaba.domain.CommonResult;
import com.atguigu.springcloud.alibaba.service.AccountService;
import java.math.BigDecimal;
import javax.annotation.Resource;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AccountController {

  @Resource AccountService accountService;

  /** 扣减账户余额 */
  @RequestMapping("/account/decrease")
  public CommonResult decrease(Long userId, BigDecimal money) {
    accountService.decrease(userId, money);
    return new CommonResult(200, "扣减账户余额成功！");
  }
}
```

### @GlobalTransactional 使用

正常下单

```http
http://localhost:2001/order/create?userld=1&productld=1&count=10&money=100
```

正常情况下，下订单后会下订单、减库存、扣余额、改订单状态

异常情况下，任何一个环节出错没有对应的回退机制，会导致中间任何一项数据出现脏数据

#### 模拟 AccountServiceImpl 添加超时

OpenFeign 的调用默认时间是 1s 以内，所以最后会抛异常

```java
@Service
public class AccountServiceImpl implements AccountService {
  private static final Logger LOGGER = LoggerFactory.getLogger(AccountServiceImpl.class);
  @Resource AccountDao accountDao;

  /** 扣减账户余额 */
  @Override
  public void decrease(Long userId, BigDecimal money) {
    LOGGER.info("------->account-service中扣减账户余额开始");
    // 模拟超时异常，全局事务回滚
    // 暂停几秒钟线程
    try {
      TimeUnit.SECONDS.sleep(20);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    accountDao.decrease(userId, money);
    LOGGER.info("------->account-service中扣减账户余额结束");
  }
}
```

数据库情况

![img](未加全局事务的数据库情况.png)

故障情况

- 当库存和账户金额扣减后，订单状态并没有设置为已经完成，没有从零改为 1
- 而且由于 feign 的重试机制，账户余额还有可能被多次扣减

#### @GlobalTransactional开启全局事务

OrderServiceImpl

```java
import com.atguigu.springcloud.alibaba.dao.OrderDao;
import com.atguigu.springcloud.alibaba.domain.Order;
import io.seata.spring.annotation.GlobalTransactional;
import javax.annotation.Resource;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@Slf4j
public class OrderServiceImpl implements OrderService {
  @Resource private OrderDao orderDao;
  @Resource private StorageService storageService;
  @Resource private AccountService accountService;

  /** 创建订单 -> 调用库存服务扣减库存 -> 调用账户服务扣减账户余额 -> 修改订单状态 简单说：下订单 -> 扣库存 -> 减余额 -> 改状态 */
  @Override
  // rollbackFor = Exception.class 表示对任意异常都进行回滚
  @GlobalTransactional(name = "fsp-create-order", rollbackFor = Exception.class)
  public void create(Order order) {
    log.info("----->开始新建订单");
    // 1 新建订单
    orderDao.create(order);
    // 2 扣减库存
    log.info("----->订单微服务开始调用库存，做扣减Count");
    storageService.decrease(order.getProductId(), order.getCount());
    log.info("----->订单微服务开始调用库存，做扣减end");
    // 3 扣减账户
    log.info("----->订单微服务开始调用账户，做扣减Money");
    accountService.decrease(order.getUserId(), order.getMoney());
    log.info("----->订单微服务开始调用账户，做扣减end");
    // 4 修改订单状态，从零到 1,1 代表已经完成
    log.info("----->修改订单状态开始");
    orderDao.update(order.getUserId(), 0);
    log.info("----->修改订单状态结束");
    log.info("----->下订单结束了，O(∩_∩)O哈哈~");
  }
}
```

再次模拟 AccountServiceImpl 添加超时，下单后三个数据库数据都没有改变，达到了分支事务异常，全局回滚的效果

# Seata 之全局事务原理

### 事务二阶段执行流程

- 一阶段加载：
  
  解析业务sql，找到业务sql要更新的业务数据，生成 before image
  
  执行业务sql，更新业务数据
  
  业务数据更新后，保存新的业务数据为 after image ，生成行锁

- 二阶段提交：
  
  如果全部事务顺利提交，Seata框架将  before image、after image、行锁全部删除

- 二阶段回滚：
  
  如果需要回滚，Seata将会用 before image 生成 逆向sql 还原业务数据，但需要先检查脏写，对比数据库当前业务数据和 after image，如果一致，则没有脏写，可以回滚；如果不一致，有脏写，则需要转人工处理。

![img](事务二阶段执行流程.png)
