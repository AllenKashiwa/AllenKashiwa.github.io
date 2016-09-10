---
layout:     post
title:      "Unity5.x新的AssetBundle机制01——构建"
date:       2016-03-21 14:47:00
---

# 1前言

![unity_cover.png](/assets/images/in-post/study_assetbundle/unity_cover.png)

Unity在5.0中推出了新的AssetBundle管理机制，本文将对此进行介绍并完成简单实践。

# 2什么是AssetBundles？

AssetBundles是一堆从你的Unity项目中导出的文件，这些文件以特殊的格式组织，并能够在你的项目中按需加载。AssetBundles通过后缀名支持所有Unity支持的文件类型。如果你想包含一些自定义的二进制数据，可以将使用.bytes作为后缀。Unity在导入时将把此类文件当作一个TextAsset。

# 3工作流

## 3.1创建

开发阶段，开发者将AssetBundles上传至至服务器

![](/assets/images/in-post/study_assetbundle/upload_assetbundle.jpg)

这就有两个阶段：

### 3.1.1你需要通过编辑器构建AssetBundles。
### 3.1.2上传至服务器。

## 3.2使用

程序运行阶段，客户端从服务器下载AssetBundles，并按需操作每个AssetBundle中的资源。
![](/assets/images/in-post/study_assetbundle/download_assetbundle.jpg)

即如下两个阶段：

### 3.2.1客户端运行时下载AssetBundles。
### 3.2.2从AssetBundles中加载对象。

# 4实践

## 4.1构建AssetBundles

### 4.1.1准备工作

我们事先准备一些简单的资源：
简单的创建两个矩形，两个球形。

![create_obj.png](/assets/images/in-post/study_assetbundle/create_obj.png)

选中一个Cube，在Inspector视口中的下方，有一个预览窗口。在预览窗口中，我们可以新建并指定资源将被打包进的AssetBundle（默认是None，这表示该资源不被打包进任何AssetBundle，而是被打包进主工程本身），如图：

![new_assetbundle.png](/assets/images/in-post/study_assetbundle/new_assetbundle.png)

图中有两个下拉框，左边的用于指定AssetBundle的名字，后边的用于指定AssetBundle Variants的名字（后文介绍）。

我们新建一个名为shape/cube的AssetBundle，将两个Cube资源指定到其中。新建一个名为shape/sphere的AssetBundle，将两个Sphere资源指定其中（对应的meta文件也将被指定到该AssetBundle）。AssetBundle的命名必须小写（即便写成大写也会被转换成小写），并且支持'/'，这样可以在界面上开辟子目录。如图：

![assetbundle_name.png](/assets/images/in-post/study_assetbundle/assetbundle_name.png)

如果你创建了一些没有指定任何资源的AssetBundle，**Remove Unused Names** 按钮可以将其全部清除。

### 4.1.2导出AssetBundle

我们在项目中新建Editor文件夹，并在其中创建一个脚本：

BuildAssetBungle.cs

```
using UnityEditor;

public class BuildAssetBundle
{

    [MenuItem("Assets/Build AssetBundles")]
    static void BuildAllAssetBundles()
    {
        BuildPipeline.BuildAssetBundles("Assets/MyAssetBundles");
    }
}
```

这样在菜单栏中，就会创建对应的按钮让我们执行构建操作。如图：

![build_btn.png](/assets/images/in-post/study_assetbundle/build_btn.png)

在点击之前需要在Assets目录下创建AssetBundles文件夹。点击之后将弹出一个进度条对话框，完成后就生成了对应的AssetBundles：

![assetbundles.png](/assets/images/in-post/study_assetbundle/assetbundles.png)

可以看到这些AssetBundles是按照我们此前在编辑器中新建的AssetBundle的目录结构生成的。并且有这相关的以.manifest为后缀的文件。一个manifest文件是描述了对应资源文件的循环冗余码（CRC）和资源依赖（asset dependencies）的文本文件。

另外，你还会看到在MyAssetBundles文件夹下有一个MyAssetBundles文件和对应的manifest文件。每当构建AssetBundles时，就会创建这两个文件。

### 4.1.3其他工具

添加下面这个脚本Editor中，可以使你获取所有AssetBundles的名字：

GetAssetBundleNames.cs:

```
using UnityEditor;
using UnityEngine;

public class GetAssetBundleNames
{

    [MenuItem("Assets/Get AssetBundle names")]
    static void GetNames()
    {
        var names = AssetDatabase.GetAllAssetBundleNames();
        foreach (var name in names)
            Debug.Log("AssetBundle: " + name);
    }

}
```

添加下面这个脚本Editor中，可以使你在改变AssetBundle时得到通知：

MyPostprocessor.cs:

```
using UnityEngine;
using UnityEditor;

public class MyPostprocessor : AssetPostprocessor
{

    void OnPostprocessAssetbundleNameChanged(string path,
            string previous, string next)
    {
        Debug.Log("AB: " + path + " old: " + previous + " new: " + next);
    }
}
```

### 4.1.4AssetBundle Variants

AssetBundle Variants是Unity5的新特性。你可以将两组不同的资源指定为同一个AssetBundle但是指定不同的Variants，这样，你可以根据平台的不同加载不同的资源（例如支持高清资源的设备加载hd的Variants，不支持高清资源的设备加载sd的Variants）。可是使用[AssetImporter.assetBundleVariant](http://docs.unity3d.com/ScriptReference/AssetImporter-assetBundleVariant.html)设置Variants。

### 4.1.5编码建议

1.标记AssetBundle

使用[AssetImporter](http://docs.unity3d.com/ScriptReference/AssetImporter.html).assetBundleName来设置AssetBundle的名字。

2.使用函数[BuildPipeline.BuildAssetBundles()](http://docs.unity3d.com/ScriptReference/BuildPipeline.BuildAssetBundles.html)构建AssetBundle

函数原型：

```
public static AssetBundleManifest BuildAssetBundles(string outputPath, BuildAssetBundleOptions assetBundleOptions = BuildAssetBundleOptions.None, BuildTarget targetPlatform = BuildTarget.WebPlayer);
```

3.操纵Asset database中的AssetBundle names接口：

[AssetDatabase.GetAllAssetBundleNames()](http://docs.unity3d.com/ScriptReference/AssetDatabase.GetAllAssetBundleNames.html)

[AssetDatabase.GetAssetPathsFromAssetBundle](http://docs.unity3d.com/ScriptReference/AssetDatabase.GetAssetPathsFromAssetBundle.html)

[AssetDatabase.RemoveAssetBundleName()](http://docs.unity3d.com/ScriptReference/AssetDatabase.RemoveAssetBundleName.html) 

[AssetDatabase.GetUnusedAssetBundleNames()](http://docs.unity3d.com/ScriptReference/AssetDatabase.GetUnusedAssetBundleNames.html)

[AssetDatabase.RemoveUnusedAssetBundleNames()](http://docs.unity3d.com/ScriptReference/AssetDatabase.RemoveUnusedAssetBundleNames.html) 

[AssetPostProcessor.OnPostprocessAssetbundleNameChanged](http://docs.unity3d.com/ScriptReference/AssetPostprocessor.OnPostprocessAssetbundleNameChanged.html) 

4.构建AssetBundle选项（BuildAssetBundleOptions）

[CollectDependencies](http://docs.unity3d.com/ScriptReference/BuildAssetBundleOptions.CollectDependencies.html) 和 [DeterministicAssetBundle](http://docs.unity3d.com/ScriptReference/BuildAssetBundleOptions.DeterministicAssetBundle.html) 选项总是启用的。

[CompleteAssets](http://docs.unity3d.com/ScriptReference/BuildAssetBundleOptions.CompleteAssets.html) 如果我们总是从assets开始而不是objects，该项将被忽略。它默认是完整的。

[ForceRebuildAssetBundle](http://docs.unity3d.com/ScriptReference/BuildAssetBundleOptions.ForceRebuildAssetBundle.html) 即便你没有改变资源，但是通过设置该项你可以强制从新构建资源。

[IngoreTypeTreeChanges](http://docs.unity3d.com/ScriptReference/BuildAssetBundleOptions.IgnoreTypeTreeChanges.html) 即便你改变了type tree你也可以通过设置该项忽略掉。

[DisableWriteTypeTree](http://docs.unity3d.com/ScriptReference/BuildAssetBundleOptions.DisableWriteTypeTree.html) 与 [IngoreTypeTreeChanges](http://docs.unity3d.com/ScriptReference/BuildAssetBundleOptions.IgnoreTypeTreeChanges.html)是冲突的，如果你将type tree设置为不启用则无法忽略。

5.Manifest file

每个AssetBundle都有一个manifest文件，包含如下信息：

**CRC（循环冗余码）**
资源文件的哈希码。在该AssetBundle中的所有资源有一个单一的哈希码，用于检查增量的构建。

**Type tree哈希码**。在该AssetBundle中所有类型有一个单一的哈希码，用于检查增量的构建。


**Class types**。该AssetBundle中所有的类类型。当为type tree做增量构建检查时将产生一个新的哈希码。

**Asset names**。该AssetBundle中所有明确包含的资源名字。依赖的AssetBundle的名字。依赖于该AssetBundle的其他AssetBundles。
manifest文件仅用于检查增量构建，运行时不需要。因此不需要打包进正式发行的游戏中。

6.Single manifest file

一个单一的manifest文件一般包含以下信息：
所有的AssetBundles。
所有AssetBundles的依赖信息。

7.Single manifest AssetBundle

一个**AssetBundleManifest**对象有如下APIs：

[GetAllAssetBundles()](http://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetAllAssetBundles.html) 返回本次构建的所有AssetBundles名字。

[GetDirectDependencies()](http://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetDirectDependencies.html) 返回直接依赖的AssetBundle名字。

[GetAllDependencies()](http://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetAllDependencies.html) 返回所有依赖的AssetBundle名字。

[GetAssetBundleHash(string)](http://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetAssetBundleHash.html) 返回指定的AssetBundle的哈希码。

[GetAllAssetBundlesWithVariant()](http://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetAllAssetBundlesWithVariant.html) 返回所有AssetBundles带Variant的名字。

8.AssetBundle 加载APIs

Unity5.x改为如下APIs:

[AssetBundle.GetAllAssetNames()](http://docs.unity3d.com/ScriptReference/AssetBundle.GetAllAssetNames.html) 返回该AssetBundle中的所有资源名。

[AssetBundle.GetAllScenePaths()](http://docs.unity3d.com/ScriptReference/AssetBundle.GetAllScenePaths.html) 如果该AssetBundle是一个场景文件，返回该场景中所有资源的路径。

[AssetBundle.LoadAsset()](http://docs.unity3d.com/ScriptReference/AssetBundle.LoadAsset.html) 从该AssetBundle中加载资源。

[AssetBundle.LoadAllAssets()](http://docs.unity3d.com/ScriptReference/AssetBundle.LoadAllAssets.html) 从该AssetBundle中加载所有资源。

[AssetBundle.LoadAssetWithSubAssets()](http://docs.unity3d.com/ScriptReference/AssetBundle.LoadAssetWithSubAssets.html) 通过名字加载该AssetBundle中的资源及子资源。
还有对应的异步接口也有提供。
组件类型不在返回了，你可以在加载了GameObject之后从其获取。

9.Typetrees

每个AssetBundle中都默认写入了一个typetree。只有Metro（Windows Store Apps）例外，它有不同的序列化方案。

---

参考链接：

http://docs.unity3d.com/Manual/AssetBundlesIntro.html

http://docs.unity3d.com/Manual/BuildingAssetBundles.html

http://docs.unity3d.com/ScriptReference/BuildPipeline.BuildAssetBundles.html