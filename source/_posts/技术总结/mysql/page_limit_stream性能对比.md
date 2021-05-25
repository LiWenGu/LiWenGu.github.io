---
title: page_limit_stream性能对比
date: 2019-06-02 12:50:00
updated: 2019-06-02 12:50:00
tags:
  - mysql
  - 性能调优
categories: 
  - [技术总结, mysql]
comments: true
permalink: prac/sum/page_limit_stream.html  
---

![][0]

<!-- more -->

# 1 背景

当导出数据量很大时，分页查询会影响内存和数据库连接池，并且速度随着分页 offset 变大而变慢，因此需要优化

# 2 三种方法对比

主要参考：  
1: https://ifrenzyc.github.io/2017/11/16/mysql-streaming/  
2: http://knes1.github.io/blog/2015/2015-10-19-streaming-mysql-results-using-java8-streams-and-spring-data.html  
测试 jvm 配置： -Xms512m -Xmx512m  
测试数据量：60W 无查询条件

## 2.1 原生分页写法

```
select * from table limit pageNo, pageSize  
```

内存：  
![][1]  
时间耗时：三次测试导出，基本在200s~210s之间

## 2.2 优化之后的游标导出

注意：因为使用的主键索引查询范围，因为不支持排序，但是可以导出后让运营在 Excel 中手动排序

```sql
# 每次查询后，将最大的 id 传入下次作为条件
select * from table where id > ${id} limit pageSize
```
![][2]
时间耗时：三次测试导出，基本在15s左右

## 2.3 使用 stream

这是 mysql 本身支持的特性，使用流式处理数据  

jpa 语法：  
```java

Stream<Dao> streamAll();

streamAll.forEach(
    // 业务处理
);
```
![][3]  
时间耗时：三次测试导出，基本在13s左右

# 3 最终选择

选择第二种，因为 stream 的缺点在于长事务，虽然是只读，不会有锁但是属于长连接，而且该 SQL 属于慢查询，在 DBA 就过不去。  
而第二种更加容易接受和理解，虽然苦了运营~

[0]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/background/2019-05-26%E7%95%AA%E8%8C%84%E9%9D%A2.jpg
[1]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/others/limit_jmm.png
[2]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/others/offset_jmm.png
[3]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/others/stream_jmm.png
