---
title: 雪花算法与Hash分表键的坑
date: 2020-12-20 01:56:00
updated: 2020-12-20 01:56:00
tags:
  - 分布式
  - 唯一ID生成算法
categories: 
  - [技术总结, mysql]
comments: true
permalink: tech/mysql/snowflake&hash.html 
---

# 1 雪花算法的位数原理


雪花算法在 Java 中默认使用 Long，它是 64 位，各个位的含义如下：


![](https://cdn.nlark.com/yuque/0/2020/png/203689/1609150636502-3a9e2cf2-2046-4158-9970-8181b6224850.png#align=left&display=inline&height=72&margin=%5Bobject%20Object%5D&name=&originHeight=72&originWidth=1438&size=0&status=done&style=none&width=1438)




第一位组：0 写死，表示只会生成正数
第二位组：41位，存储的时间戳单位位毫秒，实际存储量位：2^41/1000*60*60*24*365 = 69，即时间戳最多支持 69 年，一般都是使用 `System.currentTimeMillis() - initDate` 差值来实现
第三位组：10 位，机器数，用于分布式环境下每个机器生成的值都会不一样，默认支持 2^10 = 1024 个节点生成的值不同
第四位组：12 位，每节点每毫秒并发数，默认为 2^12 = 4096

实际业务场景中，会根据线上的业务场景改动某些位数。而且由于雪花是全局递增，因此动态改动也是可以，不过会出现 id 的明显断层。


# 2 Hash 分表原理


在 MySQL 数据量突增的场景，一般都会根据业务特性来使用横切或纵切达到提高性能的目的。这里只讨论横切，也就是是水平分表。大部分都会使用 Hash 分表，也就是简单的 Hash 取模，而分表都是按 2 的次方分，例如如果按 1024 分表，是如何计算的。


假设目前有1~1024个列，主键从 1~1024递增，那么按 1024 分表的话，每列都会分在独立的一张表，以分表列值分别为 1，2，3，按 1024 分表的计算过程如下：
```shell
1 & (1024 - 1) = 1
2 & (1024 - 1) = 2
3 & (1024 - 1) = 3
```


二进制操作的图例如下：
![](https://cdn.nlark.com/yuque/0/2020/png/203689/1609173791431-d522379a-5cb1-48f4-b726-64bcac71211d.png#align=left&display=inline&height=376&margin=%5Bobject%20Object%5D&name=&originHeight=376&originWidth=2616&size=0&status=done&style=none&width=2616)


这里都是分配均匀，三个列均匀的分配到了 3 个表，但是我们可以根据二进制规律得知，如果有一批数，这一批数其后 10 位都是一样的，那么根据 1024 分表就会出现只分在一个表的情况，例如我现在有三个列，其分表键值分别为 1025，2049，3073，如果按 1024 分表的话，这三列就会都分在第一个表，导致数据倾斜，计算过程如下图所示：


![](https://cdn.nlark.com/yuque/0/2020/png/203689/1609174481106-173a914b-265f-4717-a2d1-605cbf7335d3.png#align=left&display=inline&height=400&margin=%5Bobject%20Object%5D&originHeight=400&originWidth=2854&size=0&status=done&style=none&width=2854)


这样我们就知道了根据 hash 分表数据倾斜的数值特征：后 N 位相同的位数越多，数据倾斜的概率越大，具体后几位，根据你分表数来决定，例如例子中是按 2^10 分表，那么 N 则为 10。接着我们看下雪花算法的位数特性，看看会不会有数据倾斜问题


# 3 雪花算法的位数特性


雪花算法后 12 位为“每节点每毫秒并发数”，在线上有 100 个节点运行，而你的 QPS 为 100000，均匀的打到了每台机器，即每台机器的 QPS 为 1000，这样，你的“每节点每毫秒并发数”为 1000/1000 = 1，这样生成的 100000 个雪花算法的后 12 位全都是一样都是 0，此时如果你对雪花算法做 hash，就会出现这 100000 个数据全部在第 1 个表中，造成了严重的数据倾斜问题，只有当分表数大于 2^13 = 8192 时，才会好点，如果你按 2^13 数量分表，则因为后 12 位都是 0，此时会计算到第三组中 10 位的机器数，但是此时你也只能分成两个表，因为只计算了 1 位，如下图示例，表示的就是两个节点，在“每节点每毫秒并发数”为 1 时，按 8192 的分表的过程：

![](https://cdn.nlark.com/yuque/0/2020/png/203689/1609175735724-2e24b0e8-20ad-4201-b9c8-a11585ce09b6.png#align=left&display=inline&height=498&margin=%5Bobject%20Object%5D&name=&originHeight=498&originWidth=2408&size=0&status=done&style=none&width=2408)


“每节点每毫秒并发数”为1，即“每节点每秒并发数”为 1000 以下就会出现严重的数据倾斜问题，但是每节点 1000 的 QPS 已经很高，如果此时我们需要用雪花算法生成的值来做分表键，有什么方法呢？有两种：

1. 改进雪花算法位数特性
1. 改进 hash 分表算法



# 4 雪花算法位数的改进


既然我们知道按 2次幂的 hash 主要是看低位，那么我只要保证雪花算法生成的值低位不同即可，如果线上机器够多，例如线上有 128 台机器，这时候我们按 128 分表，我们就可以将雪花算法的第三组和第四组调换，这样生成的最后 10 位是机器节点，肯定是不一样的，从而达到了平衡。但是这样要求节点数据量大于分表数。还有一个更好的方法，那就是将第二组的时间戳与第四组调换，这样就算你线上只有一个节点，但是生成的最后 10 位的时间戳也达到了散列的效果。


# 5 hash 算法的改进


这里就涉及到 hash 的“扰乱”，即，hash 不再是简单的取余，而是增加了对原始值的“打散”，减少出现在同一个组的可能，这里参考 HashMap 的 hash 方法：
```java
 static final int hash(Object key) {
     int h;
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
 }
```


实际上的代码如下：
```java
static int tableSize = 1024;

public static void main(String[] args) throws Exception {
    SnowflakeIdWorker idWorker = new SnowflakeIdWorker(0);
    long id1 = idWorker.nextId();
    System.out.println("原始雪花算法值id1:" + id1);
    Thread.sleep(1);
    long id2 = idWorker.nextId();
    System.out.println("原始雪花算法值id2:" + id2);
    System.out.println("------------------------");
    System.out.println("table1Index:" + hash(id1) + "\n" + "table2Index:" + hash(id2));
    System.out.println("table1Index:" + hashC(id1) + "\n" + "table2Index:" + hashC(id2));
    /**
         * output:
         * 原始雪花算法值id1:793292040770158592
         * 原始雪花算法值id2:793292040774352896
         * ------------------------
         * table1Index:0
         * table2Index:0
         * 扰乱后的雪花算法值:793299744572601664
         * 扰乱后的雪花算法值:793299744568407424
         * table1Index:320
         * table2Index:384
     */
}

public static long hash (long snowflakeId) {
    return snowflakeId % tableSize;
}

public static long hashC (long snowflakeId) {
    long c = (snowflakeId ^ (snowflakeId >>> 16));
    System.out.println("扰乱后的雪花算法值:" + c);
    return c % tableSize;
}
```


我们可以看到通过原生直接简单 hash 取余都是 0，但是通过扰乱后的再取余就不一样了，达到了“扰乱散列”的效果。


还有一种 hash 改进的方式针对这种最后几位一样，但是前几位不一样的特征值，我们丢弃掉后几位，也就是直接右移几位，这里我们可以将默认的雪花算法右移 12 位，从而直接根据机器节点来取模，或者粗暴的右移 22 位，根据时间戳在取余，效果更好。在雪花算法场景下，这种方式比“扰乱散列”方式更好。


# 6 总结


这是我在分库分表时，踩下的坑，当时某个业务有个列是雪花算法生成的，因为业务都是它来查值，因此我在分表的时候把它设为分表键，前期数据量比较少，没关注，后面量在 900G 的时候，发现 0000 表占了 500G。然后就开始重新建表，重新分库，开始数据平滑迁移到新表。




