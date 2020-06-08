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

   Assets are the models，textures，sounds and all other “content”files from which you make your game。

这里再说明下，SQLite3是SQLite的第三个主要版本，避免大家突然看到Sqlite3不知道什么意思。



## Asset

### 种类

基本来说Asset分为三种：第一种就是外部导入的文件，第二种就是Unity内部产生的诸如Prefab、Scene，第三种就是Script，脚本其实也算一种Asset。

> 官方对Assets的描述：Assets are the models，textures，sounds and all other “content”files from which you make your game。

### 组成

一般来说，Asset由两部分组成，第一部分就是数据内容，比如我们导入了一个fbx文件其实它本质上就是一个数据，第二部分就是其所对应的meta文件。

Asset会被导入到Unity的Library文件夹之中，Unity本身是不会动你的原始文件的，其实你第一次打开项目或者切换平台时，Unity要加载好久其实就是干这事。真正在Editor和Runtime中使用的其实是Library里的东西。

首先大家要知道Asset的加载在Editor中和在Runtime（真机）中是不一样的，这就是为什么你在Editor中用*Profiler*查看内存的使用情况和真实在手机上运行的使用情况有很大出入的原因。

下面给出两种情况的区别

* **Editor**：为了保证开发人员使用的流畅性，会把Asset全部加载进内存，避免之后使用时再加载产生卡顿的情况出现。
* **Runtime**：严格保证按需加载。



### GUID和FileID



### 生命周期

如果想通过Resources.UnloadUnusedAssets()卸载从AssetBundle加载的资源，一定要先将AssetBundle卸载后才能生效。

&nbsp;

&nbsp;

&nbsp;

## AssetBundle

AssetBundle的本质其实就是个Zip包，它包含一个头（摘要信息）和一个体（压缩的内容）。

### 加载方式

* `LoadFromCacheOrDownload`
* `LoadFormMemory`
* `LoadFromFile`

### 卸载方式

`AssetBundle.Unload(bool unloadAllLoadedObjects);`

参数unloadAllLoadedObjects代表是否要将加载出来的Asset一起卸载了。

### 压缩方式

压缩格式在打包时通过AssetBundleOption参数选择，下面介绍两种常用的压缩方式。

* LAMZ：Unity打包的默认格式，会将序列化数据压缩成LZMA流，使用时需要整体解包。优点是打包后体积小，缺点是解包时间长，且占用内存。
* LZ4：压缩率不及LZMA，但是不需要整体解压。LZ4是基于chunk的算法，加载对象时只有响应的chunk会被解压。

BuildAssetBundleOptions：

* ChunckBaseCompression（改良版的LZ4）
* DisableWriteTypeTree（可以减小AB包的大小，内存占用，加载时间）
* DisableLoadAssetByFileName
* DisableLoadAssetByFileNameWithExtensiion

### TypeTree

前面的Options中提到了TypeTree这个概念，为什么关闭可以获得这么大的性能提升呢？

这是因为，TypeTree其实是为了跨版本的兼容性而生的。

**关闭条件**：打APK的Unity版本和打AB包的Unity版本一致就放心大胆的关闭吧。

### 建议

Unity官文对AB包大小的建议是如果放在远程的话1~2MB之间，如果放在本地的话5~10MB之间，不要超过10MB不然可能会有问题。