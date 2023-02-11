# [Godot] 多语言脚本交互

godot 社区的许多插件都是用 gdscript 写的，但是为了结构清晰自己在些项目的时候习惯用 C#，这就带来了很多不便。

### 直接与特定脚本交互
对于那些直接与脚本系统交互的gdscript插件，可能需要修改脚本源码才对 c# 生效

例如：行为树插件，该插件实现了一个行为树的图形编辑界面，可以为 `NodeGraph` 中的节点绑定状态脚本，并且可以在UI中查看和编辑行为树的 `blackboard`，或者为行为树添加一些运行时变量。

这类插件如果一开始只是支持特定语言 (获取属性和方法调用通过 `.` 获取)，就需要修改插件代码，使用`GodotObject.Call \ GodotObject.Set/Get` 来调用方法/访问属性

### 间接脚本交互
`github/godot_dialogue_manager` 这样的插件对然是纯 gdscript 实现，但是整个系统仅提供一个名为`DialogueManager`的单例对象，那访问该对象就很容易了

```c#
namespace DialogueManagerRuntime
{
  public partial class DialogueManager : Node
  {
    public static async Task<RefCounted> GetNextDialogueLine(Resource dialogueResource, string key = "0", Array<Variant> extraGameStates = null)
    {
      var dialogueManager = Engine.GetSingleton("DialogueManager");
      dialogueManager.Call("_bridge_get_next_dialogue_line", dialogueResource, key, extraGameStates ?? new Array<Variant>());
      var result = await dialogueManager.ToSignal(dialogueManager, "bridge_get_next_dialogue_line_completed");

      return (RefCounted)result[0];
    }


    public static void ShowExampleDialogueBalloon(Resource dialogueResource, string key = "0", Array<Variant> extraGameStates = null)
    {
      Engine.GetSingleton("DialogueManager").Call("show_example_dialogue_balloon", dialogueResource, key, extraGameStates ?? new Array<Variant>());
    }
  }
}
```

无论是 c# 还是 gsdcript 使用 AutoLoad 方式加载的全局单例对象都可以通过 Engine.GetSingleton("DialogueManager") 获取。

但是得到的 Manger 对象是一个  GodotObject 对象，只能通过以下方式访问对象
```c#
public Variant Call(StringName method, params Variant[] args);
public Variant CallDeferred(StringName method, params Variant[] args);
public Variant Callv(StringName method, Collections.Array argArray);
public Variant Get(StringName property);
public void Set(StringName property, Variant value);
...
```

