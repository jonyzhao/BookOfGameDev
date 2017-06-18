# 浅析AssetBundle机制
[官方深度详解 best practice](https://unity3d.com/learn/tutorials/topics/best-practices/assetbundle-fundamentals?playlist=30089)
[官方指南系列 manual](https://docs.unity3d.com/Manual/AssetBundlesIntro.html)

## （一）概要
#### 定义

`AssetBundle System`提供了一套资源文件打包、并可以被unity方便地索引的方法。打出的包下文我们简称为`ab包`。`AssetBundle`除了抽象的内涵，这个字符串本身也是unity暴露给c#的wrapper类名。

#### 设计目的
提供这套系统为的就是与unity的序列化系统相互兼容。

#### 作用
 - 资源分离，缩减安装包大小
 - 资源差异化，不同设备选择不同资源加载
 - 资源更新，不过不包含代码逻辑更新

#### 内部结构
 - header
     - assetbundle的id
     - 是否压缩
     - manifest。所包含Object的查找表（基于平衡二叉树或红黑树）
 - data
     - 如果开启压缩，则默认采用`LZMA`压缩算法将**整体压缩**成一个ab包。
     - 如果选择`LZ4`算法，则可以选择为每个 Object **单独压缩**。这样的好处是加载一个Object可以不用解压整个ab包到缓存中，加载的速度甚至和不压缩差不多。注意，Unity5.3以后才引进了LZ4这个选项。

#### 关于 LZMA 和 LZ4 算法
LZMA比LZ4压缩/解压速度更慢，但压缩率更高（压缩后的文件更小）。

详见： https://catchchallenger.first-world.info/wiki/Quick_Benchmark:_Gzip_vs_Bzip2_vs_LZMA_vs_XZ_vs_LZ4_vs_LZO




## （二）生成ab包
- 资源对应到ab包（在Unity编辑器中直接完成）
  - Unity 5后加入了`variant`子参数，可以准备高清、低清
- 调用ab包打包脚本（这里必须自己写，并且只能放到 `Editor` 目录下）

```csharp
using UnityEditor;
using System.IO;

public class CreateAssetBundles
{
    [MenuItem("Assets/Build AssetBundles")]
    static void BuildAllAssetBundles()
    {
        string assetBundleDirectory = "Assets/AssetBundles";
        if(!Directory.Exists(assetBundleDirectory))
        {
            Directory.CreateDirectory(assetBundleDirectory);
        }


        /* opt:
         * None                     =   LZMA 压缩
         * UncompressedAssetBundle  =   不压缩
         * ChunkBasedCompression    =   LZ4 压缩
         * 
         * 参见 https://docs.unity3d.com/Manual/AssetBundles-Building.html
         */
        BuildAssetBundleOptions opt = BuildAssetBundleOptions.None;
        
        BuildPipeline.BuildAssetBundles(assetBundleDirectory, opt, BuildTarget.StandaloneWindows64);
    }
}
```

### ab包小工具
Unity官方制作的AssetBundle管理器，既包含了下载和导入的逻辑，还包含了模拟ab包下载的编辑器插件。
插件安装使用：https://docs.unity3d.com/Manual/AssetBundles-Manager.html
插件源码：https://bitbucket.org/Unity-Technologies/assetbundledemo

另外，为了使工作流更可视化，Unity正尝试推出新的流插件，详见： https://blogs.unity3d.com/2016/10/25/new-assetbundle-graph-tool-prototype/


## （三）ab包的内部结构
按照上面的打ab包脚本，从菜单中执行后，在AssetBundles文件夹中有`2*(n+1)`个文件（n是自定义的ab包的数量，ab包必须全部小写）。例如我们有myprefabs和mysprites两个ab包，那么会生成：

> AssetBundles
AssetBundles.manifest
myprefabs
myprefabs.manifest
mysprites
mysprites.manifest

下面逐个分析每个文件。

### AssetBundles
是一个序列化的 `AssetBundleManifest 对象`。提供了一些API，诸如：

> 
- `GetAllAssetBundles()` which returns all the AssetBundle names in this build.
- `GetDirectDependencies()` which returns the direct dependent AssetBundle names.
- `GetAllDependencies()` which returns all the dependent AssetBundle names.
- `GetAssetBundleHash(string)` which returns the hash for the specified AssetBundle.
- `GetAllAssetBundlesWithVariant()` which returns all the AssetBundles with variant.

### AssetBundles.manifest
记载了CRC，以及各个AssetBundle之间的**依赖关系**。

例如下面有两个AssetBundle：myprefabs和mysprits，其中myprefabs里有一个Obejct用到了mysprits里的一张图，所以产生了依赖关系。

```
ManifestFileVersion: 0
CRC: 320579488
AssetBundleManifest:
  AssetBundleInfos:
    Info_0:
      Name: myprefabs
      Dependencies:
        Dependency_0: mysprites
    Info_1:
      Name: mysprites
      Dependencies: {}
```

### myprefabs
ab包。注意，使用共同资源的prefab最好打到同一个ab包中，否则会大量浪费空间。

### myprefabs.manifest
对于每个AssetBundle都生成了对应的manifest，记载了包含哪些Assets以及依赖何种资源。这个文件仅供打差异包使用，运行时不可用。
> [This manifest file is only used for incremental build, not necessary for runtime.](https://docs.unity3d.com/500/Documentation/Manual/BuildingAssetBundles5x.html)

```
ManifestFileVersion: 0
CRC: 129934035
Hashes:
  AssetFileHash:
    serializedVersion: 2
    Hash: eb750738b2f2979dc0f9a95b36fa0de9
  TypeTreeHash:
    serializedVersion: 2
    Hash: 80992b54a8950244f5c8a5c98a859660
HashAppended: 0
ClassTypes:
- Class: 1
  Script: {instanceID: 0}
- Class: 4
  Script: {instanceID: 0}
- Class: 21
  Script: {instanceID: 0}
  Script: {fileID: 1392445389, guid: f5f67c52d1564df4a8936ccd202a3bd8, type: 3}
- Class: 114
  Script: {fileID: -765806418, guid: f5f67c52d1564df4a8936ccd202a3bd8, type: 3}
- Class: 114
  Script: {fileID: 708705254, guid: f5f67c52d1564df4a8936ccd202a3bd8, type: 3}
- Class: 115
  Script: {instanceID: 0}
Assets:
- Assets/Prefabs/Button.prefab
- Assets/Prefabs/cardSprite.prefab
Dependencies:
- F:/TestProjects/TestAssetBundle/TestAssetBundle/Assets/AssetBundles/mysprites

```



## （四）加载/管理/卸载ab包
参考 https://docs.unity3d.com/Manual/AssetBundles-Native.html

### 加载ab包（见笔记本）
加载ab包主要有4个API，但具体表现会根据压缩方法、平台的不同而不同。

> 
- ~~AssetBundle.LoadFromMemoryAsync /  CreateFromMemory~~ **DO NOT USE IT ANYMORE**
- AssetBundle.LoadFromFile
- WWW.LoadFromCacheOrDownload
- UnityWebRequest's DownloadHandlerAssetBundle (on Unity 5.3 or newer)

### assetbundle管理器
[官方开源项目](https://bitbucket.org/Unity-Technologies/assetbundledemo)

### 处理ab包依赖
首先要明确的一点是：unity不会自动加载依赖关系。

好消息是，对于有依赖关系的ab包，加载的先后顺序不重要，如果ab包1依赖于ab包2，那么，只要自己保证在加载ab包1的某个Object的时候，其依赖的ab包2已经被加载（通过使用上文介绍过的`AssetBundleManifest对象`）就行。



### 实例化ab包中的asset






## 例子
https://unity3d.com/learn/tutorials/topics/scripting/assetbundles-and-assetbundle-manager