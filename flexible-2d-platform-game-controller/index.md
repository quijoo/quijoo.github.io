# List of 2D Platformer Controllers [23.2.17]

在我自己的 2D Platformer 项目中控制器手感非常僵硬，有一部分原因是缺少合适的角色动画，另一部分原因是控制器没有足够灵活。

然而在 Bilibili、Youtube 这样的视频网站教学中的控制器通常是简单的、不可拓展的、面向初学者的。

所以我希望通过收集各种有意思的控制器实现，并借鉴它们的优点，最终实现出一个自己的、有趣的、可拓展的控制器。 

## [2d-platformer-controller](https://github.com/akashenen/2d-platformer-controller)

![](https://raw.githubusercontent.com/akashenen/2d-platformer-controller/master/Gifs/demo2.gif)
这是一个 Unity2D 的控制器，作者不满足于 Unity2D 提供的物理特性，通过射线检测的方式实现自己的控制器。

包含如下特性：
1. 平滑精确的移动
2. 多段跳跃
3. 易于添加动画（学一下具体怎么实现的）
4. 单向平台
5. 墙跳/滑墙
6. 斜坡移动
7. 冲刺和空中冲刺
8. 梯子
9. 移动平台
10. 破碎平台
11. 存档点系统
12. 环境陷阱
13. 拾取物品
14. 压力板、操纵杆、其他触发器
15. 跳板

虽然这使用 Unity2D 实现，但是同样可以借鉴并在 Godot 中使用作用！这个控制器将许多 Controller 以外的功能放到 Controller 中实现了？
  
## [Unity Controller 射线检测技巧](https://www.gamedeveloper.com/design/the-hobbyist-coder-1-2d-platformer-controller)

大概讲了如何通过射线检测代替原本的2D物理，以及射线起点和角色边缘之间的间距是怎么工作的。

## [platforming-engine](https://robvansaaze.itch.io/platforming-engine)

![](https://img.itch.zone/aW1nLzI0MjIzOTAuZ2lm/original/lc3AGE.gif)
 
这个 控制器非常牛逼，超乎想象，这似乎就是奥日2Demo中的控制器！todo: 将这个控制器复刻到 Godot 上

主要学习以下实现：
1. 动画为什么这么流畅
2. 动画如何与物理配合的
3. 与环境交互时交互相关的代码如何组织的

虽然这是 GameMaker Studio 2 的实现，但是可以抄到 Godot 来

  
## [Ultimate-2D-Controller](https://github.com/Matthew-J-Spencer/Ultimate-2D-Controller)
这个是一个 Unity 的控制器，也相当流畅，但是完整版本要收费。
之后再看吧！


## 对于游戏控制器状态机的一些思考
之前实现过几个简单的状态机，但是它们都不便于扩展，这其中的原因是：*一旦改变了被控制对象的类型，状态机中对被控制对象的引用就失效了*

这意味着我们需要一种方法来将被控制对象和状态机解耦，通过 GodotObject 的反射功能可以做到这一点，但是这仍然要求被控制对象实现某些 属性/方法。

还有一种方法是通过 Blackboard 来交互，具体做法是：
1. 状态机以及所有状态类共享一个`Dictionary<string, Variant> blackboard` 对象.
2. 状态机只读写 balckboard 中的字段的值。状态机通过定义一组字段名来确定所需的功能。
3. 被控制对象需要在每一个物理帧同步 blackboard 的值。

这样整个被控制对象只需要实现同步各个属性的方法即可。其余的计算可以全部委托给状态机。并且为状态机添加新状态/新功能时 只需要添加 blackboard 的字段即可。

并且这种结构在分层状态机中同样便于实现。
