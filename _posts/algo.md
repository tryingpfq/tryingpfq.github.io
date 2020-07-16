## 算法



### 网址

[leetCode](https://leetcode-cn.com/problemset/all/)

[CS-Notes](https://github.com/CyC2018/CS-Notes)



### LeetCode

* 动态规划

  120(三角形最小路径和),

  

### 复杂度的分析

* 时间复杂度

  

* 空间复杂度



### 排序



### 图

图的表示方法

* 邻接矩阵存储方法

  这个是比较直观的，用一个二维数组来表示，假如这个图有10个顶点，那么久有graph[10][10]的二维数组标示，一维坐标标示起点，二维坐标标示终点。但显然，这个是比较浪费空间的，在无向图中，一条边的存在，要表示两次，但这个其实只用到一般的二维数组，那就可以考虑用稀疏矩阵。

* 邻接表存储方法

  每个顶点对应一条链表，链表中存储的是与这个顶点相连接的其他顶点。

* 问题

  1：微博用户是如何存储的

  对于微博用户的存储来说是比较复杂的，主要涉及到关注与被关注。这两种关系就要用两个图来存储。首先，肯定是用邻接表来存储，对于关注，也就是用户A关注了哪些用户，对于没给A顶点的链表来说，就是A关注了哪些用户。那么A被关注了的，改如何做呢，就需要一个逆邻接表，也就是在这图中，A顶点下的链表，是指关注了A的用户。

  所以链表的选择也是很关键的，在微博中，有些用户的粉丝上亿，那么在逆邻接表中，可能存在链表长度过亿，如果要判断两个用户是否互相关注，那链表的查找肯定要快的哦，该选择哪种数据结构呢，其实可以选择跳表的。

  还有一个问题，就是数据量这么大，该如何存储，这就需要用到分布式了，可以根据顶点，hash到不同的机器上。

* 练习

  1：leetcode 785，二分图判断

* BFS & DFS

  ```java
  /**
   * 邻接表 无向图
   *
   * @author tryingpfq
   * @date 2020/7/16
   **/
  public class Graph {
      //顶点的个数
      private int v;
  
      //邻接表
      private LinkedList<Integer> adj[];
  
      public Graph(int v) {
          this.v = v;
          //初始化
          adj = new LinkedList[v];
          for (int i = 0; i < v; i++) {
              adj[i] = new LinkedList<>();
          }
      }
  
      public void addEdge(int s, int t) {
          adj[s].add(t);
          adj[t].add(s);
      }
  
  
      /**
       * 广度优先遍历
       *
       * @param s 起点
       * @param t 终点
       */
      public void bfs(int s, int t) {
          if (s == t) {
              return;
          }
          boolean[] visited = new boolean[v];
          visited[s] = true;
          Queue<Integer> queue = new LinkedList<>();
          queue.add(s);
          int[] prev = new int[v];
          //初始化prev
          for (int i = 0; i < v; i++) {
              prev[i] = -1;
          }
          while (!queue.isEmpty()) {
              //出栈
              int w = queue.poll();
              for (int i = 0; i < adj[w].size(); i++) {
                  int q = adj[w].get(i);
                  if (!visited[q]) {
                      prev[q] = w;
                      if (q == t) {
                          //遍历结束
                          return;
                      }
                      visited[q] = true;
                      queue.add(q);
                  }
              }
          }
      }
  
      // 是否找到
      private boolean found = false;
  
      /**
       * 深度优先遍历
       *
       * @param s
       * @param t
       */
      public void dfs(int s, int t) {
          boolean[] visited = new boolean[v];
          int[] prev = new int[v];
          for (int i = 0; i < v; i++) {
              prev[i] = -1;
          }
          recurDfs(s, t, visited, prev);
      }
  
      private void recurDfs(int w, int t, boolean[] visited, int[] prev) {
          if (found) {
              return;
          }
          visited[w] = true;
          if (w == t) {
              found = true;
              return;
          }
          for (int i = 0; i < adj[w].size(); i++) {
              int q = adj[w].get(i);
              if (!visited[q]) {
                  prev[q] = w;
                  recurDfs(q, t, visited, prev);
              }
          }
      }
  }
  ```

* 

### 算法思想



#### 贪心算法

* 分糖果

* 钱币找零

* 区间覆盖，这个和任务调度、排课比较相似

* 典型的背包问题

* Huffman编码，这个在数据压缩的时候是肯定会用到的

  这个其实就是构造一颗带有权重二叉树，这个权重，一般可以是字母出现的频率，显然，频率越低，那么这个字母编码就可以长一点。最后每一个字母都是根据路径来编码的

* 练习

  1: leetCode 402

  2: 假设有 n 个人等待被服务，但是服务窗口只有一个，每个人需要被服务的时间长度是不同的，如何安排被服务的先后顺序，才能让这 n 个人总的等待时间最短？

* 



#### 分治算法

分治算法就是将问题划分成n个规模较小，并且结构与原问题相似的子问题，递归的解决这些问题，然后合并其结果，就得到原问题的解。

但分治和递归还是有区别的，分治算法是处理问题的一种思想，递归是一种编程技巧。



#### 动态规划

* 练习

  leetCode 120, 96

  

* 

  