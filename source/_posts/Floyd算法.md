---
title: Floyd算法
mathjax: true
date: 2021-04-11 10:54:52
categories: 算法
tags: floyd
---

## Floyd 算法

&ensp;&ensp;Floyd算法也是求最短路径的一种算法，主要用于计算两两节点之间最短的距离。不像dijstra是固定一个起点，在Floyd中每一个点都可以是起点，用来计算它到其它节点之间的最短距离。其实Floyd就像是执行了n次dijstra算法。

<!--more-->

## 算法描述

&ensp;&ensp;给定一个带权重的图G=(V,E)，可以存在负权(但不能存在负权环路)。V代表顶点的集合，E代表顶点之间的权重。

我们要计算任意两个顶点之间最短距离。

1、**例如：AB两个顶点之间的最短距离不一定是A直接到B的距离，有可能是A经过C之后再去B得到的最短距离**。

2、我们可以称C为AB的媒介，那怎样去找这些媒介呢？

3、**遍历**，没错就是遍历其它点，若存在一个媒介可以是Dis(A, C) + Dis(C, B) < Dis(A, B)，则我们就可以更新Dis(A, B)=Dis(A, C) + Dis(C, B)。最终遍历完一遍，我们就能知道AB之间的最短距离了。

4、因此，**我们在计算的过程中可以不断的更新两个点之间的最短距离**。

## 代码逻辑

&ensp;&ensp;&ensp;代码很好理解，就是**三重循环**，最外层表示媒介，里面两层表示两个端点。同时我们用tPath这个变量记录任意两点之间最短距离经过的路径，若两点之间不存在媒介，则$tPath[i][j]=-1$，表示二者之间直接连接就是最短路径。

```go
package main

import (
	"fmt"
	"strconv"
)

//https://juejin.im/post/5cc79c93f265da035b61a42e

type Floyd struct {
	tTwoPointDis [][]int
	tPath        [][]int
}

func (f *Floyd) Init(tDis [][]int) {
	f.tTwoPointDis = tDis

	r := len(tDis)

	f.tPath = make([][]int, r)
	for i := range f.tPath {
		f.tPath[i] = make([]int, r)
	}
	for i := 0; i < r; i++ {
		for j := 0; j < r; j++ {
			f.tPath[i][j] = -1
		}
	}
}

func (f *Floyd) solve() {
	fmt.Println("before")
	for _, tNums := range f.tTwoPointDis {
		for _, tNum := range tNums {
			fmt.Print(tNum, " ")
		}
		fmt.Println()
	}
	r := len(f.tTwoPointDis)
	for k := 0; k < r; k++ { // 媒介
		for i := 0; i < r; i++ {
			for j := 0; j < r; j++ {
				if f.tTwoPointDis[i][j] > (f.tTwoPointDis[i][k] + f.tTwoPointDis[k][j]) {
					f.tPath[i][j] = k // 记录媒介
					f.tTwoPointDis[i][j] = f.tTwoPointDis[i][k] + f.tTwoPointDis[k][j]
				}
			}
		}
	}
	fmt.Println("after")
	for _, tNums := range f.tTwoPointDis {
		for _, tNum := range tNums {
			fmt.Print(tNum, " ")
		}
		fmt.Println()
	}

	for i := 0; i < r; i++ {
		for j := 0; j < r; j++ {
			if i != j {
				fmt.Println(f.getPath(i, j))
			}
		}
	}
}

func (f *Floyd) getPath(i, j int) string { // 打印路径
	if f.tPath[i][j] == -1 {
		return " " + strconv.Itoa(i) + " " + strconv.Itoa(j)
	} else {
		k := f.tPath[i][j]
		return f.getPath(i, k) + f.getPath(k, j)
	}
}

func main() {
	tDis := [][]int{
		{0, 2, 6, 4},
		{127, 0, 3, 127},
		{7, 127, 0, 1},
		{5, 127, 12, 0}}

	f := new(Floyd)
	f.Init(tDis)
	f.solve()
}

```

## 结语

&ensp;&ensp;ok，这就是floyd算法，我们不能被它的名字给吓住了。其实就是利用三重循环，计算图中任意两点的最短距离。

