---
title: 四、Schema与数据类型优化
date: 2017-12-25 12:31:00
updated: 2017-12-28 00:28:00
comments: true
tags:
  - mysql
categories: 
  - [读书总结, 分布式服务框架原理与实践]
permalink: high_performance_MySQL/4.html    
---

# 1 选择优化的数据类型

1. 更小的通常更好
2. 简单就好：整型比字符操作代价更低，使用 MySQL 内建的类型而不是字符串来存储日期和时间，以及使用整型存储 IP 地址
3. 尽量避免 NULL：可为 NULL 的列会使用更多的存储空间。 InnoDB 使用单独的位（bit）存储 NULL 值，但这不适用于 MyISAM

在为列选择数据类型时，先确定大类型：数字、字符串、时间等。下一步是选择具体类型，很多数据类型可以存储相同类型的数据，只是存储的长度和范围不一样、允许的精度不同，或者需要的物理空间（磁盘和内存空间）不同。例如， TIMESTAMP 只使用 DATETIME 一半的存储空间，并且会根据时区变化，具有特殊的自动更新能力，另一方面， TIMESTAMP 允许的时间范围要小得多。  
  
本章只讨论基本的数据类型。 MySQL 为了兼容性支持很多别名，例如 INTEGER、BOOL 以及 NUMERIC ，它们只是别名，使用 SHOW CREATE TABLE 检查， MYSQL 报告的是基本类型，而不是别名。

## 1.1 整数类型

TINYINT，SMALLINT，MEDIUMINT，INT，BIGINT。分别使用8、16、24、32、64 位存储空间。它们可以存储的值得范围 -2^(N-1) ~ 2^(N-1) - 1，其中 N 是存储空间的位数。整数类型有可选的 UNSIGNED 属性，表示不允许负数，这样可以提高一倍的正数上限。  
  
整数计算一般使用 64 位的 BIGINT 整数。整数类型指定宽度，例如 INT(11)，对大多数应用这是没有意义的：它不会限制值得合法范围，只是规定了 MySQL 的一些交互工具（例如 MySQL命令行客户端）用来显示字符的个数。对于存储和计算来说，INT(1) 和 INT(20) 是相同的。

## 1.2 实数类型

实数是带有小数部分的数字。然而，它们不只是为了存储小数部分，也可以使用 DECIMAL 存储比 BIGINT 还大的整数。 MySQL 既支持精确类型，也不支持不精确类型。
因为 CPU 不支持对 DECIMAL 的直接计算，所以在 MySQL5.0+ MySQL 服务器自身实现了 DECIMAL 的高精度计算，相对而言，CPU 直接支持原生浮点计算，所以浮点运算明显更快。  
DECIMAL 的字节存储：每四个字节存储 9 个数字，例：DECIMAL(18,9) 小数点两边将各存储 9 个数字，一共使用 9 个字节：小数点前的数字用 4 个字节，小数点后的数字用 4 个字节，小数点本身占 1 个字节。  
浮点类型在存储同样范围的值时，通常比 DECIMAL 使用更少的空间。 FLOAT 使用 4 个字节存储。 DOUBLE 占用 8 个字节。MySQL 使用 DOUBLE 作为内部浮点计算的类型。  
将结果存储在 BIGINT 里，这样可以同时避免浮点存储计算不精确和 DECIMAL 精确计算代价高的问题。（根据小数的位数乘以相应的倍数）

## 1.3 字符串类型

### 1.VARCHAR 和 CHAR 类型

#### VARCHAR

VARCHAR 类型用于存储可变长字符串，如果 MySQL  表使用 ROW_ FORMAT = FIXED 创建的话，每一行都会使用定长存储，这会很浪费空间。  
VARCHAR 在列最大长度 <=255 字节的时候，额外用 1 个字节用于记录字符串的长度。 例：VARCHAR(10) 的列需要 11 个字节的存储空间。VARCHAR(1000) 的列则需要 1002 个字节，因为需要 2 个字节存储长度信息。  
MySQL5.0+ 在存储和检索时会保留末尾空格。  
但是，由于行是变长的，在 UPDATE 时可能使行变得比原来长，这就导致需要额外的工作。  
另外，InnoDB 可以把过长的 VARCHAR 存储为 BLOB，稍后讨论该问题。

#### CHAR

CHAR 类型是定长的，MySQL 总是根据定义的字符串长度分配足够的空间。  
存储 CHAR 值时，MySQL 会删除所有的末尾空格。

#### CHAR VS VARCHAR

CHAR 适合存储很短的字符串，或者所有值都接近同一个长度。例：存储密码的 MD5 值，因为这是一个定长的值。  
对于经常变更的数据， CHAR 也比 VARCHAR 更好，因为定长的 CHAR 类型不容易产生碎片。  
对于非常短的值， CHAR(1) 比 VARCHAR(1) 在存储空间上也更有效率（后者需要额外一个字节存储长度）。  
VARCHAR(100) 和 VARCHAR(200) 虽然在存储空间相同，但是在内存消耗不同，后者更大。尤其在排序和临时表（中间表）时。  
 摘自：http://tech.it168.com/a2011/0426/1183/000001183173.shtml

### 2. BLOB 和 TEXT 类型

BLOB 采用二进制存储、TEXT 采用字符存储。  
与其它类型不同，MySQL 把每个 BLOB 和 TEXT 值当做一个独立的对象处理。当其太大时， InnoDB 会使用专门的“外部”存储区域进行存储。此时每个值在行内需要 1~4 个字节存储一个指针，然后再外部存储区域实际的值。  
排序：MySQL 只对每个列的最前 max_sort_length 字节而不是整个字符串做排序。可以减少 max_sort_length 的值或者使用 ORDER BY SUBSTRING(column, length)。  
MySQL 不能讲 BLOB 和 TEXT 列全部长度的字符串进行索引，也不能使用这些索引消除排序。  
进行 ORDER BY 为了防止临时表过大，可以使用 SUBSTRING(column, length) 进行长度切割。

### 3. 使用枚举（ENUM） 代替字符串类型

MySQL 在存储枚举时非常紧凑，会根据列表值得数量压缩到一个或者两个字节中。 MySQL 在内部会将每个值在列表中的位置保存为整数，并且在表的 .frm 文件中保存 “数字-字符串”映射关系的“查找表”。  
在 VARCHAR 与 ENUM 互相 JOIN 关联时，ENUM 与 ENUM 最快。因此如果不是必须和 VARCHAR 列进行关联，那么转换这些列为 ENUM 就是个好主意。这是一个通用的设计实践，在“查找表”时采用整数主键而避免采用基于字符串的值进行关联。

## 1.4 日期和时间类型

### 1.DATETIME

这个类型能保存大范围的值，精度为秒。使用 8 个字节的存储空间。

### 2.TIMESTAMP

保存了从 1970年1月1日~2038年，MySQL 提供了 FROM_UNIXTIME() 和 UNIX_TIMESTAMP() 函数将日期和 Unix 时间戳转换。使用 4 个字节存储。

## 1.5 位数据类型

### 1.BIT

尽量少用。

### 2.SET

如果需要保存很多 true/false 值，可以考虑合并这些列到一个 SET 数据类型，它在 MySQL 内部是以一系列打包的位的集合来表示的。这样就有效的利用了存储空间。缺点是改变列的定义代价较高：需要 ALTER TABLE（这对大表是非常昂贵的操作，但是后面给出了解决方法）。一般来说，也无法再 SET 列上通过索引查找。
>在整数列进行按位操作
>```sql
>SET @CAN_READ   := 1 << 0,
>     @CAN_WRITE  := 1 << 1,
>     @CAN_DELETE := 1 << 2;
>CREATE TABLE acl (
>     perms TINYINT UNSIGNED NOT NULL DEFAULT 0    
>);
>INSERT INTO acl(perms) VALUES (@CAN_READ+@CAN_DELETE);
>SELECT perms FROM acl WHERE perms & @CAN_READ;
>```
>当然，也可以使用代码变量而不是 MySQL 变量。

## 1.6  选择标识符（identifier）

标识列与其它值进行比较（例，在关联操作中），或通过标识列寻找其它列。标识列也可能在另外的表中作为外键使用。  
选择标识列的类型时，不仅仅需要考虑存储类型，还需要考虑 MySQL 对这种类型怎么执行计算和比较。例， MySQL 在内部使用整数存储 ENUM 和 SET 类型，然后在做比较操作时转换为字符串。  
在可以满足值得范围的需求，并且预留未来增长空间的前提下，应该选择最小的数据类型。例如，TINYINT 比 INT 少了 3 个字节，但是可能导致很大的性能差异。  
尽量使用整数。如果存储 UUID 值，用 UNHEX() 函数转换为 16 字节的数字存储，并且存储在一个 BINARY(16) 列中。

## 1.7 特殊类型数据

例，IPv4 地址人们通常使用 VARCHAR(15) 列来存储 IP 地址。然而，它们实际上是 32 位无符号整数，不是字符串。所以应该用无符号整数存储 IP 地址。 MySQL 提供 INET_ATON() 和 INET_NTOA() 函数在这两种表示方法之间转换。

# 2 MySQL schema 设计中的陷阱

## 2.1 太多的列

MySQL 的存储引擎 API 工作时需要在服务器层和存储引擎层之间通过行缓冲格式拷贝数据，然后再服务器层将缓冲内容解码成各个列。列转行的操作代价是非常高的。

## 2.2 太多的关联

阿里手册规定单次关联不能超过 3 张表。

## 2.3 全能的枚举

CREATE TABLE ... ( country enum('', '0', '1', ... , '31'))  
当需要在枚举列表中增加一个新的国家时就要做一次 ALTER TABLE 操作，在 MySQL5.0- 这是一种阻塞操作，即使在 MySQL5.0+ ，如果不是在列表的末尾增加值也会一样需要 ALTER TABLE。

## 2.4 变相的枚举

枚举列允许在列中存储一组定义值中的单个值，集合（SET）列则允许在列中存储一组定义值的一个或多个值。这会导致混乱。

## 2.5 非此发明（Not Invent Here）的 NULL

CREATE TABLE ... (dt DATETIME NOT NULL DEFAULT '0000-00-00 00:00:00')  
伪造的全 0 值可能导致很多问题。（可以配置 MySQL 的 SQL_MODE 来禁止不可能的日期，对于新应用这是个非常好的实践经验）。

# 3 缓存表和汇总表

有时提升性能最好的方法是在同一张表中保存衍生的冗余数据。然而，有时也需要创建一张完全独立的汇总表或缓存表。

## 3.1 计数器表

创建一张独立的表存储计数器通常是个好主意。例，有一个计数器表，只有一行数据，记录网站的点击次数：  

```sql
CREATE TABLE hit_counter (
    cnt int unsigned not null
) engine=InnoDB;
```

每次点击：`UPDATE hit_counter SET cnt = cnt + 1;`  
问题在于，对于任何想要更新这一行的事务来说，这条记录上都有一个全局的互斥锁（mutex）。这会使得这些事务只能串行执行。要获得更高的并发更新性能，也可以将计数器保存在多行中，每次随机选择一行进行更新。  
要获得统计结果：`SELECT SUM(cnt) FROM hit_count;`。  
一个常见的需求是每隔一段时间开始一个新的计数器（例，每天一个）。如果需要这么做，则可以再简单地修改一下表设计：  

```sql
CREATE TABLE hit_counter (
    day date not null,
    slot tinyint unsigned not null,
    cnt int unsigned not null,
    primary key(day, slot)
) ENGINE=InnoDB;
```

在这个场景下，可以不用像前面的例子那样预先生成行，而是`ON DUPLICATE KEY UPDATE`代替。  

```sql
INSERT INTO daily_hit_counter(day, slot, cnt)
    VALUES (CURRENT_DATE, RAND() * 100, 1)
    ON DUPLICATE KEY UPDATE cnt = cnt + 1;
```

如果希望减少表的行数，以避免表变得太大，可以写一个周期执行的任务，合并所有结果到 0 号槽，并删除所有其它的槽：

```sql
UPDATE daily_hit_counter as c
    INNER JOIN (
        SELECT day, SUM(cnt) AS cnt, MIN(slot) AS mslot
        FROM daily_hit_counter
        GROUP BY day
    ) AS x USING(day)
SET c.cnt  = IF(c.slot = x.mslot, x.cnt, 0),
    c.slot = IF(c.slot = x.mslot, 0, c.slot); 
DELETE FROM daily_hit_counter WHERE slot <> 0 AND cnt = 0;
```

# 4 加快 ALTER TABLE 操作的速度

假如要修改电影的默认租赁期限，从三天改到五天，下面是很慢的方式：

```sql
ALTER TABLE film 
MODIFY COLUMN rental_duration tinyint(3) not null default 5;
```

`show status`语句显示这个语句做了 1000 次读和 1000 次插入操作。换句话说，它拷贝了整张表到一张新表。  
理论上，MySQL 可以跳过创建新表的步骤，即直接修改 .frm 文件而不设计表数据：

```sql
ALTER TABLE film
ALTER COLUMN rental_duraion SET DEFAULT 5;
```

# 5 总结

1. 避免过度设计
2. 使用小而简单的合适数据类型，避免使用 NULL 值
3. 关联标识符尽量使用相同的数据类型
4. 注意可变长字符串，其在临时表和排序时可能导致悲观的按最大长度分配内存
5. 尽量使用整型定义标识列
6. 小心使用 ENUM 和 SET
7. `ALTER TABLE`在大部分情况下都会锁表并且重建整张表。建议先在备库执行`ALTER`完成后将其切换为主库
