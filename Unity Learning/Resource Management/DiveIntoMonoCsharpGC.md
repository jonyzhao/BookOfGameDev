# 浅析基于Mono的C#资源回收

- [官方的优化GC教程](https://unity3d.com/learn/tutorials/topics/performance-optimization/optimizing-garbage-collection-unity-games)
- [针对Unity开发的C#内存管理](http://blog.csdn.net/ywjun0919/article/details/50688112)

C#资源，在这里指的是Unity.Object以外的类型对象（更具体地分**为托管资源**和**非托管资源**）。在Unity框架中，是由mono虚拟机通过垃圾回收机制（GC）管理的。

> 
**托管资源**：由CLR(Common Language Runtime)管理分配和释放发资源，即从mono运行时里new出来的对象。
**非托管资源**：不受CLR管理的对象，如：Windows内核对象、文件、数据库连接、套接字和COM对象等。


## mono是什么
我们总在说的“mono”，到底指的是什么？[参考这里](http://www.mono-project.com/docs/about-mono/)
> 简而言之： Mono has a C# compiler, a runtime and a lot of libraries.

直白点说，mono是.net的一套免费公开的实现。其特性要落后于.net。而Unity使用的mono更是老版的mono（2.6.5版）


## GC为什么耗时
具体是怎样判断哪些需要被回收呢？我们的直观想法是引用计数，很快很简单。但这样会有循环引用而不得释放的问题。Unity携带的mono，GC采用的是`Boehm GC`，而最新的mono已经采用了`generational GC`——sgen。即使像Boehm这样的GC，其策略也是 rootObj，而不是引用计数。概括一下Unity游戏中的GC耗时原因：

- `Boehm GC`以全局数据区和当前寄存器中的对象为根节点进行遍历，这本身就耗时；
- 启动GC前，会暂所有些需要mono内存分配的线程，完成GC后才恢复，造成卡顿。

所以，一方面我们要少制造垃圾，另一方面要控制垃圾回收调用的时机。

## 有了GC机制，为什么还有mono内存泄漏
**mono内存泄漏，并不是真的泄漏**，而是：mono虚拟机向os申请到的内存不会还给os，即使gc后，那些不用的内存（`unused size`）也是自己保存起来备用。不够的时候会再向os申请。因此，mono占用的内存（`heap size`）是只增不减的。当有些资源实际不用了，却占着空间无法被gc，就称为mono内存泄漏。

```cs
//获取mono占用内存的api
[System.Runtime.InteropServices.DllImport("__Internal")]
public static extern long mono_gc_get_used_size();
[System.Runtime.InteropServices.DllImport("__Internal")]
public static extern long mono_gc_get_heap_size();
```

当整个应用被杀死时，mono占用的内存才能还给os。一个有趣的小知识：当在PC模拟器上运行Unity游戏时，点击三角形终止游戏运行，所有内存真的会归还给操作系统吗？答：不是的。如果你用了多线程，那么unity是不对其他线程负责的，点击三角形关闭的只是unity对应的线程。若其他线程访问了unityEngine相关的数据，将是无效的。

> The play mode is just another thread inside the same mono VM. --[Dreamora](https://forum.unity3d.com/threads/threading-causes-memory-leak.87652/)

## 减少GC的常见套路
[官方总结](https://unity3d.com/learn/tutorials/topics/performance-optimization/optimizing-garbage-collection-unity-games)

### foreach
在Unity5.5以前，foreach会产生gc。

遍历字典的写法：
```cs
var tmp = dic.GetEnumerator();
while(tmp.MoveNext())
{
    //tmp.Current.key;
    //tmp.Current.Value
}
```

### 字符串拼接

```cs
string a = "afadf"
string c = a + i;
```
当i不是string类型，实际执行的是`string.Concat((object)a, (object)i)`，存在装箱拆箱。
这种字符串拼接常见于打日志操作。为此可以做一些优化。

```cs
long uin = 111;

MyDebug.Log("the player uin is: " + uin);//常见写法，有额外GC和装箱拆箱

MyDebug.Log("the player uin is: " + uin.ToString());//无装箱拆箱，有额外GC

StringBuilder sb = new StringBuilder();
MyDebug.Log(sb.Append("the player uin is: ").Append(uin).ToString()); //无装箱拆箱，无额外GC，但是这样写麻烦

```
利用泛型函数可以简化上述无装箱拆箱和额外GC的写法。

```cs
StringBuilder sb = new StringBuilder();
public static void Debug(string str)
{
    Debug.Log(str);
}

public static void Debug<T1, T2>(T1 arg1, T2 arg2)
{
    sb.Length=0;//清空sb
    sb.Append(arg1).Append(arg2);
    Debug.Log(sb.ToString());
}

//可以再补充支持3个、4个参数的。。。

MyDebug.Log("the player uin is: " , uin);//现在可以这么写，无额外GC，无装箱拆箱
```

### yield return
【补充】

### struct替代class
struct是在栈上的，更快，而且不存在GC。什么时候用呢？
通常我们创建的引用类型总是多于值类型。如果以下问题的回答都为yes，那么我们就应该创建为值类型：

- 该类型的主要职责是否用于数据存储？
- 该类型的共有借口是否完全由一些数据成员存取属性定义？
- 是否确信该类型永远不可能有子类？
- 是否确信该类型永远不可能具有多态行为？

## Xlua中对复杂值类型GC的处理

【补充】

## IDisposable与Unmanaged Resources
[使用IDisposable的正确姿势](https://stackoverflow.com/questions/538060/proper-use-of-the-idisposable-interface)

[什么是Unmanaged Resources？](https://stackoverflow.com/questions/3433197/what-exactly-are-unmanaged-resources)

对于托管资源，其由GC管理；对于非托管资源，是要