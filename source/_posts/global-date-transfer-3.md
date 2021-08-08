---
title: Globalization Date Transfer (三) - Framework date handler
date: 2021-07-13 22:48:14
tags:
	- Globalization
	- Java
	- MyBatis
	- Jackson
	- Spring
	- JDBC
categories:
	- Web
index_img: /images/globalization.jpg
mermaid: true
---

{% note info %}
How the framework or library handle date?
{% endnote %}

<!-- more -->

# Spring MVC

## @RequestParam

### java.util.Date

```
RequestParamMethodArgumentResolver.resolveArgument
  -> WebDataBinder.convertIfNecessary (create by ServletRequestDataBinderFactory)
    -> SimpleTypeConverter.convertIfNecessary
      -> TypeConverterDelegate(SimpleTypeConverter)
        -> SimpleTypeConverter.ConversionService.convert
          -> GenericConverter(FormattingConversionService$AnnotationParserConverter).converter
            -> 1.Set pattern info… (if using @DateTimeFormat)
            -> 2.ParserConverter.converter
              -> DateFormatter.parse
                ->SimpleDateFormat.parse (默认服务器时区)
```


### java.time.\*

{% note info %}
本篇以ZonedDateTime为例，演示对java.time.\*的处理
{% endnote %}

```
RequestParamMethodArgumentResolver.resolveArgument
  -> WebDataBinder.convertIfNecessary (create by ServletRequestDataBinderFactory)
    -> SimpleTypeConverter.convertIfNecessary
      -> TypeConverterDelegate(SimpleTypeConverter)
        -> SimpleTypeConverter.ConversionService.convert
          -> GenericConverter(FormattingConversionService$AnnotationParserConverter).converter
            -> 1.Create DateTimeFormatter(set pattern if using @DateTimeFormat)
            -> 2.ParserConverter.converter
              -> TemporalAccessorParser.parse
                -> ZonedDateTime.parse
                  -> DateTimeFormatter.parse
```

![pic 1: DateBinder Class Diagram](/images/date/data-binder-class-diagram.png)


## @RequestBody && @ResponseBody

MappingJackson2HttpMessageConverter (see [Section Jackson](#Jackson))

{% note success %}
**Tips:**
 1. 使用*Date*做参数时，注意结合注解**@DateTimeFormat**
 2. **@DateTimeFormat** 提供了fallbackPatterns配置，支持多种pattern解析，非常实用
{% endnote %}

# Jackson 

> 为了支持java.time.\*，**Jackson**封装了Jdk8Module插件，
> 详细信息可以看[官网](https://github.com/FasterXML/jackson-modules-java8)，它提供了java.time.\*内时间类的Serialize与Deserialize, 引入方式如下：
``` xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
</dependency>
```

### java.util.Date

#### Serializer

```
objectMapper.writeValue
  -> DateSerializer.serialize(JsonGenerator)
    -> StdDateFormat.format
      1. Calendar.setTime
      2. 字符串拼接为ISO8601格式
      
```

#### Deserializer

```
objectMapper.readValue
  -> DateDeserializer(JsonParser)
    -> StdDateFormat.parseAsISO8601 (默认0时区)
      1. 利用正则表达式解析
      2. 设置Calendar值Calendar.getTime
```

{% note info %}
StdDateFormat PATTERN_ISO8601 为：
**\d\d\d\d[-]\d\d[-]\d\d[T]\d\d[:]\d\d(?:[:]\d\d)?(\.\d+)?(Z|[+-]\d\d(?:[:]?\d\d)?)?**
{% endnote %}


### java.time.\*

#### Deserializer

```
objectMapper.readValue
  -> InstantDeserializer.deserialize
    -> DateTimeFormatter.parse (默认0时区)
```

#### Serializer

```
objectMapper.writeValue
  -> ZonedDateTimeSerializer.serialize
    -> DateTimeFormatter.format (默认0时区)
```

## @JsonFormat

> 用于对Date的序列化，当添加注解的类型为*Date*时，pattern即为*SimpleDateFormat*的pattern，本质改变了DateSerializer.\_customFormat

{% note warning %}
**@JsonFormat**也可以用于 **java.time.\***
{% endnote %}

{% note success %}
**Tips:**
 1. 注意注解 **@JsonFormat** 与 **@DateTimeFormat** 的结合
 2. **@JsonFormat** 对应于*SimpleDateFormat*，有一定的局限性
 3. 要熟悉**Jackson**的默认反序列化行为（正则匹配的方式非常灵活）
{% endnote %}

# Java

## SimpleDateFormat

> SimpleDateFormat 主要用于格式化与解析 java.util.Date

![pic 2: SimpleDateFormat pattern](/images/SimpleDateFormat.png)

#### 局限性

- 不直观，潜规则多，需要记忆个字符的具体含义，便捷性差（如yy的解析，取向前80年，向后20年 64->1964;12->2012）
- ISO支持不灵活，不可以同时支持 +0800 or +08:00
- 非线程安全

![pic 3: SimpleDateFormat wrong parse](/images/WrongYearParse.png)

{% note info %}
SimpleDateFormat 默认使用server所在时区解析，底层是CalendarBuilder创建Date
{% endnote %}


## DateTimeFormatter

> DateTimeFormatter 主要用于格式化与解析 java.time.\*

![pic 4: DateTimeFormatter](/images/DateTimeFormatter.png)

#### 优势

- 线程安全
- 兼容SimpleDateFormat pattern
- 解析更灵活（DateTimeFormatterBuilder）
- 使用方便，功能丰富

``` java
// 可以同时支持 '+08', '+0800', or '+08:00'
DateTimeFormatter isoZonedDateTimeFormatter = new DateTimeFormatterBuilder()
    .parseStrict()
    .append(DateTimeFormatter.ISO_DATE_TIME)
    .appendPattern("[xx][x][xxx]")
    .toFormatter();
```

{% note success %}
**Tips:**
 1. 优先使用**java.time.\*** 与 **DateTimeFormatter**
 2. 尽量不要用**DateTimeFormatter**兼容历史兼容*SimpleDateFormat*
 3. 尽量使用内置的Formatter 如**DateTimeFormatter.ISO_DATE_TIME**
{% endnote %}


# Mybatis
 
## DateTypeHandler

### setParam

``` java
// 将Date转换为了Timestamp(本质还是Date)，默认为server时区时间
  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Date parameter, JdbcType jdbcType)
      throws SQLException {
    // ClientPreparedStatement.setTimestamp()
    ps.setTimestamp(i, new Timestamp(parameter.getTime()));
  }
```

### getParam

``` java
  @Override
  public Date getNullableResult(ResultSet rs, String columnName)
      throws SQLException {
    // 由Resultset.getTimestamp获取日期数据
    Timestamp sqlTimestamp = rs.getTimestamp(columnName);
    if (sqlTimestamp != null) {
      return new Date(sqlTimestamp.getTime());
    }
    return null;
  }
```

## ZonedDateTimeTypeHandler

### setParam

``` java
// 与DateTypeHandler不同，此处调用的是PreparedStatement.setObject
  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, ZonedDateTime parameter, JdbcType jdbcType)
          throws SQLException {
    ps.setObject(i, parameter);
  }
```

### getParam

``` java 
  @Override
  public ZonedDateTime getNullableResult(ResultSet rs, String columnName) throws SQLException {
    // 调用了ResultSet.getObject 
    return rs.getObject(columnName, ZonedDateTime.class);
  }
```

**DateTypeHandler**与**ZonedDateTimeTypeHandler**并未对日期做处理，而是直接调用了 PreparedStatement/ResultSet 实现类 ClientPreparedStatement与ResultSetImpl，这两个类是由**Connector/J**实现，细节在下节讲解(see [Section JDBC](#JDBC))

# JDBC

> Mysql type <-> Java type table
https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-reference-type-conversions.html

{% note info %}
{% btn https://insidemysql.com/support-for-date-time-types-in-connector-j-8-0/, Connector/J 8.0.22 的升级与改造, Connector/J 8.0.22 的升级与改造 %}
简而言之，增加了时区转换的支持
{% endnote %}

## JDBC重点参数介绍

> connectionTimeZone=UTC&forceConnectionTimeZoneToSession=true&preserveInstants=true

- connectionTimeZone: 设置client(connection) timezone
- forceConnectionTimeZoneToSession: 是否强制session使用connectionTimeZone
- preserveInstants: ResultSet是否需要按照connectionTimeZone解析date

## PreparedStatement(set param)

#### ZonedDateTime

``` 
ClientPreparedStatement
  -> ClientPreparedQueryBindings.setObject
        //转为timestamp, 时区是JVM时区
    -> Timestamp.valueOf(ZonedDateTime.withZoneSameInstant(SERVER_TIMEZONE).toLocalDateTime()) 
      -> NativeServerSession.getDefaultTimezone
  -> ClientPreparedQueryBindings.bindTimestamp
        //即connectionTimeZone
    -> SimpleDateFormat.setTimezone (NativeServerSession.getSessionTimezone) 
        // 将时间转换为session timezone所在时区
    -> SimpleDateFormat.format 
```

#### Date

```
ClientPreparedStatement
  -> ClientPreparedQueryBindings.bindTimestamp
        //即connectionTimeZone
    -> SimpleDateFormat.setTimezone (NativeServerSession.getSessionTimezone) 
        // 将时间转换为session timezone所在时区
    -> SimpleDateFormat.format 
```

## ResultSet(get value)

#### ZonedDateTime & Date

> 适用于Mysql type datetime&timestamp

```
ResultSetImpl.getTimestamp
  -> ByteArrayRow.decodeAndCreateReturnValue
    -> MysqlTextValueDecoder.decodeTimestamp
      -> new InternalTimestamp  
      -> SqlTimestampValueFactory.localCreateFromTimestamp 
        //preserveInstants = true & timezone = connectionTimeZone
        -> Calendar.getInstance(timezone) 
        -> new Timestamp(Calendar.getTimeInMillis())
```

{% note success %}
**Tips:**
 1. 使用**ZonedDateTime**，应重写handler，避免使用*SimpleDateFormat*
 2. 尽量保证MysqlServer&AppServer统一时区
 3. 日期统一UTC(0时区)存储
{% endnote %}


# 小结

由以上几个类库处理Date的方式可知:
- 都提供了java.time.\*的支持
- 核心类还是SimpleDateFormat/DateTimeFormatter

从**易用性、安全性、合理性**等角度来看，推荐抛弃*java.util.Date*，使用封装性更好的**java.time.\***，在使用时需要自己编写处理类，特别是对于Mybatis。
下一章会讲解避免通过*Timestamp/SimpleDateFormat*映射**ZonedDateTime**