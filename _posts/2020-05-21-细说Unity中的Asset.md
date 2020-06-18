---
layout:     post
title:      细说 Unity 中的 Asset
subtitle:   Asset和Meta的关系，GUID和FileID，内存分配与释放
date:       2020-05-21
author:     SWZ
header-img: img/Asset和AssetBundle/bg-asset-assetbundle.jpg
catalog: true
tags:
    - 游戏开发
    - Unity
---

## 前言

Unity对Assets的描述：

> Assets are the models，textures，sounds and all other “content”files from which you make your game。

&nbsp;

## Asset

### 种类

基本来说Asset分为三种：**第一种**就是外部导入的文件，**第二种**就是Unity内部产生的诸如Prefab、Scene，**第三种**就是Script，脚本其实也算一种Asset。Plugin子目录之外的C#脚本会放在Assembly-CSharp.dll中，而Plugin及其子目录中的脚本则放置在Assembly-CSharp-firstpass.all中，这些程序库会被MonoScripts所引用，并在程序第一次启动时被加载。 

### 组成

一般来说，Asset由两部分组成，第一部分就是数据内容，比如我们导入了一个fbx文件其实它本质上就是一个数据，第二部分就是其所对应的meta文件。

Asset会被导入到Unity的Library文件夹之中，Unity本身是不会动你的原始文件的，其实你第一次打开项目或者切换平台时，Unity要加载好久其实就是干这事。真正在Editor和Runtime中使用的其实是Library里的东西。

首先大家要知道Asset的加载在Editor中和在Runtime（真机）中是不一样的，这就是为什么你在Editor中用*Profiler*查看内存的使用情况和真实在手机上运行的使用情况有很大出入的原因。

下面给出两种情况的区别

* **Editor**：为了保证开发人员使用的流畅性，会把Asset全部加载进内存，避免之后使用时再加载产生卡顿的情况出现。
* **Runtime**：严格保证按需加载。

### GUID和FileID



&nbsp;

## AssetBundle

AssetBundle的本质其实就是个Zip包，它包含一个头（摘要信息）和一个体（压缩的内容）。

### 构建方式

利用`BuildPipeline.BuildAssetBundles(string outputPath, BuildAssetBundleOptions assetBundleOptions, BuildTarget targetPlatform)`函数

#### BuildAssetBundleOptions：

* `None`（默认LZMA，会将序列化数据压缩成LZMA流，使用时需要整体解包。优点是打包后体积小，缺点是解包时间长，且占用内存）

* `ChunckBaseCompression`（改良版的LZ4，压缩率不及LZMA，但是不需要整体解压。LZ4是基于chunk的算法，加载对象时只有响应的chunk会被解压）

* `DisableWriteTypeTree`（可以减小AB包的大小，内存占用，加载时间。TypeTree其实是为了跨版本的兼容性而生的。关闭条件：打APK的Unity版本和打AB包的Unity版本一致就放心大胆的关闭吧）

* `DisableLoadAssetByFileName`

* `DisableLoadAssetByFileNameWithExtensiion`

* `UncompressedAssetBundle`（不压缩）

* `ForceRebuildAssetBundle`（强制重新打包所有AB资源）

* `StrictMode`（只要有一点错就终止打包）

> **Unity官文对AB包大小的建议是如果放在远程的话1~2MB之间，如果放在本地的话5~10MB之间，不要超过10MB不然可能会有问题。**

### 加载方式

* **AB加载**

   1. `LoadFromCacheOrDownload(PathURL + "/fileName",version)`

      使用此方式加载，将先从硬盘上的存储区域查找是否有对应的资源，再验证本地Version与传入值之间的关系，如果传入的Version>本地，则从传入的URL地址下载资源，并缓存到硬盘，替换掉现有资源，如果传入Version<=本地，则直接从本地读取资源；如果本地没有存储资源，则下载资源。此方法的存储路径无法设定以及访问。使用此方法载入资源，不会在内存中生成WebStream（其实已经将WebStream保存在本地），如果硬盘空间不够进行存储，将自动使用`new WWW`方法加载，并在内存中生成WebStream。
      
   2. `www.assetbundle`

   3. `LoadFormMemory`

      WWW和这种方式AssetBundle文件会整个镜像于内存中，理论上文件多大就需要多大的内存，之后Load时还要占用额外内存去生成Asset对象。

   4. `LoadFromFile`

      这种方式不会把整个硬盘AssetBundle文件都加载到 内存来，而是类似建立一个文件操作句柄和缓冲区，需要时才实时Load，所以这种加载方式是最节省资源的，基本上AssetBundle本身不占什么内 存，只需要Asset对象的内存。可惜只能在PC/Mac Standalone程序中使用。

* **Asset加载**

   1. `AssetBundle.Load`
   2. `AssetBundle.LoadAsync`
   3. `AssetBundle.LoadAll`

### 卸载方式

* **AB卸载**

  `AssetBundle.Unload(bool unloadAllLoadedObjects)`

  参数unloadAllLoadedObjects代表是否要将加载出来的Asset一起卸载了。

* **Asset卸载**

  1. `Resources.UnloadAsset(Object)`

  2. `Resources.UnloadUnusedAssets`

     一定要先将AssetBundle卸载后才能生效。

### 运行时加载细节

Load一个AssetBundle等于把硬盘或者网络的一个文件读到内存一个区域，这时候只是个AssetBundle**内存镜像数据块**，还没有Assets的概念。

用`AssetBundle.Load`(同`Resources.Load`) 这才会从AssetBundle的内存镜像里读取并创建一个Asset对象，创建Asset对象同时也会分配相应内存用于存放(反序列化)。

#### 举两个例子帮助理解

##### 例子1：

一个常见的错误：

- 从AssetBundle里Load了一个prefab并克隆：`obj = Instaniate(MyAssetBundle.Load('MyPrefab”)`
- 然后不需要的时候调用`Destroy(obj)`，你以为就释放干净了。
- 其实这时候只是释放了Clone对象，通过Load加载的所有引用、非引用Assets对象全都静静静的躺在内存里。
- 这种情况应该在Destroy以后用：`MyAssetBundle.Unload(true)`彻底释放干净。
- 如果这个AssetBundle是要反复读取的，不方便`Unload`，那可以在Destroy以后用：`Resources.UnloadUnusedAssets`把所有和这个Assetbundle有关的Asset都销毁。
- 当然如果这个Prefab也是要频繁创建、销毁的，那就应该让那些Assets呆在内存里以加速游戏体验。
- 由此可以解释另一个之前有人提过的话题：**为什么第一次Instantiate 一个Prefab的时候都会卡一下？**

> 因为在你第一次`Instantiate`之前，相应的Asset对象还没有被创建，要加载系统内置的AssetBundle并创建Assets。第一次以后你虽然Destroy了，但Prefab的Assets对象都还在内存里，所以就很快了。



##### 例子2：

- 从磁盘读取一个ab.unity3d文件到内存并建立一个AssetBundle对象

  `AssetBundle MyAssetBundle = AssetBundle.CreateFromFile("ab.unity3d");`

- 从AssetBundle1里读取并创建一个Texture Asset,把obj1的主贴图指向它

  `obj1.renderer.material.mainTexture = AssetBundle1.Load("wall") as Texture;`

- 把obj2的主贴图也指向同一个Texture Asset

  `obj2.renderer.material.mainTexture =obj1.renderer.material.mainTexture;`

- **Texture是引用对象，永远不会有自动复制的情况出现**(除非你真需要，用代码自己实现copy)，只会是创建和添加引用。

- `MyAssetBundle.Unload(true)` 那obj1和obj2都变成黑的了，因为指向的Texture Asset没了。

- `MyAssetBundle.Unload(false)` 那obj1和obj2不变，只是AssetBundle1的内存镜像释放了。

- `Destroy(obj1) `，obj1被释放，但并不会释放刚才Load的Texture。

- 此时`Resources.UnloadUnusedAssets`，不会有任何内存释放 因为Texture asset还被obj2用着。

- `Destroy(obj2)`，obj2被释放，但也不会释放刚才Load的Texture。

- `Resources.UnloadUnusedAssets`，这时候刚才load的Texture Asset释放了，因为没有任何引用了。

- 最后`CG.Collect`，强制立即释放内存。

**综上：**

- AssetBundle.Load 内存++【多了文件镜像】
- 实例化Prefab，显存++【多了可视对象】

> Texture加载以后是到内存，显示的时候才进入显存的Texture Memory。
>
> 所有的东西基础都是Object
> Load的是Asset，Instantiate的是GameObject和Object in Scene
> Load的Asset要Unload，new、Instantiate的object可以Destroy