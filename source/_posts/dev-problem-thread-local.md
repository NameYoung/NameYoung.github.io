---
title: Web开发问题-ThreadLocal异常值
date: 2022-05-21 15:18:21
tags:
    - Java
    - 线程池
    - Thread
    - 开发问题
categories:
    - Web开发
index_img: /images/resolve-question.jpg
mermaid: true
---

{% note info %}
当前线程中的ThreadLocal为什么出现异常值？
{% endnote %}

<!-- more -->

# 问题描述

## 当前逻辑

1. Web Request Header中会携带**uuid**参数（参数可以省略）
2. 服务器Filter在请求执行前将**uuid**设置到当前请求线程的ContextHolder中，并在请求结束时清除ContextHolder

``` java
public class Context {
    private String uuid;

    // getter & setter
}

public class ContextHolder {
    private static final ThreadLocal<Context> CONTEXT_HOLDER = 
    	new NamedInheritableThreadLocal<>("Context");

    public static void clearContext() {
        CONTEXT_HOLDER.remove();
    }


    public static Context getContext() {
        Context ctx = CONTEXT_HOLDER.get();

        if (ctx == null) {
            ctx = createEmptyContext();
            CONTEXT_HOLDER.set(ctx);
        }

        return ctx;
    }
    // 省略非必要代码
}
```

## 具体问题

1. 第一次请求，HttpRquest Header中**有uuid**，ContextHolder.CONTEXT_HOLDER uuid被成功设置并使用
2. 第二次请求，HttpRquest Header中**无uuid**，但ContextHolder.CONTEXT_HOLDER依然有值，且值为第一次请求的uuid

{% note danger %}
Web Filter会执行ContextHolder.clearContext()，不应该存在<2>中的异常情形
{% endnote %}

# 问题排查：

## 原因分析

- ContextHolder.clearContext直接调用ThreadLocal.remove，所以不存在清理失效问题。
- 整个过程并没有使用缓存，逻辑上讲，存在以下两个可能的原因

**猜测（一）**：请求处理过程中，请求线程池第一次初始化，但清理时只清理了子线程
**猜测（二）**：由ThreadLocal 源码可知，ThreadLocal.remove只是清除了对Context对象的引用，可能存在线程池Context对象复用的情况
 

## Debug排查

### 线程池初始化时机

``` java
// AbstractEndpoint.class 920行
public void createExecutor() {
    internalExecutor = true;
    TaskQueue taskqueue = new TaskQueue();
    TaskThreadFactory tf = 
    	new TaskThreadFactory(getName() + "-exec-", daemon, getThreadPriority());
    // corePoolSize 默认为10，即server.tomcat.threads.min-spare
    executor = 
    	new ThreadPoolExecutor(
    		getMinSpareThreads(), getMaxThreads(), 60, TimeUnit.SECONDS,taskqueue, tf);
    taskqueue.setParent( (ThreadPoolExecutor) executor);
}

// org.apache.tomcat.util.threads.ThreadPoolExecutor 75行
// 默认在启动时，创建所有的core threads
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory) {
    super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, new RejectHandler());
    prestartAllCoreThreads();
}
```

{% note info %}
关于Spring Boot配置 [链接1](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties.server.server.tomcat.threads.min-spare)
关于Tomcat minSpareThreads [链接2](https://tomcat.apache.org/tomcat-8.0-doc/config/http.html)
{% endnote %}

### Context对象复用

> 由上一小节可知，请求处理线程池在WebServer启动时就已经初始化，所以**猜测（一）**不成立


项目启动时，会有main线程来初始化请求线程池，即创建10个（默认）请求处理子线程，此时子线程会继承main线程中的ThreadLocal，继承方法如下：
``` java
// Thread.class 448行
if (inheritThreadLocals && parent.inheritableThreadLocals != null)
    this.inheritableThreadLocals =
        ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        
// ThreadLocal.class 400行
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (Entry e : parentTable) {
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
            	// 简单理解为获取到父线程（main线程）的CONTEXT_HOLDER，并set到子线程中
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}

// key.childValue(e.value) 默认代码如下
// InheritableThreadLocal 66行
// 该方法默认并未处理parentValue，而是直接返回，即所有子线程实际引用了同一个Context对象
protected T childValue(T parentValue) {
    return parentValue;
}

```



如上述源代码，请求线程池中的子线程本质上引用了同一个Context对象，
第一请求该对象时，更改了共同引用的Context对象的值，当第一次请求结束时，只是清除了当前线程对Context对象的引用(简单理解为，当前线程的ThreadLocal=null)，其他线程依旧保留着该Context引用。这就产生了最开始碰到的问题，即**猜测（二）**成立。


# 解决方案

通过自定义InheritableThreadLocal，重写childValue，使得子线程复制父线程数据时使用新对象

``` java
protected Object childValue(Object parentValue) {
    if (parentValue instanceof Context) {
        Context childValue = new Context();
        childValue.setUuid(((Context)parentValue).getUuid());
        return childValue;
    }
    // 省略其他代码
}
```

# 扩展

在使用线程池时，如果存在需传递ThreadLocal给线程池的情况，也会碰到诸多ThreadLocal的问题。
建议使用 [阿里TTL](https://github.com/alibaba/transmittable-thread-local)

未来会对阿里TTL做个原理介绍