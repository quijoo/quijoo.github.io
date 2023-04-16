# 游戏引擎热更新技术


## 什么是热更新技术?

游戏上线后可能会定期的更新资源(美术资源+代码)，需要一种可靠的方案来更新代码。

对于 C# 来说就是将代码编译为 .dll 文件，放入 AssetBundle 热更包里边，在客户端动态的的加载，当然这会带来一系列的限制。

下面记录一下 Unity 和 Godot 中的热更新方案。

## Unity + ILRuntime
转载自 [10274176.html](https://www.cnblogs.com/zhaoqingqing/p/10274176.html) [10458333.html](https://www.cnblogs.com/zhaoqingqing/p/10458333.html) 
ILRuntime 相当于在客户端运行了一个能动态加载C# 字节码 DLL 的程序。

ILRuntime 项目和 Unity C# 项目运行在两个不同的域中，他们可以通过以下方式交互

### 1. 根据字符串找到方法
```c#
appdomain.Invoke("HotFix_Project.InstanceClass", "StaticFunTest", null, null);
```
或者
```c#
//预先获得IMethod，可以减低每次调用查找方法耗用的时间
IType type = appdomain.LoadedTypes["HotFix_Project.InstanceClass"];
//根据方法名称和参数个数获取方法
IMethod method = type.GetMethod("StaticFunTest", 0);

appdomain.Invoke(method, null, null);
```
也可以在主工程中抛出事件，在热更工程中监听，原理和Invoke是一样的


### 2. 监听主工程的委托
在热更工程中监听主工程的事件，由主工程触发。

如果非Action或Func，则需要在主工程中写适配器，所以建议使用系统自带的Action和Func。

### 3. 继承主工程，实现主工程接口
Demo中有三个适配器示例，都继承自CrossBindingAdaptor，适配器中有内部类，继承或实现接口的方法。
```c#
public class MonoBehaviourAdapter : CrossBindingAdaptor
{
    public override Type BaseCLRType {get { return typeof(MonoBehaviour); } }

    public override Type AdaptorType {get { return typeof(Adaptor); } }

    public override object CreateCLRInstance(ILRuntime.Runtime.Enviorment.AppDomain appdomain, ILTypeInstance instance)
    {
        return new Adaptor(appdomain, instance);
    }
    //为了完整实现MonoBehaviour的所有特性，这个Adapter还得扩展，这里只抛砖引玉，只实现了最常用的Awake, Start和Update
    public class Adaptor : MonoBehaviour, CrossBindingAdaptorType
    {
        ILTypeInstance instance;
        ILRuntime.Runtime.Enviorment.AppDomain appdomain;
        public Adaptor()
        {
        }
        public Adaptor(ILRuntime.Runtime.Enviorment.AppDomain appdomain, ILTypeInstance instance)
        {
            this.appdomain = appdomain;
            this.instance = instance;
        }
        public ILTypeInstance ILInstance { get { return instance; } set { instance = value; } }
        public ILRuntime.Runtime.Enviorment.AppDomain AppDomain { get { return appdomain; } set { appdomain = value; } }
        public void Awake() {}
        void Start(){}
        void Update(){ }
        public override string ToString(){ }
    }
}
```
这里我不明白为什么这样写了之后就能继承了？
### 4. CLR重定向/热补丁
CLRRedirectionDemo，需要在AppDomain中注册 RegisterCLRMethodRedirection

在热更工程中调用主工程的代码时会重定向新的实现，新手可以从修改CLR生成的绑定代码开始。

例子是修改Debug.Log方法，在热更中打印的日志也能显示堆栈信息。

### 5. 生成CLR绑定代码
类似于slua/tolua/xlua一样， 把在热更工程会用到主工程的类添加到列表中，然后生成绑定代码。

ILRuntime提供一个智能分析在热更工程会访问主工程的代码并生成CLR绑定代码。

在CLRBindingDemo示例中生成CLR绑定代码之后性能提升10倍

在生成绑定代码之后，热更中访问主工程中方法是调用的 xxx_Binding.cs，而如果不生成则是通过反射调用。

### 6. 在热更工程使用协程
官方例子是调用主工程的方法来启动协程，我测试热更工程也可以调用MonoBehaviour的方法开启协程。

热更工程代码如下：
```c#
public static void RunTest()
{
	Debug.Log("热更工程中开启协程");
	CoroutineDemo.Instance.StartCoroutine(Coroutine());
}

static System.Collections.IEnumerator Coroutine()
{
	Debug.Log("开始协程,t=" + Time.time);
	yield return new WaitForSeconds(3);
	Debug.Log("等待了3秒,t=" + Time.time);
}
```
### 7. 对主工程的值类型做绑定
那些需要做绑定？Unity的常用值类型，比如：Vector3，Vector2

这项操作会提升性能，减少额外的CPU开销和GC Alloc。

方法：
```c#
AppDomain.RegisterValueTypeBinder(typeof(Vector3), new Vector3Binder());
```
Demo 做了十万次向量的运算，添加值类型绑定，不绑定的耗时如下：
```c#
Value: a=(100001.0, 100002.0, 100003.0),dot=0, time = 750ms
Value: a=(100001.0, 100002.0, 100003.0),dot=0, time = 1911ms

Value: a=(-0.4, -0.8, -1.7, -5.1),dot=-124.7494, time = 604ms
Value: a=(-0.4, -0.8, -1.7, -5.1),dot=-124.7494, time = 1550ms

Value: a=(100001.0, 100002.0),dot=0, time = 710ms
Value: a=(100001.0, 100002.0),dot=0, time = 1902ms
```

### 热更工程主要用例
根据我们以往的项目使用Lua做为热更脚本为例，我们在热更工程做的最多的事情：

* 在热更工程中调用主工程的方法，或监听主工程的事件，监听Unity组件触发的事件。
* 热更工程调用主工程接口加载资源
* 热更工程处理UI代码逻辑
* 读取配置，对配置解析
* 热更工程中处理网络的回调
* 热更工程基本处理所有的业务逻辑
### 为什么要写适配器
因为ILRuntime可以理解为蓝大写的C#虚拟机，这个虚拟机要在运行时和Unity的脚本进行交互。

由于IOS的AOT限制，在运行时ILRuntime中是不知道Unity中的类型，所以需要我们在主工程写适配器让ILRuntime知道如何调用在Unity的代码，或当Unity的事件触发时让ILRuntime能够监听到。

### ILRuntime 的委托
ILRuntime 内部使用委托，或者 ILRuntime 使用主工程委托都是 OK 的， 但是需要将委托实例传到 Hotfix 域的外部使用，就需要编写委托适配器（按照官网的 demo 写，Ation、Func 外的委托需要额外实现一个转换器）

### ILRuntime 继承主工程的类接口

适配器类是需要在主工程中实现的，这个适配器的本质就是在调用 Hotfix 域中的方法。

观察 ILRuntime 的 Demo 可以发现实际上就是在 Adaptor中 执行以下逻辑
```c#
static CrossBindingMethodInfo mVMethod1_0 = new CrossBindingMethodInfo("VMethod1"); 
public override void VMethod1()
{
    if (mVMethod1_0.CheckShouldInvokeBase(this.instance))
        base.VMethod1();
    else
        mVMethod1_0.Invoke(this.instance);
}
```
实际上就是预留了一个绑定的 位置！
最后记得注册适配器
```c#
appdomain.RegisterCrossBindingAdaptor(new ClassInheritanceAdaptor());
```
那么在 Hotfix 工程中如何继承该类呢？ [参考链接](https://blog.csdn.net/HiggsParticleHYX/article/details/119779888)

### 反射
参考[官方文档](https://ourpalm.github.io/ILRuntime/public/v1/guide/reflection.html) 总的来说，Hotfix 访问主工程是容易的，而主工程访问Hotfix 工程是困难的！

### CLR 重定向
这也在官方文档中写得很清楚了，主要就是编写 CLR重定向方法并且注册方法（将这个方法映射到一个CLR方法）

### CLR 绑定
热更代码中访问主工程代码是通过反射调用实现的，这样会不可避免的带来性能开销和 GC，对于经常调用的接口进行绑定可以消除反射带来的 GC Alloc

CLR 绑定生成的是借助 CLR 重定向实现的，但是手动实现需要精细的控制，所以 ILRuntime 提供了自动生成方法，我看不懂这个地方放 的代码，以后需要使用的时候再学吧！！
[文档](https://ourpalm.github.io/ILRuntime/public/v1/guide/bind.html)

## Unity + Huatuo
### 原理
**C#运行时结构**
![v2-f9db604624b4bfbaf111a9d95e040b97_720w](vx_images/338792822230449.webp =500x)

最后一个概念AOT(Ahead of time)，AOT技术指的是将高级开发语言直接转成传统的编译型编程语言(如C/C++),再编译成机器指令代码在硬件上运行。IL2CPP可以成为AOT技术。

Unity 打包运行时为：AOT + IL2CPP VM(提供基础服务，如GC)，对于底层运行时而言，是数据运行对象 + 代码机器指令两部分，huatuo 热更新就是扩展了 IL2CPP VM 服务，在使用 **原来数据对象的情况下**， 扩展了**解释执行 IL 代码**的功能，让 IL2CPP 的运行模式变为了 **内存对象 + AOT机器代码指令 + IL指令解释执行** 三个部分，huatuo 热更新时只需要利用 Unity ADF(asmdef, 程序集定义机制)让Unity对某一部分代码单独编译出一个 IL 指令的 DLL 文件。

热更新时 IL2CPP_huatuo就可以加载 IL 指令， 由 IL2CPP_huatuo 来解释执行。并且huatuo 解释执行时使用了原来的 AOT 数据内存对象，所以 huatuo 方案不需要其他热更方案的**接口导出，跨域调用等一些列问题**

开发者不需要任何特殊处理，内存占用，性能都会更好。

**huatuo的革命性优势**
huatuo第1个优势是基于AOT(本地机器代码执行)+Interpreter (IL解释执行)使用同一个内存数据对象，没有跨域访问的问题

1. 我们来拿xLua或ILRuntime热更方案来举例，这些方案都有一条原则，**尽量减少与Unity C#层的交互**，但是这种交互又避免不了而且量大，比如我们要在逻辑热更代码里面访问 Unity C#的GameObject对象数据，最终在运行的时候，GameObject 会在AOT模式下的原生内存数据结构对象。

2. 由于xLua或ILRuntime有自己的虚拟机，所以不能直接访问原生GameObject数据对象，往往要把访问里面的数据包装成函数，这样性能开销就大大的增加了。而huatuo是在IL2CPP模式下的解释执行，直接可以访问原生的数据对象。

3. 相比传统的Lua或ILRuntime热更,他能更新任意部分的代码。不用像Lua或ILRuntime一样，分热更代码+框架代码，框架代码有bug还不能热更。
### 优缺点
**优点**
1. 热更和非热更开发体验一致
2. 原生支持 95% 以上的代码热更需求，包括不限于 await/aysnc, 跨域继承， 跨域调用
3. 第三方插件接入不需要框架层适配！华佗是在 IL2CPP 完成了热更功能！
4. 额外内存开销比较小
5. 特性完整（继承，反射，多线程，volatile，ThreadStatic，Task，async），没有代码生成，没有特殊代码，没有特殊限制
**缺点**
1. 基于 IL2CPP 的底层实现导致无法真机 C# Debug
2. AOT 泛型导致的一些限制（完善中）
    * 在 HybridCLR中应该已经完善了泛型问题，主要使用两个机制 **泛型共享 + 补充元数据**
3. 无法真机的 C# Debug，但是可以断点跟踪 huatuo 指令集执行过程，在editor 可以直接使用 mono 的调试方式

**平台支持**
1. PC（稳定）
2. Android（跑通几个了）
3. IOS （比较少的APP跑通了）

### 实践
[bilibili](https://www.bilibili.com/video/BV1SW4y1a7dr)

## Godot 热更新

godot 打包时可以选择需要的 DLC 资源得到 PCK

也可以直接打包为 EXE （静态的一些资源

然后将 PCK 放到服务器设置好目录

最后在 Godot 端实现 PCK 的下载和加载

