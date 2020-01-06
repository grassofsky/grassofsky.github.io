+++
date="2020-01-05" 
title="visual computing for med reading - pt2 chapter 6 表面渲染"
categories=["visual computing for med reading"] 

+++

# visual computing for med reading  - pt2 chapter 6 表面渲染

## 6.2 基于等高线的曲面重建

### *不同片层的等高线处理的时候需要解决哪三个问题？*

需要解决的问题如下：

- 对应问题。slice n中的潜在等高线数与切片n+1中的等高线相对应。
- 拼接问题。slice n中等高线Cn上的点，如何和slice n+1中等高线Cn+1上的点连接。
- 分支问题。如果slice n中的等高线Cn和slice n+1中的等高线Cn+1a和Cn+1b相对应，那么问题来了，等高线要如何被切分。

关于拼接问题，下图给出了直观的说明：

![](/images/visual_comput_for_med_fig6_2.png)

对于拼接问题，可以选择满足下面几个要求的结果：

- 体积最大化
- 表面积最小化
- 首选将切片n中的重心移动到接近切片n+1的重心的连接。

### *邻近点可以分为那两类？*

可以分类：

- 拓扑结构的邻近点，与对应顶点有相同连接数顶点。
- 满足欧几里德距离小于特定值的顶点。

### *表面网格的表现形式有哪几种，给出python实现？*

常见的网格表现形式有：

- SharedVertex 表示。如果表面网格仅用于渲染，不用于进一步的处理，倾向于适用内存消耗少的简单的数据结构。如果要查找两个三角面共享的边的时候，该数据结构并不支持直接查找，此时可以看后面的两种数据结构。
- Winged-edge 数据结构。该数据结构的介绍参见图6.4。
- Half-edge数据结构。半边数据结构将一条边从中间切割成了两个相反方向的部分。更多详细的介绍可以参见：https://blog.csdn.net/outtt/article/details/78544053，https://github.com/LLVirtual/SampleHalfEdge

sharedVertex的python实现示例：

```python
# Example cubic:
# front-top-left: 0.0, 1.0, 0.5
# front-top-right: 1.0, 1.0, 0.5
# front-bottom-right: 1.0, 0.0, 0.5
# front-bottom-left: 0.0, 0.0, 0.5
# back-top-left: 0.0, 1.0, -0.5
# back-top-right: 1.0, 1.0, -0.5
# back-bottom-right: 1.0, 0.0, -0.5
# back-bottom-left: 0.0, 0.0, -0.5
sharedVertex = [
    [0.0, 1.0, 0.5],
    [1.0, 1.0, 0.5],
    [1.0, 0.0, 0.5],
    [0.0, 0.0, 0.5],
    [0.0, 1.0, -0.5],
    [1.0, 1.0, -0.5],
    [1.0, 0.0, -0.5],
    [0.0, 0.0, -0.5]
]

# the six face of cubic is made of index of sharedVertex
cubicFaces = [
    [0, 1, 2, 3],
    [1, 5, 6, 2],
    [4, 5, 1, 0],
    [5, 4, 7, 6],
    [4, 0, 3, 7],
    [6, 7, 3, 2]
]
```

Winged-edge介绍的示例图如下：

![](/image/visual_computing_for_med_fig6_4.png)

Winged-edge数据结构的python示例如下：

```python
class Vertex:
    def __init__(self, x=0.0, y=0.0, z=0.0):
        self.x = x
        self.y = y
        self.z = z
        
class Face:
    def __init__(self):
        self.firstEdge = None # type WingedEdge
        
    def getAllEdges(self):
        if not self.firstEdge is None:
            edge = self.firstEdge
            nextEdge = edge.endLeftEdge if edge.leftFace == self else edge.endRightEdge
            edges = [edge]
            while nextEdge != self.firstEdge:
                edges.append(nextEdge)
                nextEdge = nextEdge.endLeftEdge if nextEdge.endLeftEdge == self else nextEdge.endRightEdge
            return edges
        else:
            return None

class WingedEdge:
    def __init__(self):
        self.startVertex = None       # type Vertex
        self.endVertex = None         # type Vertex
        self.startLeftEdge = None     # type WingedEdge
        self.startRightEdge = None    # type WingedEdge
        self.endLeftEdge = None       # type WingedEdge
        self.endRightEdge = None      # type WingedEdge
        self.leftFace = None          # type Face
        self.rightFace = None         # type Face
    
```

Half-edge数据结构的python实现：

```python
## TODO
```



## 6.3 Marching Cubes

*介绍针对2D的Marching Squares算法，并给出python实现？*

算法的相关介绍可以参见：https://blog.csdn.net/whuawell/article/details/74998280

![](https://img-blog.csdn.net/20170711204000996?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2h1YXdlbGw=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

算法的主要流程是：

- 遍历每个小网格，从16中固定情形中选择划线类别
- 利用线性插值，结合网格点数值找寻交点
- 化线

python代码实现如下，参照上面链接中的实现：

```python

```



## 6.4 没有分割的体数据的表面渲染



## 6.5 分割的体数据的表面渲染



## 6.6 高阶Mesh平滑



## 6.7 Mesh简化和基于web的表面渲染

