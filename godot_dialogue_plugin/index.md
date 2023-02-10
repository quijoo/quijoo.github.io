# [Godot-Plugin] Dialogue Manager && Dialogic


Godot 有两个主流的 Dialog插件
* [Dialogue Manager](https://github.com/nathanhoad/godot_dialogue_manager)
* [Dialogic](https://github.com/coppolaemilio/dialogic)

Dialogic 复杂庞大并且有丰富的图形化界面, 但是自定功能需要阅读很多源码
DialogManager 简单, 提供一种纯文本的方式编辑 Dialog， 添加自定义功能较简单， 并且能再运行时编辑 Dialog

本文仅根据[官方文档](https://github.com/nathanhoad/godot_dialogue_manager/blob/main/docs/Writing_Dialogue.md)介绍 Dialogue Manager 使用方式

## Writting Dialogue

### Nodes
Dialogue Manager 将**对话**抽象为 dialogue 节点，dialogue 节点是以行为单位的传闻本结构，使用 `~ <dialog_title>` 标记一个名为 <dialog_title> 的节点的开始

### Dialogue

一个dialogue是形如`Character: What they say.`的一行文本

### 随机选择
`Nathan: [[Hi|Hello|Howdy]]! `
使用`[[...]]` 包裹的内容随机选择一个出现

### 变量
使用`{{...}}`在文本种添加变量， 所有的变量需要是**全局的**或者在 DialogManagerSetting 种指定

### bbcode
dialogue 支持富文本标记 bbcode

如果使用插件提供的 `DialogueLabel` 节点显示文本，可以额外的支持[wait=N]、 [speed=N] 和 [next=?] 标签

* wait 标签显示完一段对话后等待的时间
* speed 对话文本显示速度（打字速度）
* next 下一个对话的间隔？

### 随机对话
之前的 `[[]]`是随机替换一个对话中的文本， 而 `%` 提供随机的一句对话，eg:
等概率随机对话
```
Nathan: I will say this.
% Nathan: And then I might say this
% Nathan: Or maybe this
% Nathan: Or even this?
```

带权重的随机对话
```
%3 Nathan: This line as a 60% chance of being picked
%2 Nathan: This line has an 40% chance of being picked
```

### 跳转
从一段对话跳转到另一段对话(类似 goto)
```
=> another_title
```

从一段对话跳转到另一段对话后返回下一句(类似过程调用)
```
=>< another_title
```

可以跨文件导入对话(文件头部)
```
import "res://snippets.dialogue" as snippets
```
导入后使用  `=> snippets/talk_to_nathan` 来跳转对话

### 回答(选项)

```
Nathan: What would you like?
- This one
- No, this one
- Nothing
```

回答中也可以包含 bbcode、变量

默认是按照dialogue 的列表顺序进行，但是可以使用 `=>` 根据不同的选项跳转,其中 `END` 表示结束对话
```
Nathan: What would you like?
- This one
    Nathan: Ah, so you want this one?
- Another one => another_title
- Nothing => END
```

如果一个选项中包含了以 `<name>:` 开始的对话，玩家选中后会被当作一个对话进行？（没太懂）
```
Someone: Here is a thing you can do.
- That's good to hear!
    Nathan: That's good to hear!
- That's definitely news
    Nathan: That's definitely news
```

等价于
```
Someone: Here is a thing you can do
- Nathan: That's good to hear!
- Nathan: That's definitely news
```

### 条件
dialogue 中的条件
```
if SomeGlobal.some_property >= 10
    Nathan: That property is greather than or equal to 10
elif SomeGlobal.some_other_property == "some value"
    Nathan: Or we might be in here.
else
    Nathan: If neither are true I'll say this.
```

response 中的条件使用 '[' 和 ']' 包起来
```
Nathan: What would you like?
- This one [if SomeGlobal.some_property == 0 or SomeGlobal.some_other_property == false]
    Nathan: Ah, so you want this one?
- Another one [if SomeGlobal.some_method()] => another_title
- Nothing => END
```

### Mutations
使用 `set` / `do` 来修改变量和调用方法
```
if has_met_nathan == false
    do SomeGlobal.animate("Nathan", "Wave")
    Nathan: Hi, I'm Nathan.
    set has_met_nathan = true
Nathan: What can I do for you?
- Tell me more about this dialogue editor
```


请求一段对话时可以传入对象数组，该对话 检查是否存在可用的 Mutations 方法

内置的 Mutations 方法
* **emit** 发射信号
* **wait** 等待时间（单位 秒）
* **debug** print debug messgae

Mutations 可以在 dialogue 行内被调用，并且是在文本打印到行内 mutations 位置时调用
```
Nathan: I'm not sure we've met before [do wave()]I'm Nathan.
Nathan: I can also emit signals[do emit("some_signal")] inline.
```
tips: inline mutations 中的 await 是无效的， 即 inline mutations 方法必须是同步的

### Error checking
游戏运行前语法错误会被高亮

### 测试场景

### 自定义测试场景
测试场景必须继承 `BaseDialogueTestScene`,  并且提供一个 dialogue resource 和 起始标题
```
extends BaseDialogueTestScene

func _ready() -> void:
    # Change this to open your custom balloon
	DialogueManager.show_example_dialogue_balloon(resource, title)
```

### 翻译
略

## 设置
### 编辑器
当使用静态翻译key并且手动添加时`Treat missing translations as errors` （看不懂

`Wrap long lines` : 自动换行

### 运行时
`Include responses with failed conditions` : 将包括未能通过其条件检查的响应列表附加到给定的行。(看不懂)

### 全局变量
插件的运行时本身是无状态的，这意味着需要自行提供变量和方法来运行，manager 首先在当前场景中查询然后在全局作用域中查询

如果不想使用 `GameState.some_variable` 可以在 Autoload 中 达赖GameState.gd 然后直接引用 `some_variable`

##  在游戏中使用对话
你可以自行决定dialogue 的表现方式和输入控制，也可以参考[a few example balloons](https://github.com/nathanhoad/godot_dialogue_manager/blob/main/docs/Example_Balloons.md)

可以调用 `DialogueManager.show_example_dialogue_balloon(resource, title)` 来调用使用内置的样例(render/Inputs)

当开始自己构建对话时，需要获得一行对话和使用 dialogue lable node

### 获取一行对话
通过静态类 DialogueManager 来访问对话管理器

可以使用 `await DialogueManager.get_next_dialogue_line(resource, title)` 来获取一行对话， 也可以直接从对话的resource 对象调用`get_next_dialogue_line` 方法来调用
疑问：如果这里直接返回了对话，那对话中的 Mutations 将在什么时机调用呢？
回答：行内的 Mutations 交给 DialogueLabel 处理， Manager 会在调用 get_next_dialogue_line 时自行出里行间的 Mutations

```
var resource = load("res://some_dialogue.dialogue")
# then
var dialogue_line = await DialogueManager.get_next_dialogue_line(resource, "start")
# or
var dialogue_line = await resource.get_next_dialogue_line("start")
```

这里的 `dialogue_line` 是一个 `RefCounted` 类型 

现在，Dialogue _ line 包含一个字典，其中包含行 Nathan: 这里有一些选项。.此字典还包含响应选项列表。

没一个选项对应一个 `next_id` 属性, 常用于延续分支

### DialogueLabel 节点

该节点需要提供一个 `dialogue_line` 字典， 如上文提及的
该节点会处理 `bb_code`、 `wait`、 `speed` 、`inline_mutation` 节点

该节点通过 .type_out() 方法来开始打印对话文本， 并且在文本结束时 触发`finished_typing` 信号，在遇到暂停时会触发 paused_typing 信号并携带暂停持续时间，每一个字母被打印时会触发 spoke 信号并携带打印速度。

### 运行时生成 Dialogue Resource

```
var resource = DialogueManager.create_resource_from_text("~ title\nCharacter: Hello!")
```

文本首先被传到分析器，如果存在符号错误该方法会失败（失败是什么意思？
注意：文本分析在创建时进行，不是在使用时进行
如果你没有错误，这个生成的 resource 局部对象，将可以被当作一般的 dialogue resource 使用


## C# wrapper

```c#
using DialogueManagerRuntime;
var dialogue = DG.Load<Resource>("res://example.dialogue");

# do
DialogueManager.ShowExampleDialogueBalloon(dialogue, "start");
# or
var line = await DialogueManager.GetNextDialogueLine(dialogue, "start");
```
tips: 这个 RefCounted  对象和 gd 版本有相同的 property ，但是需要手动的使用 .Get(<property_name>) 来访问？？ 可以在`GetNextDialogueLine` 方法内做一个类型转换呀？

