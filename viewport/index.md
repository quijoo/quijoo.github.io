# [godot4] Viewport && CanvasLayer && Camera && SCREEN_TEXTURE


官方文档中，Viewport 、 CanvasLayer、Camera 是比较基本的概念，但是文档中解释得并不清楚（可能是翻译导致的

## `CanvasItem`
所有的 Node 和 Control 都继承自 CanvasItem, 每个 CanvasItem 都继承其父节点的 Transform

所有被显示的 CanvasItem 都是Viewport 的直接或间接子节点。

移动整个场景，应该改变 Viewport 的Transform 而不是场景的 Node

## `CanvasLayer`
![](https://docs.godotengine.org/en/latest/_images/canvaslayers.png)
比如我们希望 HUD / UI 的屏幕位置不发生改变，就可以为 UI 创建一个单独的 CanvasLayer，能够让某层独立的渲染它通过 Layer 数字规定了**渲染的优先级**，这很重要。

CanvasLayers 的渲染顺序与树结构无关，只依赖于层编号 `layer`.

也就是说，树上存在两个有不同 `layer` 、且是直接或间接父子关系的 CanvasLayer，其渲染关系/顺序不依照树结构，而是 `layer` 。

`layer` 越大，越后绘制，在图层越上面！


## `Camera2D`

一个 Viewport 中仅存在一个被激活的 Camera2D, 这意味着一个 Viewport 仅为一个相机显示画面。

3D中情况不同，可以使用多个相机对世界进行观察，并将其中一些映射到纹理。

那么2D 下如何实现这种功能呢？例如我们需要做一个 2D 的双人同屏游戏该怎么做呢？

Viewport 所观测的世界由资源对象 `World2D` 控制, 如果两个Viewport 的 `World2D` 指向同一个对象，他们的相机在 `World2D` 的不同位置，那么它们的 Viewport 所绘制的画面也是不同的。例如双人同屏游戏可以按如下结构：
```
* root
    * WorldNode
    * CanvasLayer
        * HBoxContainer
            * SubViewportContainer(Left Screen)
                * SubViewport
                    * Camera_left
            * SubViewportContainer(Right Screen)
                * SubViewport
                    * Camera_right

let LeftSubViewportContainer.SubViewport.World2D = root.GetViewport();
let RightSubViewportContainer.SubViewport.World2D = root.GetViewport();
```

这种结构是侵入性的，它需要修改整个项目的树结构。有时我们只想为场景添加一个 Mini Map UI 怎么办呢？可以使用以下结构

```
* root
    * WorldNode
        * ...
        * MainCamera
    * CanvasLayer
        * SubViewportContainer
            * SubViewport
                * MapCamera
let SubViewport.World2D = root.GetViewport();
let MapCamera.GlobalPosition = MainCamera.GlobalPosition;
```

这样就能将 SubViewportContainer 作为小地图，并且通过CanvasLayer限制其层数。

值得注意的是：这里必须要使用 CanvasLayer， 这是由硬件的并行工作原理限制的。
如果在同一层渲染，渲染 root.Viewport 时需要获得 SubViewport 的纹理，渲染 SubViewport  时需要 root.Viewport  纹理，这就在一次渲染中既需要读也需要写纹理，这在目前的硬件上是不好实现的。

所以添加了 CanvasLayer，让两个 Viewport 的绘制不是同一个调用中绘制的！！

此外 SubViewportContainer 好像有神奇的功能， 使用ViewportTexture 会抛出找不到 SubViewport 的错误（虽然也能显示，但是会报错，不清楚具体原因）
```
E 0:00:00:0625   get_node: Node not found: "CanvasLayer/CustomSubViewport" (relative to "Node2D").
  <C++ 错误>       Method/function failed. Returning: nullptr
  <C++ Source>   scene/main/node.cpp:1364 @ get_node()

E 0:00:00:0625   setup_local_to_scene: ViewportTexture: Path to node is invalid.
  <C++ 错误>       Condition "!vpn" is true.
  <C++ Source>   scene/main/viewport.cpp:76 @ setup_local_to_scene()

```


