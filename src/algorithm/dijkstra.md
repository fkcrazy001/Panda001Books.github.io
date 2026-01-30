# Dijkstra 算法

这个算法是用来算 有向图/无向图 中 **单点（源点）** 到其它每个点的最短距离。
和 Floyd 算法不同，Dijkstra **不能处理负权边**。

## 问题描述

有 \\( N \\) 个点（\\( 0 \dots N-1 \\)），\\( g[N][N] \\) 是一个有向图（邻接矩阵），其中 \\( g[i][j] = \infty \\) 说明这两个点之间没有直接连接。
其他情况则说明有一条有向边，权重为 \\( g[i][j] \\) (**必须非负**, \\( g[i][j] \ge 0 \\))。
给定一个源点 \\( start \\)。

需要给出 \\( dist[N] \\), 其中 \\( dist[i] \\) 为从 \\( start \\) 点到 \\( i \\) 点需要的最短距离。

## 算法

```py3
# 初始状态
# dist 数组记录从 start 到各个点的当前已知最短距离
dist = [inf] * N
dist[start] = 0

# visited 数组标记哪些点的最短路径已经被最终确定
visited = [False] * N

for _ in range(N):
    # 1. 贪心策略：在所有未确定的点中，找到距离 start 最近的点 u
    u = -1
    min_val = inf
    for i in range(N):
        if not visited[i] and dist[i] < min_val:
            min_val = dist[i]
            u = i
    
    # 如果找不到可达的点了，提前结束
    if u == -1: break
    
    # 2. 标记确定：点 u 的最短路径已经确定，不可再更改
    visited[u] = True
    
    # 3. 松弛操作 (Relaxation)：借由点 u，更新所有邻居 v 的距离
    for v in range(N):
        if not visited[v] and g[u][v] != inf:
            # 如果经由 u 到达 v 比直接去 v 更近，则更新
            dist[v] = min(dist[v], dist[u] + g[u][v])
```

## 证明

Dijkstra 基于 **贪心选择性质**。
维护两个集合：\\( S \\) (已确定最短路的点集) 和 \\( Q \\) (未确定点集)。
算法的核心是：每次从 \\( Q \\) 中选出一个距离 \\( start \\) 最近的点 \\( u \\)，它的最短路径即被视为确定。

**证明关键点：为什么选出的 \\( u \\) 一定是全局最短？**

使用反证法：
1. 假设存在另一条从 \\( start \\) 到 \\( u \\) 的路径比当前的 `dist[u]` 更短。
2. 这条路径必然经过 \\( Q \\) 中的某个节点（因为起点在 \\( S \\)，终点 \\( u \\) 在 \\( Q \\)）。设路径上**第一个**属于 \\( Q \\) 的节点为 \\( v \\)。
3. 路径形式为：\\( start \to \dots \to v \to \dots \to u \\)。
4. 因为**边权非负**，且 \\( v \\) 是路径上的点，所以 \\( start \to \dots \to v \\) 的距离 (\\( dist[v] \\)) 必然 \\( \le \\) 整条路径长度。
5. 根据贪心选择，\\( u \\) 是 \\( Q \\) 中距离最小的，所以 \\( dist[u] \le dist[v] \\)。
6. 结合第4、5点，得出：整条“更短”路径长度 \\( \ge dist[v] \ge dist[u] \\)。
7. 这与假设（存在比 `dist[u]` 更短的路径）矛盾。

因此，`dist[u]` 就是最短距离。如果存在负权边，第4步推导失效，证明不成立。

### 时间优化

基础算法在寻找 \\( u \\) 时需要遍历 \\( N \\) 次，总复杂度 \\( O(N^2) \\)。
可以使用 **优先队列 (Min-Heap)** 优化查找最小值的过程，将复杂度降为 \\( O(E \log N) \\) (\\( E \\) 为边数)，适合稀疏图。

```py3
import heapq

# graph 使用邻接表: graph[u] = [(v, weight), ...]
pq = [(0, start)] # 堆中存储 (distance, node)
dist = [inf] * N
dist[start] = 0

while pq:
    d, u = heapq.heappop(pq) # O(log N)
    
    if d > dist[u]: continue # 懒删除：如果取出的距离比已知的还大，丢弃
    
    for v, weight in graph[u]:
        if dist[u] + weight < dist[v]:
            dist[v] = dist[u] + weight
            heapq.heappush(pq, (dist[v], v)) # O(log N)
```
