---
title: Spark和Pandas结合使用.md
date: 2022-05-21 18:35:01
categories: 总结
tags: spark
---

&emsp;&emsp;本篇文章主要讲解spark和pandas结合应用的一个例子，我之前在工作中总是单纯的使用spark，有时候会将DataFrame转换为临时表，然后使用Hive-sql处理，或者是写Udf做稍微复杂一些的处理。在前段时间接触到spark可以和pandas结合使用，还真是又涨了点知识。

&emsp;&emsp;举一个例子，一个DataFrame的size是[m,n]，我想对其进行groupby操作，然后返对每个分组内的上下两个row进行一些操作，最后返回一个和和原DataFrame大小一致的新df，最初我想到的一个方案是对组内每两行先打一个相同的tag，然后再结合window进行操作，不过这种方案比较麻烦，不直观，而且对每两行之间做一些复杂的运算可能也不是很友好。或者另一个场景，一个df，第一列是id，第二列是不同的品牌，想要统计某个id下各个品牌的数量(可以有重复)，这时一种常规的方案是对每个品牌进行映射到一个新的列，然后若这一行是品牌1，则打标记为1，其它为0，品牌2的列同理，这样处理起来略显麻烦，pandas就可以很简单的处理。

&emsp;&emsp;再举一个例子，我想对df分组，计算组内分位数，但是我要求分位数必须是组内出现的数字，而不是插值之后的数字，并且还要对一些异常值进行判断。你可能会想到使用hivesql中的percentile来做，但是它返回的未必是组内的值，而且对异常值判断不是很方便。

&emsp;&emsp;对于上面两个例子，单纯的使用spark和hivesql都不是最优雅的方案，这时候pandas就出现了，它可以很好地解决这两类问题。

<!--more-->

## 一、环境介绍

&emsp;&emsp;简单介绍下我所应用的环境：首先是要安装pyarrow这个库，其次python版本不要高于3.7，最后spark环境，我之前使用的是2.4，一直报错，折腾了好久，后来发现是环境问题，切换到3.2就可以了。

```shell
spark:spark-3.2.0-bin-hadoop2.7
python:3.7
pyarrow:8.0.0
```

## 二、Spark与Pandas结合

&emsp;&emsp;spark和pandas结合其实就是把一部分sparkdataframe，转换为pandas dataframe，然后可以比较方便的进行数值计算，就像写单机pandas操作一样。

&emsp;&emsp;我们需要用到 **pyspark.sql.functions import pandas_udf, PandasUDFType**，这两个函数，顾名思义，pandas_udf就是用户自定义的pandas函数

```python
def pandas_udf(
    f: PandasScalarToScalarFunction,
    returnType: Union[AtomicDataTypeOrString, ArrayType],
    functionType: PandasScalarUDFType,
) -> UserDefinedFunctionLike: ...

```

pandas_udf就三个参数：

- f：即用户自定义的func

- returnType：自定义的func返回的值类型

- functionType：枚举值，包含下面四种方式，表示的是我们的函数是按照什么样的方式进行映射，即返回值和输入是怎样的映射关系:

  ```python
  SCALAR: PandasScalarUDFType          ：标量，即返回一个值
  SCALAR_ITER: PandasScalarIterUDFType ：迭代器，这个还需要我探索一下
  GROUPED_MAP: PandasGroupedMapUDFType ：分组映射，分组之后返回df，可以和原df大小一致，也可不一致，用户自己可以控制
  GROUPED_AGG: PandasGroupedAggUDFType ：分组聚合，分组之后返回一个常量值
  ```



&emsp;&emsp;今天主要介绍一下GROUPED_MAP和GROUP_AGG的用法。

| 方法        | 输入       | 输出      | 配合使用方式 |      |      |
| ----------- | ---------- | --------- | ------------ | ---- | ---- |
| GROUPED_MAP | Datafram   | Dataframe | Apply        |      |      |
| GROUPED_AGG | 一列或多列 | 常量      | agg          |      |      |
|             |            |           |              |      |      |



### 1、GROUPED_AGG

&emsp;&emsp;就拿上面那个计算组内分位数的例子来做，

```python
import findspark
findspark.init("/usr/local/spark-3.2.0-bin-hadoop2.7")

import pandas as pd
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql import types as T
from pyspark.sql.functions import pandas_udf, PandasUDFType


spark = SparkSession.builder.appName('xxx').config("spark.driver.host", "xxx").config("spark.driver.bindAddress", "xxxx").getOrCreate()


class M(object):

    def f(self):
        df = spark.createDataFrame(
            [("a1", 2.2, 5), ("a1", 3.16, 1), ("a1", 2.168, 3), ("a1", 2.5, 4), ("a2", 1.915, 2), ("a2", 2.0, 4),
             ("a2", 1.509, 1), ("a2", 2.1, 2)],
            ["id", "max_wheel", "cnt"])
        df.show()
        # 定义pandas_udf，输入函数、返回值类型，聚合类型
        mean_udf = F.pandas_udf(M.__t_agg, "string", F.PandasUDFType.GROUPED_AGG)
        # 应用udf，注意是使用agg，传入两列
        df = df.groupBy(["id"]).agg(mean_udf(df["max_wheel"], df["cnt"]).alias("res"))
        df.show()

        df = df.withColumn("res", F.split(df["res"], ","))
        df = df.withColumn("r1", df["res"].getItem(0)).withColumn("r2", df["res"].getItem(1))
        df.show()
        df.filter(df["id"] == 'a1').show()

    # 定义具体的函数
    @staticmethod
    def __t_agg(v1, v2):
        df = pd.DataFrame(list(zip(v1, v2)), columns=["max_wheel", "cnt"])  # 传入两列，转换为df
        a1 = round(df["max_wheel"].quantile(q=0.5, interpolation="lower"), 2) # 不插值
        a2 = round(df["cnt"].quantile(q=0.5, interpolation="lower"), 2)
        return ",".join([str(a1), str(a2)])

if __name__ == '__main__':
    m = M()
    m.f()

--------------------------------------------------------------
输入
+---+---------+---+
| id|max_wheel|cnt|
+---+---------+---+
| a1|      2.2|  5|
| a1|     3.16|  1|
| a1|    2.168|  3|
| a1|      2.5|  4|
| a2|    1.915|  2|
| a2|      2.0|  4|
| a2|    1.509|  1|
| a2|      2.1|  2|
+---+---------+---+

--------------------------------------------------------------
输出
+---+---------+----+---+
| id|      res|  r1| r2|
+---+---------+----+---+
| a1| [2.2, 3]| 2.2|  3|
| a2|[1.92, 2]|1.92|  2|
+---+---------+----+---+
```

### 2、GROUPED_MAP

&emsp;&emsp;就拿上面计算每个id下不同品牌的数量的例子来看：

```python
class M(object):

    def f1(self):
        df = spark.createDataFrame(
            [('a1', 'a'), ('a1', 'b'), ('a1', 'c'), ('a1', 'a'), ('a1', 'd'),('a1', 'a'), ('a1', 'd'),
             ('a2', 'c'), ('a2', 'b'), ('a2', 'c'), ('a2', 'b'), ('a2', 'd'),('a2', 'a'), ('a2', 'c')],
            ["id", "brand"])
        df.show()
        # 设置返回类型，包括返回的列名，每列值的类型
        schema = T.StructType()
        schema.add(T.StructField("id", T.StringType(), True))
        for col in ["a", "b", "c", "d"]:
            schema.add(T.StructField(col, T.IntegerType(), True))
        print(schema)
        
        # 构建pandas_udf
        cnt_udf = F.pandas_udf(M.__t_map, schema, F.PandasUDFType.GROUPED_MAP)
        # 引用udf，传入的相当于整个df，注意使用的是apply
        df1 = df.groupBy(["id"]).apply(cnt_udf)
        df1.show()

    @staticmethod
    def __t_map(df):
        res = {"id": [], "a": [],  "b": [],  "c": [],  "d": []}
        for index, row in df.iterrows():
            if index > 0:  # 返回的df是groupby之后的，若没有此操作，则返回的df是和原df大小一致的
                break
            res["id"].append(row["id"])

        res["a"].append(df[df["brand"] == 'a'].shape[0])
        res["b"].append(df[df["brand"] == 'b'].shape[0])
        res["c"].append(df[df["brand"] == 'c'].shape[0])
        res["d"].append(df[df["brand"] == 'd'].shape[0])

        res_df = pd.DataFrame(res)
        return res_df
if __name__ == '__main__':
    m = M()
    # m.f()
    m.f1()
    
--------------------------------------------------------------
输入
+---+-----+
| id|brand|
+---+-----+
| a1|    a|
| a1|    b|
| a1|    c|
| a1|    a|
| a1|    d|
| a1|    a|
| a1|    d|
| a2|    c|
| a2|    b|
| a2|    c|
| a2|    b|
| a2|    d|
| a2|    a|
| a2|    c|
+---+-----+
--------------------------------------------------------------
输出
+---+---+---+---+---+
| id|  a|  b|  c|  d|
+---+---+---+---+---+
| a1|  3|  1|  1|  2|
| a2|  1|  2|  3|  1|
+---+---+---+---+---+
```



## 三、总结

&emsp;&emsp;好了，这就是本篇文章对pyspark和pandas一起应用的一个例子，分别介绍了GROUP_AGG和GROUP_MAP两个场景，后续还需要再补充一个SCALAR_ITER的应用，希望能对您有用。



[Ref]: https://spark.apache.org/docs/2.4.4/sql-pyspark-pandas-with-arrow.html

