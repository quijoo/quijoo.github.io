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

### 线性约束的非线性规划问题
但是优化问题相关的第三方库都是使用迭代的数值优化方法来计算解, 这会导致算法复杂度可能无法预估。
所以这里我们用一个朴素的算法来解决线性约束的非线性规划问题, 算法分成两步:

由于所有的约束都是线性约束, 所以可行域一定是多边形区域(或者开放的凸多边形区域)
最优解的点只会取约束直线上某点或者顶点(不会取可行域内的点)
如何计算 **可行域边界** ？
1. 每个约束条件与其余约束条件计算公共区域(`Intercepted` 方法)
    * Tips1: 存在线段和射线约束, 需要检查交点是否在线段或者射线上
    * Tips2：直线的法线代表可行域区域, 可以通过两条直线的**方向** 和 **法线方向**判断两线相交分割出的四条射线哪两段是有效的 (见 `Intercepted` 中判断 `Direction` 和 `Normal` 夹角 <= 90>)
        {{< figure src="/images/line_cast.jpg" title="线段剪切" >}}
    * Tips3: 线的相交只会缩短端点, 所以需要进行 Max/Min 判断(见 Intercepted)
    * Tips4: 那些 左端点 >= 右端点 的边界视为无效边界( 见`SolveCollition` 和 `Line.IsValid()`)
        更深层的意思是, 出现无效边时, 可能有以下几种情况(但是只要去除所有无效边即可获得正确的可行域)
        {{< figure src="/images/enum_line_cast.png" title="线段剪切枚举" >}}

2. 得到一个边界集合(元素可能是 直线/线段/射线), 其中某些边界有效, 某些边界无效
3. 对边界集合中所有有效的边界, 计算其与 O(0,0) 点的距离, 并取最小距离，最小距离仅可能以下情况：
    * O 向边界线作垂线的垂足(需要保证垂足再 minT 和 maxT 约束之间)
    * 边界射线端点
    * 边界线段断电
    所以, 只需要遍历所有的边界线垂足和边界顶点, 取与 O 最近的点即可 (见 SolveCollition 第二个 for 循环)
该算法复杂度为 $O(n^2)$, 比计算出所有交点, 再检查焦点是否在可行域内的方法($O(n^3)$)更快
```c#
public class Line
{
    public Vector2 Point;
    public Vector2 Normal;
    public Vector2 Direction;
    public float minT = float.MinValue;
    public float maxT = float.MaxValue;
    public Line(Vector2 _p, Vector2 _n)
    {
        Point = _p; Normal = _n;
        Direction = new Vector2(Normal.y, -Normal.x);
    }
    public bool IsRay() => (minT == float.MinValue && maxT != float.MaxValue) || (minT != float.MinValue && maxT == float.MaxValue);
    public bool IsLine() => minT == float.MinValue && maxT == float.MaxValue;
    public bool IsSegment() => minT != float.MinValue && maxT != float.MaxValue;
    public bool IsValid() => minT <= maxT;
    public Vector2 GetRayPoint()
    {
        if (minT == float.MinValue && maxT != float.MaxValue)
        {
            return Point + maxT * Direction;
        }
        if (minT != float.MinValue && maxT == float.MaxValue)
        { 
            return Point + minT * Direction;
        }
        return Point;
    }
    public Vector2 GetRayDirection()
    {
        if (minT == float.MinValue && maxT != float.MaxValue)
        {
            return -Direction;
        }
        if (minT != float.MinValue && maxT == float.MaxValue)
        {
            return Direction;
        }
        return Vector2.Zero;
    }
    public void Perpendicular(Vector2 point, out Vector2 foot, out float distance)
    {
        Line pLine = new Line(point, this.Direction);
        Intersect(this, pLine, out float t1, out float t2);
        foot = pLine.Point + pLine.Direction * t2;
        if(t1 <= maxT && t1 >= minT) distance = (foot - point).Length();
        else distance = float.MaxValue;
    }
}
static public void Intersect(Line B1, Line B2, out float t1, out float t2)
{
    Vector2 P1 = B1.Point; Vector2 P2 = B2.Point;
    Vector2 D1 = B1.Direction; Vector2 D2 = B2.Direction;
    t2 = (D1.y * (P1.x - P2.x) - D1.x * (P1.y - P2.y)) / (D2.x * D1.y - D1.x * D2.y);
    t1 = (P2.x + t2 * D2.x - P1.x) / D1.x;
}
static public void Intercepted(ref Line B1, ref Line B2)
{
    Intersect(B1, B2, out float k1, out float k2);

    if (B1.Direction.Dot(B2.Normal) >= 0) B1.minT = Mathf.Max(k1, B1.minT);
    else B1.maxT = Mathf.Min(k1, B1.maxT);

    if (B2.Direction.Dot(B1.Normal) >= 0) B2.minT = Mathf.Max(k2, B2.minT);
    else B2.maxT = Mathf.Min(k2, B2.maxT);
}
static public Vector2 SolveCollition(Line[] lines)
{
    for(int i = 0; i < lines.Length; i++)
    {
        for (int j = 0; j < i; j++)
        {
            Intercepted(ref lines[i], ref lines[j]);
        }
    }
    float distance = float.MaxValue; Vector2 Selected = Vector2.Zero;
    foreach (var line in lines)
    {
        // 跳过无效边
        if (!line.IsValid()) continue;
        // 计算线上垂点
        line.Perpendicular(Vector2.Zero, out Vector2 f, out float d);
        if (d < distance) { Selected = f; distance = d; }
        // 计算射线起点
        if (line.IsRay())
        {
            Vector2 rayPoint = line.GetRayPoint();
            if (rayPoint.Length() < distance) { Selected = rayPoint; distance = rayPoint.Length(); }
        }
        // 计算线段端点
        if (line.IsSegment())
        {
            Vector2 L = line.Point + line.minT * line.Direction;
            Vector2 R = line.Point + line.maxT * line.Direction;
            if (L.Length() < distance) { Selected = L; distance = L.Length(); }
            if (R.Length() < distance) { Selected = R; distance = R.Length(); }
        }
    }
    if (distance != float.MaxValue) return Selected;
    else return Vector2.Zero;
}
```


**最后一个问题** 如何将线性规划的约束条件转化为 (Point, Normal) 的形式？

设 约束条件为 $ax + by \ge c$, 边界上一点 $(x_0, y_0)$

边界直线的法线方向有 $(a, b)$ 和 $(-a, -b)$ 两个方向

可以计算得到边界直线外一点 $(x_1, y_1) = (x_0 + a, y_0 + b)$, 将该点带入直线方程
$$
a (x_0 + a) + b(y_0 + b) = (ax_0 + by_0) + a^2 + b^2 = c + (a^2 + b^2) \ge c
$$
所以 $N(a, b)$ 是该边界指向内部的法线

**无解条件**
* 如果所有 QP 向量分解到 x, y 轴上并同时有 $x^-, x^+, y^-, y^-$ 方向的移动, 那么是不可解的
* 拉格朗日乘数法无解 

**问题** 解决一次碰撞后可能产生新的碰撞,如果新的碰撞和原有的碰撞是冲突的那该怎么办？

**解决** 暂时没想到什么更好的方法, 就顶一个数字m, 每次都进行 m 次移动, 如果找不到安全位置就判定为失败即可。

### 方法三(网格划分 + 碰撞解算)
这是一个启发式的方法(没什么依据,乱想的):

1. 使用网格划分, 记每个格点上的所有 QP 的长度之和为 $W = \sum_i |QP_i|$

2. 取 W 值最小的格点并在此处执行碰撞解算

## 总结
实际上由于不知道物体周围的全部碰撞信息, 碰撞解算并不能避免掉未知的碰撞, 一次碰撞解决后可能引入新的碰撞, 所以基于解算的方法在不知道全局碰撞信息的情况下是不可靠的。

而基于迭代的解算完全不需要使用解算得到的移动方向, 只需要使用当前所有碰撞法线的求和得到移动方向。

所以这篇博客实际上**没什么用**, 唯一的作用可能是O(n^2)的解决了一个特定的线性规划问题。 确实没什么用？
