---
layout:     post
title:      "Just for fun——Unity5制作贪吃蛇小游戏"
date:       2016-04-10 22:17:00
---

# 前言

说出来你可能不信。有一天晚上睡不着觉，就突然想到做游戏开发这么久，还没亲手实现过经典游戏贪吃蛇呢。然后就在床上想实现的方法，越想越手痒，要不是怕把枕边人吵醒，第二天又要上班，真要立刻爬起来把这小游戏实现了。于是想了一会儿，理清思路之后就逼自己入睡了。

第二天下班之后，回到家就开始动手，已经搞定了。为了节约篇幅，本篇文章只简单说下思路。不想看我废话，需要直接参考代码的同学可以到我的[**Github**](https://github.com/AllenKashiwa/RetroSnaker)。

# 准备工作

我使用了Unity自带的创建简单3D物体的功能来制作蛇头和蛇身的prefab。蛇身直接用Cube，而**蛇头需要添加BoxCollider和Rigidbody组件**。为了将不同的物体区别开，我还写了一个纯色的shader：

```
Shader "Custom/PureColor" {
	Properties{
		_Color ("颜色", Color) = (0.8 ,0,0,0)
	}
	SubShader {
		Pass{
			Color[_Color]
		}
	}
	FallBack "Diffuse"
}

```

我们可以创建一个材质，将材质的shader设为Custom/PureColor，再将该材质赋给特定物体。可以用这个shader创建多个材质与其他物体区分开。
如图是我们需要的所有资源：

![retro_snaker_res.png](/assets/images/in-post/retro_snaker_res.png)

其中，所有的材质都使用上述的shader并指定不同的颜色。
除此之外，我们需要在Project Settings的Input中增加四个按钮：
UP,DOWN,LEFT,RIGHT
我们的关卡也很简单：
1.创建几个Sphere物体当做贪吃蛇的食物，设置位置为整数以便于我们的贪吃蛇在运动时能够吃个正着（因为Unity默认的Cube物体的边长是1，Sphere的半径是0.5）。
2.将SphereCollider的**半径设小一点**，例如0.25以便于我们的贪吃蛇碰到这些食物时看起来像真的吞噬了它们。
3.将SphereCollider**设为触发器**。
看起来像这样：


![retro_snaker_level](/assets/images/in-post/retro_snaker_level.png)

# 实现

首先定义一个枚举来代表节点运动的四个方向：

```
public enum MoveDir
{
	UP,
	LEFT,
	RIGHT,
	DOWN
}
```

我们仅需三个类来完成这个游戏的核心部分。
**RetroSnaker**负责游戏的主要控制，包括游戏状态更新，用户输入，产生新节点等。

部分代码如下：

```
	void Awake()
	{
		_snakeHeadPrefab = Resources.Load(_snakeHeadResPath) as GameObject;
		_snakeNodePrefab = Resources.Load(_snakeNodeResPath) as GameObject;
	}

	// Use this for initialization
	void Start()
	{
		RetroSnakerStart();
	}

	// Update is called once per frame
	void Update()
	{
		if (_isInPause)
			return;
		_timer += Time.deltaTime;
		if (_timer >= _timeGap)
		{
			_timer = 0.0f;
			MoveSnake();
		}
		ChangeMoveDir();
	}
```

**RetroSnakerStart**函数在游戏开始时生成一条起始蛇。**MoveSnake**函数主要调用当前所有节点的**Move**函数来完成运动。**ChangeMoveDir**函数接受用户输入并记录**转向**。

**SnakeNode**是组成贪吃蛇上每一个节点上的类。主要控制该节点运动。

部分代码如下：

```
	public void Move()
	{
		switch (moveDir)
		{
			case MoveDir.UP:
				MoveUp();
				break;
			case MoveDir.DOWN:
				MoveDown();
				break;
			case MoveDir.LEFT:
				MoveLeft();
				break;
			case MoveDir.RIGHT:
				MoveRight();
				break;
		}
	}
```

**Apple**是我们的贪吃蛇要吃的东西，将挂载到我们创建的Sphere物体上。

代码如下：

```
using UnityEngine;
using System.Collections;

public class Apple : MonoBehaviour
{
	public static int numberOfObjects = 0;

	#region mono
	// Use this for initialization
	void Start()
	{
		++numberOfObjects;
	}

	void OnTriggerEnter(Collider collider)
	{
		gameObject.SetActive(false);
		--numberOfObjects;
		RetroSnaker snake = GameObject.FindObjectOfType<RetroSnaker>();
		if (snake != null && snake.isActiveAndEnabled)
			snake.OnEatApple();
	}
	#endregion mono

}

```

在Apple类中我们使用一个静态成员numberOfObjects记录Apple的数量，这样当该成员变为0时，则我们吃光了Apple，完成关卡。

主要难点我感觉在于贪吃蛇接受输入后转向，其余节点在下次绘制时要跟随上一节点的运动方向。所以我用一个list来记录总共需要发生几次**转向**。用户每产生一个有效的输入就增加一个转向。每次绘制后逐一对这些转向遍历，并将改变了的索引加1，下次绘制时即可改变后一节点的运动方向。如果索引超出了贪吃蛇的长度，则可以移除该转向。
这里贴出完成一次移动后设置节点运动方向，以及移除一个转向的代码：

```
List<int> tempIndexs = new List<int>();
for (int i = 0; i < _indexs.Count; i++)
	{
		int index = _indexs[i];
		if (index > 0 && index < _snake.Count)
		{
			_snake[index].moveDir = _snake[index - 1].moveDir;
			_indexs[i]++;
		}
		if (_indexs[i] < _snake.Count)
			tempIndexs.Add(_indexs[i]);
	}
	_indexs = tempIndexs;
```

最后，还是建议你到我的[**Github**](https://github.com/AllenKashiwa/RetroSnaker)查看完整工程。Have Fun!