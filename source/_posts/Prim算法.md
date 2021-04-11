---
title: Prim算法
mathjax: true
date: 2021-04-11 16:17:50
caytegories: 算法
tags: prim
---

## Prim算法

&ensp;&ensp;假定我们要给各个村子修路，将村子之间互相连通起来，但是呢又不想直在任意两两村子之间直接修，那样会浪费成本。因此我们可以考虑在部分村子之间修，只要保证这些路可以将所有的村子连通起来就好（即村A和村B之间没有直接连通，但是可以通过村C来中转，从A-C-B）。也就是说**我们有N个村子，我们可以修N-1条路，来保证村之间可以有路连通**。也称为最小生成树（最小支撑树），即保持"连通性"的前提下的最小子图，子图各个边的权重之和最小。

<!--more-->

## 解决方案

### 贪心法

&ensp;&ensp;我们设定无向图G=(P, E)为连通图，P为G中的所有顶点，E为顶点之间的边。我们要从中筛选出部分边构成最小生成树，使的边权重之和最小。

我们定义V为已经修好路的顶点，U为还未进行修路的顶点，V中的顶点构成了最小生成树后的子树，U中的点会逐个进入V中，最终生成一个最小生成树。

1、首先我们将第一个顶点放入V中，并将其从U中删除。

2、从U中选择一个距离V最近的顶点$u_k$，即从U中选择一个顶点，它距离V中所有点的最短距离，是U中的顶点的最小的。

3、将$u_k$从U中删除，加入V中。

4、以此类推，直到U中为空，即得到了最小生成树的权重。

## 代码逻辑

&ensp;&ensp;第一种写法，我们首先将第一个顶点加入V中，然后开始尝试m-1次，将剩下的顶点依次纳入V中。重点在于如何求U中距离V最近的顶点，我们这里直接两层循环，遍历U中顶点，计算其与V中所有顶点的最短距离，保存距离最短的U中的顶点的索引。遍历结束，索引对应的U中顶点加入到V中即可。时间复杂度为$O(n^3)$

```go
func prim(matrix [][]int) {
  m := len(matrix)
  V := []int{0}
  var U []int
  for i := 1; i < m; i++ {
    U = append(U, i)
  }

  for k := 0; k < m-1; k++ {
    min := 1<<31 - 1
    ind1, ind2 := -1, -1

    for i := range V {
      for j := range U {
        if matrix[V[i]][U[j]] < min {
          min = matrix[V[i]][U[j]]
          ind1, ind2 = i, j
        }
      }
    }
    fmt.Println(V[ind1], U[ind2], min)
    V = append(V, U[ind2])
    U = append(U[:ind2], U[ind2+1:]...)
  }
}

func main() {
  matrix := [][]int{
    {0, 1, 2, 3},
    {1, 0, 2, 128},
    {2, 2, 0, 4},
    {3, 128, 4, 0},
  }
  prim(matrix)
  fmt.Println()
  prim1(matrix)
}
```

&ensp;&ensp;上面那种写法的时间复杂度比较高，我们可以考虑进行下优化。在找距离V最近的顶点时，是存在优化空间的。不需要每次都遍历V和U，我们可以用一个数组记录下U中顶点到V的最短距离。

1、dis数组的长度为顶点的个数，当V中只有第一个顶点$v_0$，dis中记录了该顶点与U中所有顶点的最近距离（无向图，a->b = b->a）。

2、当V中新增一个顶点$v_1$时，我们可以对dis进行一次更新。若U中存在顶点$u_k$其距离$v_1$的值小于其距离$v_0$的值，我们就可以更新dis中的信息。

3、直接根据dis中的距离来计算距离V最近顶点即可。

代码如下：

```go
func prim1(matrix [][]int) {
	m := len(matrix)
	dis := make([]int, m)
	for i := 0; i < m; i++ {
		dis[i] = matrix[0][i]
	}
	status := make(map[int]bool, m)

	res := 0
	for i := 0; i < m-1; i++ { // 需要找剩余的点
		t := -1
		for j := 1; j < m; j++ { // 每次遍历剩余的所有点
			if !status[j] && (t == -1 || dis[t] > dis[j]) {
				t = j
			}
		}
		res += dis[t]
		fmt.Println(i, t, dis[t])
		for j := 1; j < m; j++ {
			if matrix[t][j] < dis[j] {
				dis[j] = matrix[t][j]
			}
		}
		status[t] = true
	}
	fmt.Println(res)
}

func main() {
	matrix := [][]int{
		{0, 1, 2, 3},
		{1, 0, 2, 128},
		{2, 2, 0, 4},
		{3, 128, 4, 0},
	}
	prim(matrix)
	fmt.Println()
	prim1(matrix)
}

```



