---
layout:     post
title:      "UGUI动态加载对话框"
subtitle:   ""
date:       2016-03-06 22:45:00
author:     "Allen"
header-img: "img/post-bg-2015.jpg"
tags:
    - 编程
---

#UGUI动态加载对话框

这是一篇关于使用Unity新的UI系统制作对话框并实现动态加载的教程。

#1.准备工作

##1.1新建一个Unity项目（Unity版本需要4.6以上）,2D项目3D项目均可。

##1.2如图设置Game窗口的Aspect为1920 x 1080
![设置Game窗口]({{ site.url }}/img/in-post/load_ugui_dialog_dynamically/game_aspect.jpg)

##1.3将Scene窗口调整为2D查看模式。

##1.4在Project窗口中新建Scenes，Scripts，Resources/Prefabs等文件夹，保存当前场景至Scenes文件夹。

#2.编辑UI对象

##2.1创建UIRoot
如图使用菜单新建一个Panel
![新建一个Panel]({{ site.url }}/img/in-post/load_ugui_dialog_dynamically/new_panel.jpg)
此时会在Hierarchy自动生成一个Canvas对象及其子对象Panel，一个EventSystem对象，如图：
![hierarchy]({{ site.url }}/img/in-post/load_ugui_dialog_dynamically/hierarchy.jpg)
讲Canvas重命名为UIRoot，并设置其Tag为UIRoot。

##2.2创建对话框及其他组件
###2.2.1创建对话框
将Panel对象重命名为Dialog。如图设置将其Rect Transform的Anchor Preset设置为middle - center,将其width设置为300，height设置为150。
![dialog_set]({{ site.url }}/img/in-post/load_ugui_dialog_dynamically/dialog-set.jpg)

###2.2.2创建其他组件
如上方法使用创建UI对象菜单依次为Dialog对象添加一个Text子对象，一个Button子对象。此时Hierarchy窗口如图所示：
![final_hierarchy]({{ site.url }}/img/in-post/load_ugui_dialog_dynamically/final_hierarchy.jpg)
将Text对象及Button对象的Rect Transform设置为如图所示:
Text:
![text]({{ site.url }}/img/in-post/load_ugui_dialog_dynamically/text.jpg)
Button:
![button]({{ site.url }}/img/in-post/load_ugui_dialog_dynamically/button.jpg)
其中这两个对象的文字部分设置为自己喜欢的即可。
这是完成后的Game窗口：
![final_game]({{ site.url }}/img/in-post/load_ugui_dialog_dynamically/final_game.jpg)

##2.3将编辑好的UI对象存成Prefab
将Dialog对象选中并拖动到Resources/Prefabs文件夹中，并删除Hierarchy下的Dialog对象，为动态加载做准备。

#3.动态加载

##3.1编辑脚本

在Scripts文件夹中新建C#脚本。
LoadDialog.cs:
{% highlight c# linenos %}
using UnityEngine;
using System.Collections;

public class LoadDialog : MonoBehaviour {
	
	void Start () {
		GameObject UIRoot = GameObject.FindWithTag("UIRoot");
		if(UIRoot != null)
		{
            Object dialogPrefab = Resources.Load("Prefabs/Dialog") as Object;
			if(dialogPrefab != null)
			{
				GameObject dialog = Instantiate(dialogPrefab) as GameObject;
				dialog.transform.SetParent(UIRoot.transform, false);
                dialog.transform.SetAsLastSibling();
            }
            else
            {
                Debug.Log("Failed to load prefab file");
            }
		}else
		{
            Debug.LogError("There is not a GameObject with tag UIRoot");
		}
	}
}
{% endhighlight %}

##3.2设置脚本
在MainCamera中添加LoadDialog脚本。

最后运行程序，查看效果。

我已将我的工程文件上传至[github](https://github.com/AllenKashiwa/StudyUnity)以供参考,运行UGUIDialog.unity即可。

—— Allen 丙申年正月廿八 于上海