---
title: Redis Get Key
tags:
  - redis
  - 中间件
categories:
  - Redis
index_img: /images/redis-logo.png
date: 2021-01-24 18:30:42
mermaid: true
---

{% note info %}
为什么 GET [key] 复杂度为 O(1) ?
{% endnote %}

<!-- more -->

# 键值对存储结构

> Redis 所有键值对存储在一个全局字典表中，利用key来快速查找匹配到对应的键值对，时间复杂度O(1)
可以类比 **Java HashMap** 

![pic 1: redis 字典结构](/images/redis-dic.png)

{% note success %}
补充：
- ht[0]、ht[1] 是为rehash准备的
- 字段具体结构会在后续文章展开
{% endnote %}

# Key查找方法

> 源文件 src/dict.c#dicFind

``` c
dictEntry *dictFind(dict *d, const void *key)
{
	// *d 即为键值对字典对象指针
    dictEntry *he;
    uint64_t h, idx, table;

    if (dictSize(d) == 0) return NULL; 
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 获得key的hash值
    h = dictHashKey(d, key);
    // table即为上图中的 ht数组（共有两个元素）
    for (table = 0; table <= 1; table++) {
    	// 计算key的index
        idx = h & d->ht[table].sizemask;
        // 获取到index对应链表的头指针
        he = d->ht[table].table[idx];
        while(he) {
        	// 逐个判断是否相等
            if (key==he->key || dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        // 若没有正在镜像 rehash, 则直接结束 即rehash时两个table都要进行扫描
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```

由源代码可知, 查找key（dictEntry）只需两步
1. 计算hash值并计算索引
2. 链表上逐个匹配

## 调用链

{% mermaid %}
flowchart TB
    getGenericCommand-->lookupKeyReadOrReply
	subgraph t_string.c
    getCommand-->getGenericCommand
    end
    subgraph db.c
    lookupKeyReadOrReply-->lookupKeyReadWithFlags
    lookupKeyReadWithFlags-->lookupKey
    lookupKey-->dictFind
    end
{% endmermaid %}

# 小结

{% note info %}
由以上可知，Redis GET Key 并不是严格意义上的 O(1), 实际上是O(1+M) M即链表上查找所经过的长度。但是为什么文档上写O(1)呢？这是因为Redis rehash保证了每个桶（dictEntry 链表）持有尽可能少的元素，所以时间复杂度近似O(1)。
{% endnote %}


下一篇，继续从源码入手研究rehash策略。