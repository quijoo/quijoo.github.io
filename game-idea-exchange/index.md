# [游戏机制] 交换球


<!-- {{< bilibili BV1Sv411M7LP >}} -->

## 灵感来源

前段时间玩了 [传送门2](https://wiki.teamfortress.com/wiki/Portal_2/zh-hans), 想到一个新的游戏机制
> 玩家A可以发射一个**交换球**，交换球碰撞到特定的**可交换物体** B后交换 A 和 B 的位置

该机制可以用来设计同传送门相似的各种奇思妙想的关卡，需要将整个机制分解为以下两个机制
1. 移动场景中的物体
2. 玩家获得位置移动

## 需要解决的问题？
* 上文中 A 和 B 的**体积** 和 **碰撞 Layer/Mask** 都是不一样的，如何保证交换的物体在交换后不会与场景中已有的物体发生冲突呢？
* 如何进行碰撞测试？

## 如何进行有效的冲突测试？
引擎提供的冲突检测仅给出碰撞点和碰撞方向, 要实现将物体放置到安全位置还需要知道冲突重叠区域在碰撞法线方向上投影的最大宽度(将物体平移到恰好不碰撞的位移)

所以我们定义一个 **恢复偏移向量**: 
> 假设物体 A 和环境碰撞体 B 发生碰撞, 碰撞点为 P, 碰撞法线为 N, 那么以 P 点为起始点方向为 -N 发射射线, 与 A 的碰撞体积计算几何交点 $Q_i$ 取最长的 $Q_iP$ 
QP, QP 则是**恢复偏移向量**

按照 QP 移动物体能解决 A/B 的冲突但是可能引入新的冲突, 所以需要多次进行 冲突测试-避免冲突

**几何交点**
```c#
public static List<Vector2> GeometryIntersection(Polygon polygon, Ray ray)
{
    List<Vector2> res = new List<Vector2>();
    foreach(Segment segment in polygon.GetSegments())
    {
        Vector2 intersection = GeometryIntersection(segment, ray);
        if(intersection != null)
        {
            res.Add(intersection);
        }
    }
    return res;
}

public static List<Vector2> GeometryIntersection(Segment segment, Ray ray)
{
    Vector2 A = segment.Start;
    Vector2 B = segment.End;
    Vector2 P = ray.Start;
    Vector2 D = ray.Direction;
    float k = ((B.x - P.x) * D.y - (B.y - P.y) * D.x) / ((B.x - A.x)*D.y - (B.y - A.y) * D.x);
    float t = (B.x - P.x - k * (B.x - A.x)) / D.x;
    if(k >= 0 && k <=1 && t >= 0) return P + t * D;
    else return null;
}
```

## 解决碰撞冲突(网格搜索)

目前已知物体 A / B, 要交换它们的位置，当某个物体改变位置后如果发生碰撞，我们可以**微调**目标位置，使其以最小的偏移实现 **无碰撞交换**

但是微调是一个很暧昧的概念，什么叫微调，移动多远的距离算微调，如果对微调距离不加限制，那么 A/B 呆在原地也恰好不发生碰撞，所以限制微调距离是必要的

{{< admonition type=info title="Tip" open=true >}}
交换指 *位置互换*，微调指在一定的 *容忍范围* 内移动目标位置
{{< /admonition >}}

所以我们可以得到一个朴素的假算法
> 假定有目标位置 $P(x_0, y_0)$, 容忍距离 $d$, 有点集:
$$
D = \lbrace (x, y) | \sqrt{(x-x_0)^2 + (y-y_0)^2} < d \rbrace
$$
搜索点集中距离 P 最近并且无碰撞的点作为新的目标位置

上述算法存在几个问题
* 点集 D 是无穷集合无法枚举
* 环境的碰撞不是有序的(无法简单的二分)

### 方法一(网格枚举)
可以使用网格划分 D ，然后由P点到边缘的顺序枚举每一个网格点，并对点其进行 **碰撞测试**，找到第一个无碰撞的点 记划分粒度为 k, 算法复杂度为 $O(\frac{d^2}{k^2})$
(可以使用QP 向量来减小碰撞测试的次数)

**缺点** 网格碰撞, 可能因为空间的整数划分而找不到安全点(可能实际上有安全点)
**解决** 配合碰撞解算(方法三)

### 方法二(碰撞解算)
> 在该方法中, 我们将物体在某一位置的所有碰撞简化为与多条直线的碰撞(直线与法线 N 垂直,并且过点 P)
> 对于任意碰撞 $c_i$, 可以计算碰撞的 $(QP)_i$ 向量, 我们移动物体解决碰撞的本质是 计算一个 $M_i$ 向量, 该向量在 $(QP)_i$ 方向的投影大于等于 $|(QP)_i|$


$$
\frac{\vec{M} \cdot \vec{QP}}{|\vec{QP}|} \ge |\vec{QP}| \\\
\quad\\\
\Rightarrow \vec{M} \cdot \vec{QP} \ge |\vec{QP}|^2
$$



此外还需要满足 M 尽可能地小的约束(确保靠近选中的位置)
所以对于所有的碰撞点:

$$
\begin{cases}
\vec{M} \cdot \vec{QP_i} \ge |\vec{QP_i}|^2,\quad i = 1,2,3,...\\\ 
\\\
\arg \mathop{\min}\limits_{M} |\vec{QP_i}|
\end{cases}
$$

我们观察上式, 所有的 $QP$ 是已知的, 每一个不等式约束都是一个线性约束设  $M(x, y)$, 每一个约束都是一个关于 $x, y$ 的不等式
所以该问题就转化为了熟悉的不等式约束的优化问题, 优化目标是 $M$ 位移的距离

{{< admonition type=info title="Tip" open=true >}}
可以使用[带 KKT 条件的拉格朗日乘数法](https://blog.csdn.net/qq_41133375/article/details/105642352)来解 M
{{< /admonition >}}

**无解条件**
* 如果所有 QP 向量分解到 x, y 轴上并同时有 $x^-, x^+, y^-, y^-$ 方向的移动, 那么是不可解的
* 拉格朗日乘数法无解 

**问题** 解决一次碰撞后可能产生新的碰撞,如果新的碰撞和原有的碰撞是冲突的那该怎么办？

**解决** 暂时没想到什么更好的方法, 就顶一个数字m, 每次都进行 m 次移动, 如果找不到安全位置就判定为失败即可。

### 方法三(网格划分 + 碰撞解算)
这是一个启发式的方法(没什么依据,乱想的):

1. 使用网格划分, 记每个格点上的所有 QP 的长度之和为 $W = \sum_i |QP_i|$

2. 取 W 值最小的格点并在此处执行碰撞解算
