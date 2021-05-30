---
title: Globalization Date Transfer (上) - java.util.Date
date: 2021-05-22 15:21:52
tags:
	- Globalization
	- Java
	- MyBatis
categories:
	- Web
index_img: /images/globalization.jpg
mermaid: true
---


{% note info %}
Date transfer in globalization web app
{% endnote %}

<!-- more -->
# 关于日期与时间

> 做国际化产品时，一定会涉及到不同国家不同时区转换问题。为保证时间一致性，后端该如何选择数据结构，如何定义API？


## 框架介绍

- Spring
- MySQL 5.7
- MyBatis

## GMT && UTC

GMT与UTC，是常听到两个时间相关的名词，可以简单的理解为UTC基于GMT的升级，UTC使用原子时钟定义时间，克服了GMT的弊端（时间流逝不均匀），所以UTC也就最终成为世界标准时间。


1. [参考链接-格林尼治标准时间GMT](https://zh.wikipedia.org/wiki/%E6%A0%BC%E6%9E%97%E5%B0%BC%E6%B2%BB%E6%A8%99%E6%BA%96%E6%99%82%E9%96%93)
2. [参考链接-协调世界时UTC](https://zh.wikipedia.org/wiki/%E5%8D%8F%E8%B0%83%E4%B8%96%E7%95%8C%E6%97%B6)

## UNIX时间戳 && 	ISO8601


**UNIX时间戳：** 是类UNIX系统中所采用的时间格式，记录的是UTC(0时区) 1970年1月1日0时0分0秒起至现在的总秒数，
由于一般以32位有符号整数表示，故表示范围有限，存在比较著名的[2038问题](https://stackoverflow.com/questions/2012589/php-mysql-year-2038-bug-what-is-it-how-to-solve-it)，有兴趣的话可以一看。

**ISO8601：** 是国际标准组织对日期表示制定的一套标准，常见的形式如：‘2021-05-30T12:00:00+08:00’，详细介绍查看该[链接](https://www.iso.org/iso-8601-date-and-time-format.html)


## Mysql 中的 Date

MySql 中主要的时间表示方式，有Timestamp与Datetime。其实两者差别不大，从表现形式、底层存储上看基本是一个数据结构，主要的不同点如下：

| 功能 | Datetime | Timestamp |
|:----:|:----:|:----:|
| 表示范围 | '1000-01-01 00:00:00.000000' to '9999-12-31 23:59:59.999999 | '1970-01-01 00:00:01.000000' to '2038-01-19 03:14:07.999999'|
| 存储的时候是否要转为UTC(零时区) | 否 | 是 |
| 字节大小 | 5字节 | 4字节 |


所以在实际应用中，最好还是有限考虑Datetime。一方面是其表示范围广，另一方面是在实际应用中会避免掉Mysql server自动转时区问题。


{% note primary %}
[Mysql 8.0.19开始支持日期携带分区](https://dev.mysql.com/doc/refman/8.0/en/time-zone-support.html)
{% endnote %}


## Java 中的 Date

{% note info %}
Java 的时间类比较乱，如java.util.\*，java.sql.\*, java.time.\*
{% endnote %}


尽管java 8推出了新的*java.time*，而且是推荐使用，但*java.util.Date*依然保留。目前很多框架对*java.time*的支持并不友好（如 *Jackson*）。
所以本系列文章会通过*java.util.Date* 与 *java.time.ZonedDateTime*演示，来指导实际应用时的选型。本篇主要以*java.util.Date*为例。


# Date 参数转换


Request 的Date参数使用ISO8601标准定义，然后利用Jackson [@JsonFormat](https://www.baeldung.com/jackson-jsonformat)来自动将时间串参数转换为服务器时区的Date，代码如下：

``` java
public class DateDemoRequest {
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:MM:SSZ")
    private Date date;

// 省略 setter getter
}
```


# Mybatis - Mysql 转换

{% note info %}
Mybatis 默认使用 DateTypeHandler处理date参数，并不会做时区转换。
MySQL-Connector中的ClientPreparedQueryBindings会处理时区。
{% endnote %}

### 概念介绍
- [MySQL session timezone](https://dev.mysql.com/doc/refman/8.0/en/time-zone-support.html)
- [Connector connectionTimeZone&forceConnectionTimeZoneToSession](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-connp-props-datetime-types-processing.html#cj-conn-prop_connectionTimeZone)

**简单讲：**
当table column为Timestamp类型时，Mysql 会根据session timezone是否是0时区来转换timestamp的值；
jdbc url properties设置connectionTimeZone时，ClientPreparedQueryBindings会将时间（Date&Timestamp）转换为该时区的时间，但是并不会修改session timezone。只有同时设置forceConnectionTimeZoneToSession=true时，才会修改session timezone。

#### connectionTimeZone配置方式

``` properties
jdbc:mysql://localhost:3306/test?connectionTimeZone=UTC&forceConnectionTimeZoneToSession=true
```

{% note warning %}
**假设Server时区为+08:00**
**假设MySQL session timezone为+08:00**
{% endnote %}


### 默认情况

> 不设置connectionTimeZone（默认服务器时区）与 forceConnectionTimeZoneToSession（默认MySQL session timezone）


| 数据库字段类型 | request参数值 | server转换后的date | 生成的sql value | 最终存储值 |
|:----:|:----:|:----:|:----:|:----:|
| Timestamp | 2021-05-30T21:00:00+07:00 | 2021-05-30T22:00:00+08:00 | 2021-05-30T22:00:00+08:00 | 2021-05-30T14:00:00+00:00(0时区) |
| Datetime | 2021-05-30T21:00:00+07:00 | 2021-05-30T22:00:00+08:00 | 2021-05-30T22:00:00+08:00 | 2021-05-30T22:00:00+08:00 |

### 只设置connectionTimeZone=UTC


| 数据库字段类型 | request参数值 | server转换后的date | 生成的sql value | 最终存储值 |
|:----:|:----:|:----:|:----:|:----:|
| Timestamp | 2021-05-30T21:00:00+07:00 | 2021-05-30T22:00:00+08:00 | 2021-05-30T14:00:00+00:00 | 2021-05-30T06:00:00+00:00(0时区) |
| Datetime | 2021-05-30T21:00:00+07:00 | 2021-05-30T22:00:00+08:00 | 2021-05-30T14:00:00+00:00 | 2021-05-30T14:00:00+00:00 |


### 设置connectionTimeZone=UTC && forceConnectionTimeZoneToSession=true

| 数据库字段类型 | request参数值 | server转换后的date | 生成的sql value | 最终存储值 |
|:----:|:----:|:----:|:----:|:----:|
| Timestamp | 2021-05-30T21:00:00+07:00 | 2021-05-30T22:00:00+08:00 | 2021-05-30T14:00:00+00:00 | 2021-05-30T14:00:00+00:00(0时区) |
| Datetime | 2021-05-30T21:00:00+07:00 | 2021-05-30T22:00:00+08:00 | 2021-05-30T14:00:00+00:00 | 2021-05-30T14:00:00+00:00 |

# 小结

  由以上各类情况可以判断，当设置**connectionTimeZone=UTC && forceConnectionTimeZoneToSession=true**，Timestamp与Datetime 数据是一致的（都是0时区），所以不论是数据查询还是数据导出都会比较方便，所以建议设置connectionTimeZone=UTC&forceConnectionTimeZoneToSession=true。

  若不设置，则务必保证Server 与 MySQL的时区一致（都设置为UTC），避免潜在的数据转换问题。