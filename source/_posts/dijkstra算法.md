---
title: dijkstra算法
mathjax: true
date: 2020-07-05 16:07:53
categories: 算法
tags: dijkstra
---

## Dijkstra算法

&emsp;&emsp;中文名又叫迪杰斯特拉算法，是一种单源最短路径算法，用于计算一个节点到其它所有节点的最短路径。通俗的讲就是确定好一个起点之后，计算起点到其它点最短路径。常用于一些路由计算或者路径规划等场景。

<!--more-->

## 算法描述

&emsp;&emsp;给定一个带权有向图G=(V, E)，V代表顶点集合，E代表顶点之间的权重。

1、把顶点分成两个集合S、U，S代表已经获得最短路径的顶点，起初只有源点一个，U代表未加入路径的顶点。（保持源点s到S中各个顶点的最短路径长度不大于源点s到U中各个顶点的最短路径长度）

2、从U中选出一个顶点k，是从源点到U中所有顶点距离最短的一个，将k加入S，并从U中移除顶点k

3、根据S中现有的顶点，更新s到U中各个顶点的距离，比如之前s->m的距离是无穷大，现在经过（s->k） + （k->m）为常数值。

4、重复2、3两步，直到U中的顶点为空

## 代码逻辑

&emsp;&emsp;整体的代码逻辑也很简单，首先我们需要两个列表，一个表示访问过的点S一个表示未访问过的点U。一个map path 用于存储从源点到已访问过点的路径。然后我们每次只需要计算从源点s经过S中的某个/某些点之后 到 U中各个点的距离，只需要找出到U中距离最短的点即可。我们可以把S中最后一个经过的点称之为pre，U中访问的点为next，找到路径最短的next之后，我们将next移动到S中。并且源点s到next的路径只是在源点s到pre的基础上加了个u，将s->u的路径加入到path中即可。



```go
package main

import "fmt"

// https://github.com/muzixing/graph_algorithm/blob/master/dijkstra.py

const(
	MaxDis  int = 1<<7-1
)

type Dijkstra struct {
	tPints  []string
	tTwoPointDis map[string]int
}

func (d *Dijkstra) Init(tPoints []string, tDis [][]int) {
	if len(tPoints) != len(tDis) {
		panic("点数与矩阵的大小不一致")
	}
	d.tTwoPointDis = make(map[string]int, 0)
	for i := 0; i < len(tPoints); i++ {
		for j := 0; j < len(tPoints); j++ {
			key := tPoints[i] + "_" + tPoints[j]
			d.tTwoPointDis[key] = tDis[i][j]
		}
	}
	d.tPints = tPoints
}

func (d *Dijkstra) dijkstra() {
	tPoints := d.tPints[1:]                  // 未访问过的点
	visited := []string{d.tPints[0]}         // 访问过的点
	src := d.tPints[0]                       // 起点
	pre, next := src, src

	path := make(map[string][]string, 0)     // 起点到其它点的路径
	path[src + "_" + src] = []string{"A"}

	distanceGraph := make(map[string]int, 0)  // 起点到其它点的距离
	for len(tPoints) > 0 {
		distance := MaxDis
		var ind int = 0
		var dst string

		var nextInd int = 0

		for _, v := range visited {
			for ind, dst = range tPoints {
				newDis := d.tTwoPointDis[src + "_" + v] + d.tTwoPointDis[v + "_" + dst]  // 从起点src到已访问过的点v + 从v到未访问过点的距离
				if newDis < distance {
					distance = newDis
					pre = v
					next = dst
					nextInd = ind
					d.tTwoPointDis[src + "_" + dst] = distance
				}
			}
		}

		for _, tPoint := range path[src + "_" + pre] {
			path[src + "_" + next] = append(path[src + "_" + next], tPoint)
		}
		path[src + "_" + next] = append(path[src + "_" + next], next)  // 记录从src到next需经过的路径

		distanceGraph[src + "_" + next] = distance                     // 记录从src到next的距离

		visited = append(visited, next)
		tPoints = append(tPoints[:nextInd], tPoints[nextInd+1:]...)
	}

	fmt.Println(path)
	fmt.Println(distanceGraph)
}

func main() {
	d := new(Dijkstra)
	tPoints := []string{"A", "B", "C", "D"}
	tDis := [][]int{
		{0,      2, 6, 4},
		{127, 0, 3, 127},
		{7, 127, 0, 1},
		{5, 127, 12, 0}}

	d.Init(tPoints, tDis)
	d.dijkstra()
}

```



## 结语

&emsp;&emsp;Ok，整体的代码逻辑就是这样的，从最初不了解dijkstra算法，到了解用代码实现之后，发现其中的逻辑不算复杂。只要我们能够理解S U两个列表，以及中间状态的存储path，还有如何从U中获得下一个要访问的点。整个问题就解决了。
