---
title: Globalization Date Transfer (二) - java.time.ZonedDateTime
date: 2021-06-22 15:21:52
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

# Date vs ZonedDateTime

### Date 缺点

详情可以看这篇大佬的[吐槽](https://codeblog.jonskeet.uk/2017/04/23/all-about-java-util-date/)
实际开发来讲，比较棘手的问题如下：
- 明明是个瞬时时间点，但是却可以修改，所以使用过程中存在隐患（如某个方法不小心修改了数据）(非线程安全)
- 方法有歧义，开发易挖坑（如toString()方法默认转换为本地时区时间，但实际其并没有时区属性）
- 格式转换比较麻烦（如输出ISO8601格式）
- 日期运算比较麻烦（如计算两个日期相隔天数）

### java.time.* 优点

- 包含时区，方便时区转换
- 丰富的计算方法，方便时间比较计算
- 线程安全（immutable)

{% note info %}
[JodaTime](https://www.joda.org/joda-time/)也强烈建议开发者使用java.time.*
{% endnote %}

# Spring Date参数转换

> Spring如何处理请求/响应中ISO8601格式的日期？

### RequestParam

对于@*RequestParam*声明的参数，可使用@*DateTimeFormat*进行解析。

### RequestBody && ResponseBody

对于 Request/Response Body 数据的解析与格式化，可使用**Jackson**进行处理，即使用注解@*JsonFormat*。

{% note info %}
Spring 中Jackson解析的核心类为**MappingJackson2HttpMessageConverter**
{% endnote %}

# Mybatis ZonedDateTime

> 上一篇已经介绍的Mybatis对Date的处理，本篇介绍下Mybatis对ZonedDateTime处理方法

### ZonedDateTimeTypeHandler

同**java.util.Date**装换方式类似，将ZonedDateTime装换为ServerSessionTimeZone时区的Timestamp(Date)

# Mysql Timestamp 与 DateTime

Timestamp与DateTime Mybatis会一视同仁，都会转为java.sql.Timestamp(Date)

# 测试

### Mysql global session(+07:00) && server timezone +08:00

| 数据库字段类型 | request参数值 | server转换后的ZonedDateTime | 生成的sql value | 最终存储值 |
|:----:|:----:|:----:|:----:|:----:|
| Timestamp | 2021-05-30T21:00:00+07:00 | 2021-05-30T14:00:00+00:00 | 2021-05-30T22:00:00+08:00 | 2021-05-30T15:00:00+00:00 |
| Datetime | 2021-05-30T21:00:00+07:00 | 2021-05-30T14:00:00+00:00 | 2021-05-30T22:00:00+08:00 | 2021-05-30T22:00:00+08:00 |

### Mysql global session(+07:00) && server timezone +08:00 && connectionTimeZone=UTC&forceConnectionTimeZoneToSession=true

| 数据库字段类型 | request参数值 | server转换后的ZonedDateTime | 生成的sql value | 最终存储值 |
|:----:|:----:|:----:|:----:|:----:|
| Timestamp | 2021-05-30T21:00:00+07:00 | 2021-05-30T14:00:00+00:00 | 2021-05-30T14:00:00+00:00 | 2021-05-30T14:00:00+00:00 |
| Datetime | 2021-05-30T21:00:00+07:00 | 2021-05-30T14:00:00+00:00 | 2021-05-30T14:00:00+00:00 | 2021-05-30T14:00:00+00:00 |


# 小结

实际使用的建议，同Date一样，务必保证Server 与 MySQL的时区一致（都设置为UTC），避免潜在的数据转换问题。
同时也要设置**connectionTimeZone=UTC && forceConnectionTimeZoneToSession=true**