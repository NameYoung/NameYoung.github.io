---
title: SpringBoot DBUnit
date: 2021-03-07 21:27:30
tags:
	- DBUnit
	- SpringBoot
	- UnitTest
	- MyBatis
categories:
	- UnitTest
index_img: /images/dbunit-logo.jpg
mermaid: true
---

{% note info %}
如何在Spring项目中使用DBUnit?
{% endnote %}

<!-- more -->

> 项目中一直打算引入数据库单测，抽空研究了一下DBUnit. 并根据项目做了些简单修改，本文将以 **MyBatis+MySQL** 为例创建 DBunit Test

# DBUnit 简述

> [DBUnit项目地址](http://dbunit.sourceforge.net/)

{% note primary %}
DBUnit 是一个数据库单测工具，基于JUnit.
目的是执行单测试时保证数据库数据已知以及数据可重复利用
{% endnote %}

# Spring DBunit

> [项目地址](https://springtestdbunit.github.io/spring-test-dbunit/sample.html)

{% note primary %}
Spring 项目的维护者 @philwebb 创建了一个开源项目spring-test-dbunit 方便DBUnit 在Spring‘套餐’中使用。PS: 由于他本人很忙，此项目已经废弃。
{% endnote %}

# 演示

## 依赖引入

> 以gradle为例

``` groovy
dependencies {
	implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.1.4'
	// 用flyway来管理数据库脚本
	implementation 'org.flywaydb:flyway-core'

	runtimeOnly 'mysql:mysql-connector-java'

    // 核心包
	testImplementation 'com.github.springtestdbunit:spring-test-dbunit:1.2.0'
	testImplementation 'org.dbunit:dbunit:2.7.0'
	testImplementation 'com.h2database:h2'
	// 常规包
	testImplementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter-test:2.1.4'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testImplementation 'junit:junit'
}

```

## 数据库脚本

**src/main/resources/db/migration/V1__init_user.sql**
``` sql
// 若类路径有flyway包，则spring boot会自动加载
create table if not exists user
(
    id         int auto_increment primary key,
    last_name  char(30) not null,
    first_name char(30) not null
);
```

## 创建基本类

**UserMapper.java**

``` java
@Mapper
public interface UserMapper {
    User selectById(@Param("id") Long id);
}
```

**User.java**

``` java
public class User {
    private Long id;
    private String lastName;
    private String firstName;

    // setter && getter
}
```

**UserMapper.xml**
``` xml
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.spring.mapper.UserMapper">
    <resultMap type="com.example.spring.model.User" id="userResultMap">
        <id column="id" property="id"/>
        <result column="last_name" property="lastName"/>
        <result column="first_name" property="firstName"/>
    </resultMap>

    <select id="selectById" resultMap="userResultMap" parameterType="long">
        select *
        from
            `user`
        where id = #{id}
    </select>
</mapper>
```

## 测试类

### 测试配置文件

> 覆盖默认配置，本例使用h2内存数据库，所以达到每次执行测试都有一个新库的效果，不会影响真实数据库环境

**test/java/resourcesapplication.properties**
``` properties
spring.datasource.url=jdbc:h2:mem:test
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=root
spring.datasource.password=root

mybatis.mapper-locations=classpath:mybatis/*.xml
```

### DBUnit dataset文件

> 使用DBUnit自身的CSV DataSet, 也可使用xml等文件配置数据集

**test/java/resources/csv/table-ordering.txt**
``` txt
user
```

**test/java/resources/csv/user.csv**
``` txt
id,first_name,last_name
3,wukong,sun
4,seng,tang
```

### DBUnit 配置

> 使用CSV文件时，需要做一些自定义配置。使用XML可以参考spring-test-dbunit的DEMO

**CsvLoader.java**
``` java
public class CsvLoader extends AbstractDataSetLoader {
    @Override
    protected IDataSet createDataSet(Resource resource) throws Exception {
        return new CsvURLDataSet(resource.getURL());
    }
}
```

**CustomTestConfiguration.java**
``` java
@TestConfiguration
public class CustomTestConfiguration {

    @Bean
    public DataSetLoader dataSetLoader() {
        return new CsvLoader();
    }
}
```

### 测试类

**DbunitTest.class**
``` java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = CustomTestConfiguration.class)
@MybatisTest
@DbUnitConfiguration(databaseConnection = "dataSource", dataSetLoader = CsvLoader.class)
@TestExecutionListeners({DependencyInjectionTestExecutionListener.class,
                         DbUnitTestExecutionListener.class})
public class DbunitTest {
    @Autowired
    private UserMapper userMapper;

    @Test
    @DatabaseSetup("classpath:csv/user.csv")
    public void testSelectOne() {
        User userA = userMapper.selectById(3L);
        assertEquals("wukong", userA.getFirstName());

        User userB = userMapper.selectById(4L);
        assertEquals("tang", userB.getLastName());
    }
}

```


# 其他组件集成

## Spring JPA

Spring JPA 配置思路基本一致

## MyBatisPlus

> 与MyBatis一致

{% note success %}
引入 com.baomidou:mybatis-plus-boot-starter:{version} 
使用 **@MybatisPlusTest** 替换 **@MybatisTest**
{% endnote %}

# 小结

终于可以免于手动测试了…… 后续抽时间讲一下原理
