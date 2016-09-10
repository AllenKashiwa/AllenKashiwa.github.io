---
layout:     post
title:      "Unity层级浅谈"
date:       2016-07-24 11:52:00
---

![封面来源于网络](/assets/images/in-post/unity_cover.png)

# 前言

用于管理本知识集所有源代码的github库此前一直以分支的形式对应不同的知识点。现在开来这样做是有点麻烦的。各位拉取之后还得迁到对应分支。我现在改为全部在master下新建项目的形式，大家拉取之后打开对应的项目就行了。原本的master分支我已经迁出一个OldMaster保留下来。

好了，言归正传，我们来谈谈今天的话题，Unity中的层级(layer).

# 什么是Layer

层级是Unity中场景物体的一种属性。摄像机可以指明渲染层级以渲染场景中的部分物体。灯光可以指明照射层级以照亮部分物体(可以指定照亮某些层级的物体以显示阴影)。层级还能用于设置物理碰撞关系。

# 添加与设置Layer

添加与设置Layer都可以选中一个物体之后在Inspector面板中操作。如下图所示：

![查看MainCamera的层级](/assets/images/in-post/maincamera_layer.png)

![查看已有层级](/assets/images/in-post/exist_layers.png)

Unity内置了5个层级，并预留了3个层级给引擎使用。用户可以自定义序号为8至31的层级。总共有32个层级是方便用户使用int类型做位操作取得想要的层级组合。

# 用法实验

## 摄像机与灯光

我们来创建一个简单的场景，以更好的领会层级的用法。

新建一个Unity工程，默认场景中已经有一个MainCamera和一个Directional Light。新建一个Plane作为我们的地面。新建一个Cube物体为其添加Cube层级。新建一个Sphere物体为其添加Sphere层级。新建一个Capsule物体保留Default层级。保存场景为UseLayer。

指定MainCamera的Culling Mask为不拍摄Sphere，如图所示：

![摄像机的拍摄层级](/assets/images/in-post/camera_culling_mask.png)

指定Directional Light的照射层级为不包含Cube，如图所示：

![Directional Light的照射层级](/assets/images/in-post/light_culling_mask.png)

调整物体与摄像机的位置及角度，使摄像机的拍摄视锥能够包含场景中的所有物体，最终可以达到如下图的效果：

![运行效果](/assets/images/in-post/runtime_shadow.png)

可以看到我们拍摄了Game视图中没有渲染Sphere物体和其影子，渲染了Cube物体却没有影子。Scene视图的摄像机是渲染所有层级物体的，可以用于比较。在Scene视图中Sphere是有影子的，说明我们的Directional Light是照射了Sphere层级的物体的。Capsule物体只用于比较。

## 物理检测

层级另外的用法是物理检测。点击菜单**Edit->Project Settings->Physics**.可以查看项目的物理碰撞设置：

![物理层级碰撞矩阵](/assets/images/in-post/physics_matrix.png)

这里的设置将影响OnCollisionEnter等消息的发送。

而射线检测本就要去指定检测层级：

```
int layerMask = LayerMask.NameToLayer("Cube");
if (Physics.Raycast(transform.position, Vector3.forward, Mathf.Infinity, layerMask))
	Debug.Log("The ray hit the Cube");
```

# 关于子物体层级的设置与讨论

在游戏中，我们经常需要动态指定物体的层级，而Unity目前没有提供指定物体子节点的层级的接口。如果只是简单的设置物体的layer属性，是无法改变物体子节点的层级的。我们需要自己实现改变子物体层级的接口。

Unity的官方社区和论坛都有人讨论这个问题。这里附上链接供大家参考：

http://forum.unity3d.com/threads/change-gameobject-layer-at-run-time-wont-apply-to-child.10091/

http://answers.unity3d.com/questions/168084/change-layer-of-child.html

http://answers.unity3d.com/questions/26479/fast-layer-assignment.html

我参照网上的做法也写了一个测试。在刚刚的工程中新建一个场景，命名为SetLayer。新建脚本如下，并挂载MainCamera下：

```
using System.Collections.Generic;
using UnityEngine;

public class SetLayer : MonoBehaviour
{

	void Start()
	{
		GameObject root = Util.CreateHeirarchy();

		float startTime = Time.realtimeSinceStartup;
		Util.SetLayerOnAll(root, LayerMask.NameToLayer("Cube"));
		float totalTimeMs = (Time.realtimeSinceStartup - startTime) * 1000;
		print("Set layer on all time: " + totalTimeMs + "ms");

		startTime = Time.realtimeSinceStartup;
		Util.SetLayerRecusively(root, LayerMask.NameToLayer("Sphere"));
		totalTimeMs = (Time.realtimeSinceStartup - startTime) * 1000;
		print("Set layer on all recursive time: " + totalTimeMs + "ms");

		startTime = Time.realtimeSinceStartup;
		Util.SetLayerNotRecusively(root.transform, LayerMask.NameToLayer("Cube"));
		totalTimeMs = (Time.realtimeSinceStartup - startTime) * 1000;
		print("Set layer on not recursive time: " + totalTimeMs + "ms");
	}

}

public static class Util
{
	public static GameObject CreateHeirarchy()
	{
		GameObject root = new GameObject();

		GameObject[] children = new GameObject[100];
		for (int i = 0; i < 100; i++)
		{
			GameObject child = new GameObject();
			child.transform.parent = root.transform;
			children[i] = child;
		}

		GameObject[] grandchildren = new GameObject[1000];
		for (int i = 0; i < 1000; i++)
		{
			GameObject grandchild = new GameObject();
			grandchild.transform.parent = children[Random.Range(0, 99)].transform;
			grandchildren[i] = grandchild;
		}


		for (int i = 0; i < 10000; i++)
		{
			GameObject greatgrandchild = new GameObject();
			greatgrandchild.transform.parent = grandchildren[Random.Range(0, 999)].transform;
		}

		return root;
	}

	public static void SetLayerOnAll(GameObject obj, int layer)
	{
		if (null == obj)
			return;

		foreach (Transform trans in obj.GetComponentsInChildren<Transform>(true))
		{
			trans.gameObject.layer = layer;
		}

	}

	public static void SetLayerRecusively(GameObject obj, int layer)
	{
		if (null == obj)
			return;

		obj.layer = layer;
		foreach (Transform child in obj.transform)
			SetLayerRecusively(child.gameObject, layer);
	}

	public static void SetLayerNotRecusively(Transform root, int layer)
	{
		Stack<Transform> moveTargets = new Stack<Transform>();
		moveTargets.Push(root);
		Transform currentTarget;
		while (moveTargets.Count != 0)
		{
			currentTarget = moveTargets.Pop();
			currentTarget.gameObject.layer = layer;
			foreach (Transform child in currentTarget)
				moveTargets.Push(child);
		}
	}
}

```

测试结果是三种写法消耗时间差不多，普遍GetComponentsInChildren更快。

# 结束语

如果你喜欢本文，那就点个喜欢吧。本文的github库在[这里](https://github.com/AllenKashiwa/StudyUnity)，欢迎大家fork。