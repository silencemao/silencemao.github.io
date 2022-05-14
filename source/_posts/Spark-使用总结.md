---
title: Spark-使用总结
date: 2021-07-20 23:14:26
categories: 总结
tags: spark
---

&emsp;&emsp;在使用spark的过程中经常会遇到一些问题，有的是自己对api不熟悉引起的，还有一些问题是spark本身存在的bug，为了避免在同一个位置摔倒两次，所以要把平时遇到的问题记录下来。其实自己之前积攒了很多问题，想着一一把它记录下来（代码已经有了），可是随着时间的推移，之前的一些问题现在看上去还是自己太过初级了，就一直都没有动笔。Anyway，好记性不如烂笔头，还是要行动起来。

<!--more-->

### 1、"org.apache.spark.sql.AnalysisException: resolved attribute(s)"

&emsp;&emsp;这个问题我还真是遇到了两次，其实它后面还跟着一些报错，"missing column xxx from col1, col2, col3"等，具体的报错我记不太清楚了。**主要问题是**，当两个dataframe df1, df2 join时，比如关联的字段是[a1, a2]，明明关联字段在左右两个表中都存在，但是**关联的时候就是会报错找不到字段**。很奇怪的问题，对df1，df2追本溯源，他们都来自同一张表，经过不同的变换之后，想把df1，df2关联起来时，就会报上面这个错误。

#### 解决方案

&emsp;&emsp;解决方案很简单，把关联字段[a1,a2]分别改成别的名字，比如都变成[a11,a22]，这个操作要在df1和df2中都执行一次。然后进行关联即可。

> Ref：https://gankrin.org/how-to-fix-spark-error-analysisexception-resolved-attributes/

