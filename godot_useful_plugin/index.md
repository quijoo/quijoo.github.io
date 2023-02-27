# [Godot4.0] 实用插件搜索计划 第一期


godot4.0 rc 版本发布后，新插件如雨后春笋的涌现
很多插件我还没有尝试，但是很多感兴趣的插件，列表如下

## [Panku Console - *A in-game console*](https://github.com/Ark2000/PankuConsole)
![](https://raw.githubusercontent.com/Ark2000/PankuConsole/master/assets/preview.webp)

**特性如下：**
* 漂亮灵活的多窗口系统
* 简单开箱即用
* 创建窗口实时检测表达式的值
* 绑定表达式到key
* Popup Notification
* 系统debug 信息
* Powerful Inspector Generator（游戏内显示export变量）
* 历史表达式管理

这个真有点猛吧, 很多In-game 的控制台都是侵入式的，需要修改很多游戏代码，但是这个插件利用 `GodotObject` 的 `call(name, params[]) / get(name) / set(name, value)` 方法监控变量，计算表达式，调用方法！


{{< admonition type=info title="Tip" open=true >}}
[2023.2.24] 使用情况：设计很方便，主要功能是能在游戏运行时监控对象，但是 godot 本来就有类似的性能监视器功能只是是在编辑器界面的。
对于屏幕小而不能同时到编辑器和游戏窗口的情况很方便。

但是在使用过程中遇到了恶性BUG：
1. 监控的对象一旦被销毁，则会触发空引用报错 。。。
2. 有些时候会出现狂掉帧（原因不明，但是关掉这个插件就恢复了）
{{< /admonition >}}


## [Softbody 2D - *A squishy shapes generator*](https://github.com/Ughuuu/godot-4-softbody2d)
在 Unity3D 中有很多方法使用软体物理(Juice), 但是 Godot4.0 中仅提供了 3D 软体，而没有2D 软体

虽然可以手动的通过 `Rigidibody2D`， `Spring` 等组件来构造弹性物体，但是 弹簧和钢体之间的连接方式以及数值设置非常令人头疼， 这个插件可以基于 Polygon2D 生成器来生成**多边形轮廓**、**内部顶点**、**骨骼**、**刚体**

{{< youtube 7n8giBCOX3Q >}}

## [VP-GAMES's series of plugins](https://github.com/VP-GAMES)

* [Quest Editor(G4)](https://github.com/VP-GAMES/Godot4QuestEditor) : 一个任务编辑器
* [Dialogue Editor(G4)](https://github.com/VP-GAMES/DialogueEditor) : 对话编辑器
* [Inventory Editor (G4) ](https://github.com/VP-GAMES/InventoryEditor): 库存系统
* [Localization Editor (G4)](https://github.com/VP-GAMES/Godot4LocalizationEditor) : 本地化编辑器

这个没了解, 但是看起来不错，之后可以试试 库存和本地化管理器

## [Simple Grass Textured - *A 3D grass editor*](https://github.com/IcterusGames/SimpleGrassTextured)
一个简单的3D草地编辑器，主要功能就是种草，我应该用不上，但是很好奇是怎么实现在草地上种草的(没写过3D 项目)

{{< youtube E_LTrFUfrrw >}}

## [Compute Worker - *A simple compute node*](https://github.com/wardensdev/ComputeWorker)
这是一个比较有趣的东西， 一个支持 GPU 的运算节点，可能哟点类似各个游戏引擎中的 ComputeShader，用于计算大量数据（例如数以千/万计的动画），该节点简化了数据从内存拷贝到缓冲区的过程（大概吧，只看了简介，没用过），想要通过这个项目了解 Godot 的 ComputeShader 如何工作的/

## [Gameplay Attributes](https://github.com/OctoD/godot-gameplay-attributes)
游戏属性是一组用于描述某些节点 使用戈多制作的 2D 和 3D 游戏的角色属性。

## [Controller Icons - *A controller UI*](https://github.com/rsubtil/controller_icons/)
这是一个简单的支持各种控制器的 Controller UI 插件，可以方便的在频幕上显示提示 UI 等
**支持列表**
Xbox 360
Xbox One
Xbox Series
PlayStation 3
PlayStation 4
PlayStation 5
Nintendo Switch Controller
Nintendo Switch Joy-Con
Steam Controller
Steam Deck
Amazon Luna
Google Stadia

## [Godot RL Agents - *A reinforcement learning framework*](https://github.com/edbeeching/godot_rl_agents_plugin)

AI  Agent 插件, 这个好像有点牛逼，一个强化学习框架，提供了很多功能。
**功能如下**
* 提供 在Python 中运行机器学习算法的接口
* 提供三个著名 RL 框架包装器: **StableBaselines3**、**Sample Factory** 和 **Ray RLLib**
* 支持 记忆模型、LSTM、注意力模型
* 支持 2D / 3D 游戏
* 一套 AI 传感器用于增强 Agent 对于游戏世界的感知

## [Smoother - *A physics smoother node*](https://github.com/anatolbogun/godot-smoother-node)

用于平滑运动的节点，即使在真率很低的情况下，也能有等同于屏幕刷新率的效果！

虽然但是，有以下缺点：
1. 适用于刚体，不适用于复杂节点
2. 插值落后真实移动
3. 没有 Look Forward

效果真挺好！
![](https://user-images.githubusercontent.com/7110246/209624079-86824089-444d-4f6e-bd02-b2b38e3952c4.gif)


## [Beehave - *A full-featured behaviour tree*](https://github.com/bitbrain/beehave)

![](https://raw.githubusercontent.com/bitbrain/beehave/godot-4.x/assets/logo.svg)

这是 [github](www.github.com) 上标星最多的 godot 行为树项目， 相比其他很多行为树插件，该插件看起来文档详细，并且作者自己也在使用并且也会经常更新(比起很多行为树插件bug多，几乎没人维护，这个应该是唯一能用的了)

行为树的树结构是借用了 Godot 的 `SceneTree`, 没有实现基于 `GraphyNode` 的节点编辑器, 但是基本节点和功能都覆盖了。

行为树相关知识可以参考 [Godot Behavior Tree Example](https://github.com/viniciusgerevini/godot-behavior-tree-example)
{{< youtube YHUQY2Kea9U >}}

## [Dialogue Manager - *The bestest dialogue manager*](https://github.com/nathanhoad/godot_dialogue_manager)

这是一个简单的对话管理器插件，功能简单只专注于**对话内容** 和 **对话过程** 的管理（对话过程可以调用方法，类似于godot 动画中调用一样）

**简洁**, 没有多余的 GUI，仅有一个用于编写 .dialogue 文件的编辑器

**高效**, 对话采用**文本编辑**，类似代码中的流程控制，一个 .dialogue 文件甚至可以 import 其他的dialogue，这很符合程序员的想法（hhhh

**独立**, 它是一个完全独立的系统，仅仅提供 `DialogueManager` 的数个 API ，对原本游戏系统几乎"零"侵入。

正因为它足够简单和高效，拓展或者实现自定义 UI 都非常简单！
  
![](https://raw.githubusercontent.com/nathanhoad/godot_dialogue_manager/main/docs/hero.png)

{{< admonition type=info title="Tip" open=true >}}
[2023.2.24] 使用情况：很简单易用，暂时没有发现 BUG

{{< /admonition >}}



几乎把 godot 4.0 的插件列表翻了一遍，对上边这些比较感兴趣，并且打算 **Panku Console** 和 **Beehave** 应用到自己的项目中！

