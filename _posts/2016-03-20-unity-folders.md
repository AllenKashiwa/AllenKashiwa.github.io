---
layout:     post
title:      "Unity3d开发中的特殊文件夹"
subtitle:   "\"Unity3d开发中的特殊文件夹\""
date:       2016-03-20 16:45:00
author:     "Allen"
header-img: "img/post-bg-2015.jpg"
tags:
    - Unity
---
#Assets
Assets文件夹是unity项目中放置游戏资源的主文件夹。该文件夹中的内容将直接反应在编辑器的Project视口中。许多系统API基于该文件夹。

#与脚本编译顺序相关的文件夹
在unity开发中，我们可以为自己的项目随意命名文件夹以管理游戏资源，包括代码资源。但unity保留了一些特殊文件夹用来做特殊用途，例如编译顺序。
unity的脚本编译有4个阶段（phase），脚本处在哪个编译阶段取决于脚本所在的文件夹。如果你的一些脚本需要引用一些别的文件夹中定义的类，则需要关心他们的编译顺序。你引用的类需要先于你的当前类编译。或者当你需要引用其他语言的脚本时，那么该脚本必须处于更早的编译阶段。
unity中的4个编译阶段如下：
1.处于 **Standard Assets**, **Pro Standard Assets** 和 **Plugins** 文件夹中的运行时脚本（Standard Assets 文件夹需是Assets的一级子文件夹.）
2.处于 **Standard Assets**, **Pro Standard Assets** 和 **Plugins** 文件夹下的以 **Editor** 命名的一级子文件夹中的脚本。
3.顶级 **Editor** 文件夹中的脚本。
4.其他 **Editor** 文件夹中的脚本（例如其他文件夹下的以 **Editor** 命名的子文件夹）
另外在 **Assets** 文件夹中以 **WebPlayerTemplates** 命名的顶级子文件（即Assets/WebPlayerTemplates）将不会编译。若是处于别的文件夹下的 **WebPlayerTemplates** （如Assets/Scripts/WebPlayerTemplates）将不会防止编译。
##注意：
**Editor** 文件夹中的脚本主要用来扩展unity编辑器的功能方便开发。这些脚本将不会打包进最终发布的游戏中。项目中可以使用多个 **Editor** 文件夹，但是该文件夹中的脚本不允许用当GameObject对象的组件（Component）。

**Plugins** 文件夹中存放用于扩展unity功能的插件（多为C/C++写成的原生动态链接库(DLLs)）。这些插件可以访问第三方代码库，系统API以及其他超出Unity功能的模块。

#Editor Default Resources
我们使用 **Editor** 文件夹中的脚本扩展unity编辑器的功能时，可以使用函数[EditorGUIUtility](http://docs.unity3d.com/ScriptReference/EditorGUIUtility.html).Load 加载资源。该函数将优先加载Assets下的以 **Editor Default Resources** 命名的一级子目录。如果没有找到将尝试通过名字查找内置于编辑器中的资源。

#Gizmos
Unity的[Gizmos](http://docs.unity3d.com/ScriptReference/Gizmos.html)类可在Scene视口中绘制图像用来显示设计细节。[Gizmos.DrawIcon](http://docs.unity3d.com/ScriptReference/Gizmos.DrawIcon.html)函数可以在场景视口中绘制一个图标以标记特殊的对象和位置。该函数使用的图像文件需要位于 **Gizmos** 中。

#Resources
Unity允许你按需动态加载游戏资源到场景中。[Resources.Load](http://docs.unity3d.com/ScriptReference/Resources.Load.html) 函数可以加载项目中位于任何位置的 **Resources** 文件夹中的资源。你可以有多个Resources文件夹，不管是否是顶级文件夹都可以。

#StreamingAssets
当你需要使用某种保留原格式的资源，而不是经过Unity处理过的格式资源时，你可以将该资源放置于 **StreamingAssets** 文件夹中。该文件夹中的资源将在游戏安装时原样拷贝到目标设备相应的文件夹下。可参考[Streaming Assets](http://docs.unity3d.com/Manual/StreamingAssets.html)查看详情。

#Hidden Assets
在导入阶段，Unity将完全忽略以下文件夹下的资源。
·以 **Hidden** 命名的文件夹。
·以 ‘.’ 开头的文件和文件夹
·以 ‘~’ 开头的文件和文件夹
·以 ‘cvs’ 命名的文件和文件
. 以 ‘tmp’ 为扩展名的文件

[参考链接][1]:http://docs.unity3d.com/Manual/SpecialFolders.html
[参考链接][2]:http://docs.unity3d.com/Manual/ScriptCompileOrderFolders.html