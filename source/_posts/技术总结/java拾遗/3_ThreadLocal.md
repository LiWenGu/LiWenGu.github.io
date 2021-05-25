---
title: ThreadLocal
date: 2018-04-13 18:00:00
updated: 2018-04-13 19:00:00
comments: true
tags:
  - java
categories: 
  - [技术总结, java拾遗]
comments: true
permalink: A2B_Java/3_ThreadLocal.html    
---

# 1. Thread：ThreadLocalMap = 1:1

每个 Thread 内部维护了一个 ThreadLocal.ThreadLocalMap 对象

# 2. ThreadLocalMap：[ThreadLocal, Entry] = 1:16

每个 ThreadLocalMap 内部维护的键值对是 [ThreadLocal, Entry]。  
而在底层，查找的时候是通过 `ThreadLocal.threadLocalHashCode & (table.length - 1)` 获取值，得到 0~15 之间的值，并获取到 Entry[i]，进而获取 Entry.value。  
>因此一个线程的 ThreadLocal 最好不要超过 16 个

如下底层源码：  
```java
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    private static final int INITIAL_CAPACITY = 16;

    private Entry[] table;
}
```
由于底层通过数组维护，如果在 set 时碰到了 hash 碰撞则进行 replace：  
```java
for (Entry e = tab[i];
        e != null;
        e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();

    if (k == key) {
        e.value = value;
        return;
    }

    if (k == null) {
        replaceStaleEntry(key, value, i);
        return;
    }
}
```

# 3. 图解

![1][]

# 4. 注意

1. ThreadLocal 并未解决多线程访问共享对象的问题，而是每个线程一个 独占的变量
2. ThreadLocal并不是每个线程拷贝一个对象，而是直接new（新建）一个
3. 如果ThreadLocal.set()的对象是多线程共享的，那么还是涉及并发问题。

# 5. Spring 中的 ThreadLocal 使用

Spring使用ThreadLocal解决线程安全问题。一般情况下，只有无状态的Bean才可以在多线程环境下共享，在Spring中，绝大部分Bean都可以声明为singleton作用域。就是因为Spring对一些Bean（如RequestContextHolder、TransactionSynchronizationManager、LocaleContextHolder等）中非线程安全状态采用ThreadLocal进行处理，让它们也成为线程安全的状态，因为有状态的Bean就可以在多线程中共享了。  
一般的Web应用划分为展现层、服务层和持久层三个层次，在不同的层中编写对应的逻辑，下层通过接口向上层开放功能调用。在一般情况下，从接收请求到返回响应所经过的所有程序调用都同属于一个线程。

参考：https://blog.csdn.net/u010887744/article/details/54730556  
参考：http://neoremind.com/2010/11/threadlocal_learn/

[1]: https://img-blog.csdn.net/20170125180420388?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDg4Nzc0NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center
