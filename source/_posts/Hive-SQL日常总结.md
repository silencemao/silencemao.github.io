---
title: Hive/SQL日常总结
date: 2020-05-16 09:39:36
categories: SQL
tags:
---

# Hive/SQL日常总结

&emsp;&emsp;说来惭愧，工作了有一段时间了，才开始接触SQL。自己之前从来没有和SQL打过交道。为了不在工作中拖后腿，自己挤时间把《SQL必知必会》这本书看完了。看完之后虽然对SQL有了基本的认识，但是应用起来还是不太熟练，有时候还需要上网查一查相关资料才用。现在把日常使用中会用到的点记录下来，方便以后应用。

<!--more-->

## 1、如何比较两个表的内容是否完全一致

### （1）、分组 inner join

&emsp;&emsp;这个问题我一直没有找到比较简洁有效的方式。看网上有一种做法，假设我们有两个表t1， t2。

&emsp;&emsp;首先对t1按行分组，计算分组的条数num1。

&emsp;&emsp;然后对t2按行分组，计算分组的条数num2。

&emsp;&emsp;最后两个表t1、t2进行inner join，按照所有的列名字以及num进行关联。

&emsp;&emsp;若num1=num2=inner join之后的条数，则说明两个表的内容完全一致。

&emsp;&emsp;假设表结构如下所示，只有两列

|  id  | name |
| :--: | :--: |
| xx1  | Tom  |
| xx2  | Jone |

```sql
对t1进行分组
select id, name count(*) as num 
from t1 
group by id, name;

对t2进行分组
select id, name count(*) as num 
from t2 
group by id, name;

使用t1 inner join t2
select * 
  (select id, name count(*) as num 
  from t1 
  group by id, name) as tmp1
inner join
   (select id, name count(*) as num 
    from t2 
    group by id, name) as tmp2
 on tmp1.id=tmp2.id and tmp1.name=tmp2.name;
```

&emsp;&emsp;若上面第一步num1值等于第二步的num2值，并且等于第三步输出的个数，说明两个表的内容是完全一样的。这个做法对于表的column比较少的情况比较方便，一旦column很大的情况下写起来就不太方便了。

### （2）、minus 做减法

&emsp;&emsp;另一种做法是直接用两个表相互做减法，看返回的是否均为空。

```sql
select * from t1
minus 
select * from t2;

select * from t2
minus
select * from t1;
```

如果返回的内容均为空的话，就说明两个表的内容完全一致，但是**对于表中有重复的行的话这种方法就不适用了。** 还有我在hue页面尝试这个方法的时候，提示我没有minus这个关键字。下次得去hive客户端试试了。

[](https://zhuanlan.zhihu.com/p/113617244)

## 2、分区表增加新的字段

&emsp;&emsp;之前遇到一个问题，就是一个非空分区表需要添加新的字段，然后把数据写入进去。

```sql
alter table table_nam add columns(c1 int);
```

添加完字段之后，发现写入进去之后c1这个列全为null，当时以为是自己计算的错误。然后我在写入之前查了下，发现c1这个字段的数据是有的并且不为null，但是但是写入之后就为null了。后来查资料说到是添加新的字段的方式有问题，然后只能把那个表删掉，重新建表，写入数据。

对于非空分区表添加新的字段的正确方式：

```sql
alter table table_name add columns(c1 int) cascade;
```

[](https://community.cloudera.com/t5/Community-Articles/Adding-new-columns-to-an-already-partitioned-Hive-table/ta-p/245636)

[](https://blog.csdn.net/aijiudu/article/details/79066835)



## 3、将一个表的内容写入到另一个表中

&emsp;&emsp;如果是覆盖原始数据的话，直接使用insert overwrite

```sql
Insert overwrite table table_name partition(dt=’t’)
       Select col1, col2, col2,….
       From table_name
       Where dt=’t-1’

```

上面是将同一个表中一个分区的写入到另一个分区内。

&ensp;&ensp;直接写入一个分区内，相当于追加到对应的分区内。

```sql
Insert into table table_name partition(dt=’t’)
       Select col1, col2, col2,….
       From table_name
       Where dt=’t-1’
```



**注意**：分区字段要写完整

​      选择数据时不能使用 select *， 因为select * 会选中所有字段，包括分区字段，但是我们写入的表中分区字段是作为文件夹名字的，      即实际表中没有分区字段，假如我们表中有7个字段（非分区字段），另外还有4个分区字段，我们在select * 的时候会选出11个字段，但是我们写入的表只有7个字段需要被写入，这样的话就会报错。

因此我们在选择数据时，要用select 选出那7个非分区字段。

## 4、创建表的方式

### 1、直接建表法

```sql
create table t1(
    id      int,
    name    string,
    hobby   array<string>,
    add     map<String,string>
)
row format delimited
fields terminated by ','
collection items terminated by '-'
map keys terminated by ':'
;
```

然后load data进入到表中

```sql
load data local inpath '/user/hive/warehouse/...data' overwrite into table t1;
```

### 2、select 方法

```sql
create table t1 as
select
    id,
    name
from t2;
```

### 3、like建表法

```sql
create table t1
like t2;
```

## 5、删除文件

### 1、删除文件

```shell
hadoop fs -rm -r /user/hive/warehouse/database/table_name/dt=xxxx/city_code=xxxxx

```

上述命令直接在terminal中执行即可，其实就是常用的linux命令前面加上hadoop fs，还有列出某个表的信息

```shell
hadoop fs -ls /user/hive/warehouse/database/table_name/dt=xxxx
```

### 2、删除分区

```shell
alter table table_name drop if exists partition(dt=xxxx, city_code=xxxx);

```

上述命令需要在hive客户端中执行。

## 6、时间处理

### 1、转换为时间戳

标准格式是指'2021-06-30 10:10:00'这种格式，即'yyyy-MM-dd HH:mm:ss'，

```sql
unix_timestamp('2021-06-30 10:10:10')
```

若时间不是标准格式的，比如 '20210630'这种的情况，也可以使用unix_timestamp来转换，但是需要你传入格式，即告诉这个函数你的时间是什么格式的

```sql
unix_timestamp('20210630', 'yyyyMMdd')
```

### 2、时间戳转换为日期

时间戳转换为标准格式/指定格式，需要用到from_unixtime(date, format)，此时的时间是到秒级的，即你的时间戳长度为10位。

```sql
from_unixtime('1625839005', 'yyyy-MM-dd HH:mm:ss')/from_unixtime('1625839005', 'yyyy-MM-dd')
```

对于一些时间戳是到毫秒级其长度为13位，因此我们在转换前需要先取其前10位，即**对字符串进行截取指定长度**，这个在mysql和hivesql中是由一些差异的。**在mysql中有left，right两个函数，但是在hive中可以使用substr来做**。

```sql
from_unixtime((cast(substr('1625839005000', 0, 10) as bigint)), 'yyyy-MM-dd HH:mm:ss')
```

## 7、分位数

在hivesql中，取分位数还是比较简单，有两个函数可供使用，

```sql
percentile(col, p)
```

col为我们要处理的列，但是要求col的值必须都为int，p为0-1的小数，表示分位数，0.3表示3分位数

```sql
percentile_approx(col, array(0.2, 0.3), 9999)
```

col也是我们要处理的列，此时该列的值可以为浮点型也可以为整型，后面可以穿入一个array，一次取多个分位数

