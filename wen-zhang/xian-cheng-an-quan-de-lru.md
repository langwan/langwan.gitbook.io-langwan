# 线程安全的LRU

LRU是Least Recently Used的缩写，即最近最少使用，是一种常用的页面置换算法，选择最近最久未使用的页面予以淘汰。该算法赋予每个页面一个访问字段，用来记录一个页面自上次被访问以来所经历的时间 t，当须淘汰一个页面时，选择现有页面中其 t 值最大的，即最近最少使用的页面予以淘汰。

线程安全是指在高并发下保持数据的一致，所以一般我们会选择明确标记为thread safe的LRU使用，因为既然是一种高效缓存往往也意味着高并发。

LRU的几个特征值：

1. 从外观上看是一个key value存储，或者更接近于任何语言中的map
2. size - 有一个容器的上限值，最多可以储存多少个key，超过上限值的的时候淘汰最早的内容。
3. 从实现上来说是一个map和一个链表的组合。

### golang实现 LRU

1. golang实现LRU可以使用泛型来解决数据类型问题。你也可以不用，你也可以用。
2. 链表部分可以使用 "container/list" 下的 list.list来实现。

需要实现的方法

New 创建新的

Add 添加（会更新最近未使用）

Get 获取（会更新最近未使用）

Set 设置（会更新最近未使用）

Peek 获取（不会更新最近未使用）

Range 遍历所有的元素

实现代码：

* [chilru](https://github.com/langwan/chilru) thread safe lru cache, written in go
