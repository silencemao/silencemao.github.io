---
title: Spark和Pandas结合使用.md
date: 2022-05-21 18:35:01
categories: 总结
tags: spark
---

&emsp;&emsp;本篇文章主要讲解spark和pandas结合应用的一个例子，我之前在工作中总是单纯的使用spark，有时候会将DataFrame转换为临时表，然后使用Hive-sql处理，或者是写Udf做稍微复杂一些的处理。在前段时间接触到spark可以和pandas结合使用，还真是又涨了点知识。

&emsp;&emsp;举一个例子，一个DataFrame的size是[m,n]，我想对其进行groupby操作，然后返对每个分组内的上下两个row进行一些操作，最后返回一个和和原DataFrame大小一致的新df，最初我想到的一个方案是对组内每两行先打一个相同的tag，然后再结合window进行操作，不过这种方案比较麻烦，不直观，而且对每两行之间做一些复杂的运算可能也不是很友好。

&emsp;&emsp;再举一个例子，我想对df分组，计算组内分位数，但是我要求分位数必须是组内出现的数字，而不是插值之后的数字，并且还要对一些异常值进行判断。你可能会想到使用hivesql中的percentile来做，但是它返回的未必是组内的值，而且对异常值判断不是很方便。

&emsp;&emsp;对于上面两个例子，单纯的使用spark和hivesql都不是最优雅的方案，这时候pandas就出现了，它可以很好地解决这两类问题。

<!--more-->
