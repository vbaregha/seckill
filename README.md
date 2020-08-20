# seckill 

* [1 秒杀系统业务分析](#1-秒杀系统业务分析)
* [2 开发环境](#2-开发环境)
* [3 工程创建](#3-工程创建)
* [4 业务实现](#4-业务实现)
  * [4.1 数据库建表](#4.1-数据库建表)
  * [4.2 DAO实体和接口开发](#4.2-DAO实体和接口开发)
  * [4.3 Service层开发](#4.3-Service层开发)
  * [4.4 Controller层开发](#4.4-Controller层开发)
* [5 并发优化](#5-并发优化)

-----------------------------

## 1 秒杀系统业务分析

首先分析，秒杀系统问题的本质其实是对商品库存的管理。主要业务逻辑如下图：

<div align="center"><img src="pics//1553423680(1).png" width="500px"></div>
用户针对库存业务分析：

<div align="center"><img src="pics//1553423931(1).png" width="500px"></div>
用户的购买行为：

<div align="center"><img src="pics//1553424057(1).png" width="500px"></div>
**难点**：如何高效地处理“竞争”。

## 2 开发环境

InteliJ IDEA + Maven + Tomcat8 + JDK8

## 3 工程创建

新建一个 Maven 工程，并完善相应的目录结构。

pom 文件的依赖可以分为 4 部分：

- **日志**。使用的是 slf4j + logback 的组合。
- **数据库**。数据库连接池 + DAO框架，MyBatis依赖。
- **Servlet web**。jsp 等。
- **Spring**。主要是 Spring 相关的依赖。

全部依赖参考 [pom.xml](https://github.com/MinheZ/seckill/blob/master/pom.xml)。

## 4 业务实现

### 4.1 数据库建表

- 秒杀库存表
- 秒杀成功明细表

### 4.2 DAO实体和接口开发

首先是实体类 entity 的编写，分为 [Seckill](https://github.com/MinheZ/seckill/blob/master/src/main/java/com/seckill/entity/Seckill.java) 和 [SuccessKilled](https://github.com/MinheZ/seckill/blob/master/src/main/java/com/seckill/entity/SuccessKilled.java) 。

[SeckillDao](https://github.com/MinheZ/seckill/blob/master/src/main/java/com/seckill/dao/SeckillDao.java) 和 [SuccessKilledDao](https://github.com/MinheZ/seckill/blob/master/src/main/java/com/seckill/dao/SuccessKilledDao.java) 接口为查询数据库，或者修改数据库的一些方法。

剩下的就是一些 MyBatis 整合 Spring 的配置文件编写。

### 4.3 Service层开发

 [SeckillService](https://github.com/MinheZ/seckill/blob/master/src/main/java/com/seckill/service/SeckillService.java) 接口开发，完成秒杀业务逻辑的一些方法。

```java
List<Seckill> getSeckillList();	// 获取秒杀列表

Seckill getById(long seckillId);	// 通过 ID 获取秒杀对象

Exposer exportSeckillUrl(long seckillId);	// 判断是否需要暴露秒杀接口

// 执行秒杀操作
SeckillExecution executeSeckill(long seckillId, long userPhone, String md5)
            throws SeckillException, RepeatKillException, SeckillCloseException;
// 秒杀存储过程优化
SeckillExecution executeSeckillProcedure(long seckillId, long userPhone, String md5);
```

业务逻辑为，当秒杀开始前，秒杀页面只显示秒杀商品类型和倒计时。只有当秒杀开始的时候，才暴露秒杀地址，防止脚本提前登录。

定义一个 [Exposer](https://github.com/MinheZ/seckill/blob/master/src/main/java/com/seckill/dto/Exposer.java) 类来实现此功能。主要是通过比较系统当前时间与秒杀开始和结束的时间。

定义一个 [SeckillExecution](https://github.com/MinheZ/seckill/blob/master/src/main/java/com/seckill/dto/SeckillExecution.java) 类来封装秒杀操作之后的结果。其中用枚举类型 [SeckillStatEnum ](https://github.com/MinheZ/seckill/blob/master/src/main/java/com/seckill/enums/SeckillStatEnum.java) 封装秒杀过程之后的各种状态，例如：成功、失败等。

具体的实现类 [SeckillServiceImpl](https://github.com/MinheZ/seckill/blob/master/src/main/java/com/seckill/service/impl/SeckillServiceImpl.java) 也就是对数据库的一些增删改查，包括对秒杀接口暴露，判断秒杀状态等一些列方法的具体实现。这里要注意 `executeSeckill` 方法的事务性，该操作必须是原子的。

### 4.4 Controller层开发

首先明确 SeckillController业务流程：

<div align="center"><img src="pics//1553561741(1).png" width="400px"></div>
详情页流程逻辑：

<div align="center"><img src="pics//1553561891(1).png" width="500px"></div>
一般来说，Controller 层的 URL 表达方式默认使用 Restful 规范：

- GET -> 查询操作
- POST -> 添加/修改操作
- PUT -> 修改操作（幂等操作）
- DELETE -> 删除操作

下图为秒杀 API 的 URL 设计：

<div align="center"><img src="pics//1553562354(1).png" width="500px"></div>

### 4.5 SSM框架整合原理以及思路

由于Spring MVC是Spring框架中的一个模块，所以整合只涉及两个模块整合.

- Spring和MyBatis整合
- Spring MVC 和MyBatis整合

1.  **在【web.xml】中配置 DispatcherServlet 并使spring-*的配置文件在Tomcat启动时自动加载：** 

```java
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                      http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1" metadata-complete="true">
<!-- 配置springMVC需要加载的配置文件
         spring-dao.xml,spring-service.xml,spring-web.xml
         Mybatis - > spring -> springmvc
-->
    <servlet>
        <servlet-name>seckill-dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring/spring-*.xml</param-value>
        </init-param>
    </servlet>
                
<!-- 拦截所有请求-->   
                
    <servlet-mapping>
        <servlet-name>seckill-dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

2.  **在【spring-mybatis.xml】中完成 spring 和 mybatis 的配置：** 

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context.xsd">
    <!-- 配置整合mybatis过程 -->
    <!-- 1.配置数据库相关参数properties的属性：${url} -->
    <context:property-placeholder location="classpath:jdbc.properties" />

    <!-- 2.数据库连接池 -->
    <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
        <!-- 配置连接池属性 -->
        <property name="driverClass" value="${jdbc.driver}" />
        <property name="jdbcUrl" value="${jdbc.url}" />
        <property name="user" value="${jdbc.username}" />
        <property name="password" value="${jdbc.password}" />

        <!-- c3p0连接池的私有属性 -->
        <property name="maxPoolSize" value="30" />
        <property name="minPoolSize" value="10" />
        <!-- 关闭连接后不自动commit -->
        <property name="autoCommitOnClose" value="false" />
        <!-- 获取连接超时时间 -->
        <property name="checkoutTimeout" value="10000" />
        <!-- 当获取连接失败重试次数 -->
        <property name="acquireRetryAttempts" value="2" />
    </bean>

    <!-- 3.配置SqlSessionFactory对象 -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <!-- 注入数据库连接池 -->
        <property name="dataSource" ref="dataSource" />
        <!-- 配置MyBaties全局配置文件:mybatis-config.xml -->
        <property name="configLocation" value="classpath:mybatis-config.xml" />
        <!-- 扫描entity包 使用别名 -->
        <property name="typeAliasesPackage" value="org.seckill.entity" />
        <!-- 扫描sql配置文件:mapper需要的xml文件 -->
        <property name="mapperLocations" value="classpath:mapper/*.xml" />
    </bean>

    <!-- 4.配置扫描Dao接口包，动态实现Dao接口，注入到soring容器中 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <!-- 注入sqlSessionFactory -->
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
        <!-- 给出需要扫描Dao接口包 -->
        <property name="basePackage" value="org.seckill.dao" />
    </bean>


</beans>
```

**其中jdbc.properties存放jdbc相关参数**

```java
jdbc.driver=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://127.0.0.1:3306/seckill?useUnicode=true&characterEncoding=utf8
jdbc.username=root
jdbc.password=123456
```

 **3.在【spring-web.xml】中完成 Spring MVC 的相关配置：** 

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context.xsd
	http://www.springframework.org/schema/mvc
	http://www.springframework.org/schema/mvc/spring-mvc-3.0.xsd">
    <!-- 配置SpringMVC -->
    <!-- 1.开启SpringMVC注解模式 -->
    <!-- 简化配置：
        (1)自动注册DefaultAnootationHandlerMapping,AnotationMethodHandlerAdapter
        (2)提供一些列：数据绑定，数字和日期的format @NumberFormat, @DateTimeFormat, xml,json默认读写支持
    -->
    <mvc:annotation-driven />

    <!-- 2.静态资源默认servlet配置
        (1)加入对静态资源的处理：js,gif,png
        (2)允许使用"/"做整体映射
     -->
    <mvc:default-servlet-handler/>


    <!-- 3.配置jsp 显示ViewResolver -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="viewClass" value="org.springframework.web.servlet.view.JstlView" />
        <property name="prefix" value="/WEB-INF/jsp/" />
        <property name="suffix" value=".jsp" />
    </bean>

    <!-- 4.扫描web相关的bean -->
    <context:component-scan base-package="org.seckill.web" />
</beans>
```

4.在spring-service完成service包的注解扫描以及配置事务管理器

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context
	http://www.springframework.org/schema/context/spring-context.xsd
	http://www.springframework.org/schema/tx
	http://www.springframework.org/schema/tx/spring-tx.xsd">
    <!-- 扫描service包下所有使用注解的类型 -->
    <context:component-scan base-package="org.seckill.service" />

    <!-- 配置事务管理器 -->
    <bean id="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!-- 注入数据库连接池 -->
        <property name="dataSource" ref="dataSource" />
    </bean>

    <!-- 配置基于注解的声明式事务 -->
    <tx:annotation-driven transaction-manager="transactionManager" />
</beans>
```

整合框架繁杂且重复，SpringBoot解放了SSM整合的繁琐。

## 5 并发优化

### 5.1 缓存优化

在优化之前，首先弄清楚秒杀的高并发发生在哪？如下图，红色部分代表可能会有高并发区域：

<div align="center"><img src="pics//1553603148(1).png" width="350px"></div>
**详情页**：参与秒杀的第一步，所有参与用户都会访问此页面。

**系统时间**：在进行当前时间与秒杀开始时间对比的过程中，由于系统访问一次内存的时间(Cacheline)非常短，大约是10ns，因此这一部分可以不做具体优化。

**地址暴露接口**：使用服务端缓存：Redis 集群。6 个 Redis 实例，3 个节点，每个节点都有自己的 slave 做备份，详细搭建过程自行 google 。

**Redis 与数据库数据一致性保证**：

<div align="center"><img src="pics//1553603931(1).png" width="350px"></div>
键值上设置过期时间，超时穿透；或者每当数据库变更时，主动更新至缓存。

### 5.2 数据库优化

业务逻辑中原始数据落地的方法：

<div align="center"><img src="pics//1553604430(1).png" width="350px"></div>
数据库行级锁持有的时间（最差）：(update + 网络延迟 + GC)  +  (insert + 网络延迟 + GC) 。GC 的 Stop the World 时长大概在 50ms 左右，当系统中并发量越高，GC 就越频繁。

**优化的方向**：如何减少行级锁持有的时间？

**优化思路**：

- 把客户端逻辑放到 MySQL 服务端，避免网络延迟和 GC 的影响。
  - 定制 SQL 方案：update /* + [suto_commit] */，需要修改 MySQL 源码。
  - 使用存储过程：整个事务在 MySQL 端完成。[seckill.sql](https://github.com/MinheZ/seckill/blob/master/src/main/sql/seckill.sql)

本项目学习自[慕课网](https://www.imooc.com/u/2145618/courses?sort=publish&skill_id=220)。
