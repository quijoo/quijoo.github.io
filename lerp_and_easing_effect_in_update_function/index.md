# Lerp 函数与递推缓动

## 从问题出发
当我在实现 2D 平台游戏中 **WallJump** 时遇到了麻烦，角色的水平移动是关于输入的函数，即:
```c#
void Update()
{
    Velocity.X = input_map(HorizontalInput);
    MoveAndSlide();
}

```

而 **WallJump** 需要赋予玩家一个 **斜向初速度**，此时的水平速度会被 **HorizontalInput** 覆盖

```c#
[Export] float WallJumpVelocity;
void WallJump(Vector2 direction)
{
    Velocity = dir * WallJumpVelocity.X + Vector2.Up * WallJumpVelocity.Y;
    MoveAndSlide();
}
```

最终的效果就是玩家只会产生向上的移动。S有一个简单的技巧可以回避这个问题，在 **WallJump** 发生的一瞬间启动一个 **计时器** 计时结束前屏蔽玩家输入。这能解决问题，但是不够优雅和一致，这是在回避问题而不是解决问题, 这个问题的根源是: 

> **当我们输入时是否应该立即作用于角色**

于是我采用 **缓动** 的方式改变使 **HorizontalInput** 延迟作用到 **Velocity**


这个视频作者的思路和我大致相同，但并非使用自定义的缓动曲线，而是使用时常出现 Unity 教程中的方法:

{{< youtube 2S3g8CgBG1g >}}

```c#
void Update()
{
    // horizontal move
    var targetVelocity = input_map(HorizontalInput);
    Velocity = Lerp(Velocity, targetVelocity, k);
    MoveAndSlide();
}
```

这是一个 **幂函数($y=a^x$)** 缓动曲线, 初学者通常误以为是线性插值而对 Lerp 函数产生误解, 我后面会解释原因。


## 该问题引发的思考
为什么使用 Lerp 插值能解决问题呢？采用自定义缓动的曲线为什么不行呢？

#### 分析一: 自定义缓动曲线方法
通常自定义的曲线需要确定 **起始值(s)**、**终点值(f)**、**插值变量(p)** 从而在每帧插值出当前需要的值并更新。例如:

```c#
private Vector2 cachedVelocity;
private float p;
void Update(float delta)
{
    
    var targetVelocity = input_map(HorizontalInput); // 获取预期的水平速度
    Velocity = Lerp(cachedVelocity, targetVelocity, p);
    MoveAndSlide();

    p = Mathf.Clamp(t + delta, 0f, 1f); // 更新插值变量(t)
}
void Start()
{
    cachedVelocity = this.Velocity;
    p = 0f;
}
```

这种方法中 **当前值(c)** 不参与计算，而是 **插值变量(p)** 在控制插值过程，这会导致一个问题： 
> **当前值(c)** 和 **终点值(f)** 并非固定, 例如 **WallJump** 触发时会直接修改 `Velocity.X`, `HorizontalInput` 也随着输入改变而改变。
>
> **终点值(f)** 发生连续变化时, **插值变量(p)** 不变时, 插值结果可能会产生巨大突变。（因为自定义曲线的斜率是无法预知的）
>
> **当前值(c)** 发生突变时, 需要重新启动一个插值过程 即：重置插值的 **插值变量(p = 0)** 和 **起始值(s = c)**

#### 分析二: 递推的 Lerp 方法
而递推的 Lerp 的方法是利用 **当前值(c)** 代替 **起始值(s)**, 用户输入作为 **目标值(f)**， 这样不论如何 速度都会平滑的改变：
```c#
void upadte()
{
    var targetVelocity = input_map(HorizontalInput);
    Velocity = Lerp(Velocity, targetVelocity, 0.1f); // 注意这里 p 固定为 0.1, 后面会讨论
}
```
> **终点值(f)** 发生连续变化时, 插值结果是通过 Lerp 函数以 **起始值(s)** 为起点插值的, 所以一定是连续的。
>
> **当前值(c)** 发生突变时, 插值仍然是从 **起始值(s)** 开始的, 所以仍然是连续的。
>
> 上述两种情况保证了 Velocity 的 **连续性**


这种不依赖 **插值变量(t)** 的 **无状态迭代方法** 很有意思。于是我跳脱出这个问题，开始思考该方法更加一般的应用。

## 无状态递推缓动函数
首先我们思考 $c_{n+1} = lerp(c_n, f, 0.1)$ 的本来面貌！

**Lerp 函数定义**
我们只考虑 `float` 插值的情况，其他多维变量大同小异
```c#
float Lerp(float s, float f, float p)
{
    if(p > 1) return f;
    if(p < 0) return s;
    return (1 - p) * s + p * f;
}
```

显然这是一个线性插值， 但是将其应用于缓动时， **起始值(s)** 传入的是**当前值(c)**，所以这个过程是 **递推的** 用数列来表达缓动过程就是：

$$
x_{n} = (1 - p) \cdot x_{n-1} + p \cdot f
$$

求解 $x_{n}$ 通项公式是一个简单的高中数列问题：
$$
x_{n}   - f = (1-p)\cdot (x_{n-1}     - f) \\\
x_{n-1} - f = (1-p)\cdot (x_{n-2} - f)\\\
...\\\
x_{1} - f = (1-p)\cdot (x_{0} - f)\\\
$$

累乘可得 $x_n = (1-p)^n \cdot (x_0 - f) + f$

这里的变量的含义是累计迭代的次数 $n \in (1, +\infty)$ 而缓动函数的变量是 $t \in (0, 1)$, 我们假设每帧的更新时间是 $\triangle t$, 可以得到 $n = \frac{t}{\triangle t}$

从而: $g(t) = x_{\frac{t}{\triangle t}} = (1-p)^{\frac{t}{\triangle t}} \cdot (x_0 - f) + f$

至此我们得到了 **递推式** $x_{n} = (1 - p) \cdot x_{n-1} + p \cdot f$ 的 **函数式** $g(t) = x_{\frac{t}{\triangle t}} = (1-p)^{\frac{t}{\triangle t}} \cdot (s - f) + f$

当 `s = 0, f = 1, p=[0.01 -> 0.08]` 时该函数图像如下图（p 越大 曲线弯曲程度越大）

<iframe src="https://www.desmos.com/calculator/vo8dsr0q1k?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

当 `s=0, f =[1.0 -> 1,4], p = 0.05` 时 该函数曲线如下 （f 越大曲线渐进线越高）

<iframe src="https://www.desmos.com/calculator/nvw2xzpxs6?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

### 曲线分析

#### 当前值(c) 的稳定性
对于我们的需求，最佳的曲线是，当 **终点值(f)** 发生变化时 **当前值(c)** 不发生改变。但是遗憾的是这样的曲线并不存在。证明很简单，略。

既然 **c** 不可能不发生变化，那么在 **f** 变化时保证 **c** 平滑变化也是可以接受的。

计算 $g(t, f)$ 对 $f$ 的偏导数:
$$
\frac{\partial g}{\partial f} = 1 - (1 - p)^{\frac{t}{\triangle t}}
$$
其中 $p, \triangle t$ 几乎是不变的, 所以对于某时刻 $t$, **当前值(c)** 随 **终点值(f)** 的变化而平滑变化

#### 当前值(c) 的变化速率
由于 $g(t)$ 是一个以 $y=f$ 为渐近线的渐进曲线。所以 我们可以计算 $\delta = |g(t) - f| = (1-p)^{\frac{t}{\triangle t} \cdot (f -s)}$ 来表示值得接近程度,

$(1-p)^{\frac{t}{\triangle t}}$ 表示,随着越来越接近 **终点值(f)** 变化将越来越缓慢, 所以该曲线影响 渐进速度的量是 $(1-p)^{\frac{t}{\triangle t}$ 项

当 `s = 0, f = 1, p = 0.83, dt = [0.1 -> 0.7]` 是如下图：（$\triangle t$ 越大曲线弯曲程度越小）

<iframe src="https://www.desmos.com/calculator/nslvhpefyz?embed" width="500" height="500" style="border: 1px solid #ccc" frameborder=0></iframe>

可见 $p$ 和 $\triangle t$ 都会影响曲线的**弯曲程度(渐进时间)**, 并且 影响的方向相反(一个变平滑一个变弯曲)

可见 $(1-p)$ 和 $\triangle t$ 是影响响应时间的主要部分(通过分析偏导数可以得到), 在游戏引擎中 $\triangle t$ 在物理帧中通常是常数, 所以只能通过调整 $p$ 的值来调整曲线的响应时间

这里产生了一个新问题：为了使 120fps($\triangle t = \frac{1}{120}s$) 的游戏和 60fps($\triangle t = \frac{1}{60}s$) 的游戏中的缓动具有相同的响应时间, 所以需要约束 $p$ 与 $\triangle t$ 的关系。


#### $p$ 与 $\triangle t$ 的约束

为了描述渐进曲线的接近程度, 上述分析中定义了 $\delta = |g(t) - f| = (1-p)^{\frac{t}{\triangle t}} \cdot (f -s)$ 来描述渐进程度

我们再定义一个小量 $\epsilon \cdot (f -s)$, 当 $\delta = |g(t) - f| = (1-p)^{\frac{t}{\triangle t}} \cdot (f -s) < \epsilon \cdot (f -s)$ 表示缓动完成。作以下变形：

$$
(1-p)^{\frac{t}{\triangle t}} < \epsilon \\\
\quad\\\
t > \frac{\triangle t \cdot \ln \epsilon}{\ln(1-p)}
$$

所以当 $t > t_{max}$ 时缓动结束， 可以根据该约束得到 $p$ 与 $\triangle t$ 的约束关系：

$$
p = 1 - \epsilon ^{\frac{\triangle t}{t_{max}}}
$$

所以最初的式子演变为：

$$
g(t, t_{max}, \epsilon) = (\epsilon ^{\frac{\triangle t}{t_{max}}})^{\frac{t}{\triangle t}} \cdot (s-f) + f = \epsilon^{\frac{t}{t_{max}}} \cdot (s - f) + f
$$

该曲线有以下特性:
1. 曲线的响应速度与 $\triangle t$ 无关
2. 通过控制参数 $t_{max}$ 控制响应速度
3. 通过设置 $\epsilon$ 改变 $t = t_{max}$ 的渐进程度
4. 在 `Update` 调用中使用方便
    ```c#
    void Update(float delta)
    {
        var targetVelocity = input_map(HorizontalInput); // 获得期望速度
        float tMax = 0.5f; // 响应时间
        float p = 1 - Mathf.Pow(Mathf.Epsilon, delta / tMax); // 计算上述的 p 值
        Velocity = Lerp(Velocity, targetVelocity, p);   // 线性插值
        MoveAndSlide();
    }
    ```

#### 代码测试
```python
import matplotlib.pyplot as plt
import numpy as np
import copy
import math
# params
f = 1
s = 0
epsolon = 0.001
dt = 0.01
t_max = 0.5

# figure setting
fig = plt.figure()
axes = fig.add_axes([0.1, 0.1, 0.8, 0.8])

# generate x sequence
t = np.linspace(0, 1, int(1/dt))

# pre-compute `p`
p = 1 - math.pow(epsolon, dt / t_max)

# g(t) = (1-p) ^ (t/dt) * (s-f) + f =  epsolon ^ (t/t_max) * (s-f) + f
y1 = epsolon ** (t / t_max) * (s - f) + f

# simulation game engine's update 
y, tmp = [], copy.deepcopy(s)
for ti in t:
    tmp =  (1 - p) * tmp + p * f
    y.append(tmp)

# draw call
axes.plot(t, np.array(y), c="green", label="y=h(x)", ls='-.', alpha = 1, lw=2, zorder=1)
axes.plot(t, y1, c="blue", label="y=lerp", ls=':', alpha = 0.5, lw=2, zorder=1)
axes.legend()
plt.show()
```
![](/images/python_curve.png)
## 更进一步的思考

使用 Lerp 插值的缓动函数过于单一，如果我们需要使用更加灵活多样的缓动函数，并且要将他们改变为 无状态 递推的方法应该要怎么做呢？

例如 我们的目标缓动函数是 $func(t) = t^2$


**第一步**

$func(t + \triangle t) = (t + \triangle t) ^ 2$


**第二步**

假设存在一个迭代函数 $h(\cdot)$, 并且满足 $func(t + \triangle t) = h[func(t)]$

这里的 $h(\cdot)$ 相当于原方法里的 Lerp 


**第三步**
为了得到 $h$ 的解析形式 令 $k=func(t), \quad t=func^{-1}(k)$

带入原式 $h(k) = func(func^{-1}(k)+\triangle t) = (\sqrt{k} + \triangle t)^2$, 所以只要 $func$ 的反函数存在就一定能计算得到 $h(k)$
```c#
float h(float k)
{
    return g(g_inverse(k) + 1);
}

void Update()
{
    position = h(position);
}
```

然后如果要限制响应时间，变化程度的话，参考前文的方法吧！本来还想做一个统一的参数化方法的，但是累了！

问题就解决啦，我真是个小天才，卧槽。

