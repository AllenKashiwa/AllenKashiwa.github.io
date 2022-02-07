---
layout:     post
title:      "Unity如何自动生成动画状态机"
date:       2016-10-07 20:42:00
---

![封面来源于网络](/assets/images/in-post/unity_cover.png)

# 前言
不好意思，拖更几个月了，最近屁股先锋中毒太深。今天来说说如何在Unity的编辑模式下自动生成动画状态机。

# 为何需要自动生成动画状态机？
游戏角色的动画状态中有许多相似的状态，每个角色都需要创建动画状态机，创建诸如Idle,Run,Walk,NormalAttak等相似的状态，甚至状态机的迁移条件也有雷同。为了不然创建这些相似的状态机并指定状态中相对应的动画这类繁琐的事全部交给手工操作，自动化生成这些状态机成为理所当然。

# 如何自动化生成动画状态机？
## 官方参考
官方文档中有关于这个的一份[参考代码](https://docs.unity3d.com/ScriptReference/Animations.AnimatorController.html)。你可以从中大致知道这一流程，推荐大家看一看。本文将在此基础上完整演示从美术资源到生成状态机乃至角色prefab的整个过程。包含指定动画融合，状态机分层等知识点。

## 实例演示
本文提供的实例演示的所有代码及资源将提供在github库中的AnimatorFactory这个项目中。欢迎大家fork。

### 准备工作
为了模拟从美术资源到产出状态机的全过程，我们需要准备我们的资源。官方提供的Standard Assets正好可以拿来使用。下载Standard Assets可以在官方网站的如下位置找到：

找到Unity旧版本：

![image](http://baizihan.com/assets/images/in-post/old_unity.png)

找到对应版本的标准资源

![image](http://baizihan.com/assets/images/in-post/standard_assets.png)

我们创建新的项目，并导入资源：

![image](http://baizihan.com/assets/images/in-post/import_package.png)

我们仅保留动画和模型资源和一些文件结构：

![image](http://baizihan.com/assets/images/in-post/show_assets.png)

### 自动化流程

我们在Editor目录下创建一个名为AnimatorFactory的CSharp文件，我们的自动化流程将一步一步通过完善这个脚本实现。至于为何要在Editor目录创建，可以参考我的这篇文章:[Unity3d开发中的特殊文件夹](http://baizihan.com/2016/03/unity-folders/)。

#### 1 准备加载动画的工具类

在AnimatorFactory脚本中添加如下代码：

```

using System.Linq;
using UnityEngine;
using UnityEditor;
using UnityEditor.Animations;

public static class AnimatorFactoryUtil
{
	public static AnimationClip LoadAnimClip(string path)
	{
		return (AnimationClip)AssetDatabase.LoadAssetAtPath(path, typeof(AnimationClip));
	}

	public static AnimationClip LoadAnimClip(string fbxPath, string animPath)
	{
		var objs = AssetDatabase.LoadAllAssetsAtPath(fbxPath);
		return objs.Where(o => o is AnimationClip && o.name.Equals(animPath)).Select(o => o as AnimationClip).FirstOrDefault();
	}
}

```

我们的动画资源不在Resources文件夹下，这里使用AssetDatabase的LoadAssetAtPath方法加载动画。多参数版的函数演示了在同一个fbx文件下加载指定动画的方法。

#### 2 创建窗口

在AnimatorFactory脚本中添加如下代码：

```

public class AnimatorFactory : EditorWindow
{
	#region const
	private const string AnimatorSavePath = "Assets/Standard Assets/Characters/ThirdPersonCharacter/Animator/";
	private const string PrefabSavePath = "Assets/Standard Assets/Characters/ThirdPersonCharacter/Prefabs/";
	private const string ModelPath = "Assets/Standard Assets/Characters/ThirdPersonCharacter/Models/";
	private const string AnimationPath = "Assets/Standard Assets/Characters/ThirdPersonCharacter/Animation/Humanoid";
	private const string AnimatorControllerSuffix = "AnimatorController.controller";
	#endregion const

	#region private members
	private string characterName;
	private AnimatorController productController;
	private AnimatorStateMachine baseLayerMachine;
	private AnimatorStateMachine crouchLayerMachine;
	// base layer states
	private AnimatorState stateIdle;
	private AnimatorState stateMove;
	private AnimatorState stateJump;
	private AnimatorState stateDeath;
	// crouch layer states
	private AnimatorState stateWalk;
	private AnimatorState stateWalkLeft;
	private AnimatorState stateWalkRight;
	#endregion private members

	#region EditorWindow
	[MenuItem("Window/AnimatorFactory")]
	public static void OpenWindow()
	{
		EditorWindow.GetWindow(typeof(AnimatorFactory));
	}

	void OnEnable()
	{
		// set default name
		characterName = "Ethan";
	}

	void OnGUI()
	{
		GUILayout.Label("Animation Settings", EditorStyles.boldLabel);
		characterName = EditorGUILayout.TextField("Character Name： ", characterName);

		if (GUILayout.Button("Generate Controller"))
			GenerateController();
	}
	#endregion EditorWindow
	
```

这样在Window菜单栏下就会有一个AnimatorFactory的菜单项，点击会打开这样一个窗口：

![image](http://baizihan.com/assets/images/in-post/animator_factory_window.png)

EditorWindow可以用来扩展我们的Unity，开发一些我们自己需要的工具，更多相关信息可以参考Unity的官方文档：
https://docs.unity3d.com/Manual/ExtendingTheEditor.html

定制编辑器的话题，有机会本站也会继续和大家交流。

**Character Name** 参数用于指定我们想要生成的角色的名字，这里是Standard Asset中的**Ethan**，所以我们在OnEnable函数中指定默认的角色名。
**Generate Controller** 按钮用于执行GenerateController函数，生成我们的动画状态机。

#### 3 创建AnimatorController

```

    public void GenerateController()
	{
		if (string.IsNullOrEmpty(characterName))
			return;

		// create animator controller
		CreateController();
		// add controller parameters
		AddParameters();
		// Create Anim States
		CreateAnimStates();
		// Bind aniamtor controller to prefab
		BindControllerToPrefab();
	}
	
```

这里分四步逐渐创建我们的状态机并创建人物Prefab。先来看第一步:

```

	/// <summary>
	/// Show how to create a controller, and set layer parameters
	/// </summary>
	private void CreateController()
	{
		productController = AnimatorController.CreateAnimatorControllerAtPath(AnimatorSavePath + characterName + AnimatorControllerSuffix);
		// get base machine in base layer
		baseLayerMachine = productController.layers[0].stateMachine;
		// set base machine parameters
		baseLayerMachine.entryPosition = Vector3.zero;
		baseLayerMachine.exitPosition = new Vector3(400f, 200f);
		baseLayerMachine.anyStatePosition = new Vector3(0f, 200f);

		// add crouch layer to controller
		productController.AddLayer("CrouchLayer");
		// get a copy from controller's layer
		AnimatorControllerLayer[] layers = productController.layers;
		// set layer parameters
		layers[1].defaultWeight = 1f;
		layers[1].blendingMode = AnimatorLayerBlendingMode.Override;
		// save layer setting to controller
		productController.layers = layers;
		// get state machine in crouch layer
		crouchLayerMachine = productController.layers[1].stateMachine;
		// set crouch machine parameters
		crouchLayerMachine.entryPosition = Vector3.zero;
		crouchLayerMachine.exitPosition = new Vector3(600f, 200f);
		crouchLayerMachine.anyStatePosition = new Vector3(0f, 200f);
	}

```

我特意添加了一个**CrouchLayer**层级，并演示设置其参数的方法。这里需要注意的是productController.layers这个属性返回的是一个拷贝而不是引用，所以直接改变层上的defaultWeight等参数不会生效，需要将设置好参数后的层级信息赋回layers。

#### 3 指定参数

```
/// <summary>
	/// Show how to add parameters to the controller
	/// </summary>
	private void AddParameters()
	{
		// use AddParameter interface
		productController.AddParameter("FloatA", AnimatorControllerParameterType.Float);
		productController.AddParameter("FloatB", AnimatorControllerParameterType.Float);
		productController.AddParameter("TriggerA", AnimatorControllerParameterType.Trigger);
		productController.AddParameter("TriggerB", AnimatorControllerParameterType.Trigger);
		productController.AddParameter("TriggerC", AnimatorControllerParameterType.Trigger);
		productController.AddParameter("BooleanA", AnimatorControllerParameterType.Bool);
		// if you want to set default value
		AnimatorControllerParameter playSpeed = new AnimatorControllerParameter();
		playSpeed.name = "PlaySpeed";
		playSpeed.type = AnimatorControllerParameterType.Float;
		playSpeed.defaultFloat = 1.0f;
		productController.AddParameter(playSpeed);
	}
```

我们知道动画状态机的迁移条件可能需要用到参数，这里展示了添加动画参数的两种方式，一种指定名字和类型就可以，另一种可以设置默认值。

#### 4 创建动画状态并绑定动画

```

private void CreateAnimStates()
	{
		// Create base layer states
		CreateBaseLayerState();
		// Create crouch layer states
		CreateCrouchLayerState();
	}

	private void CreateBaseLayerState()
	{
		CreateIdle();
		CreateMove();
		CreateJump();
		CreateDeath();
		SetBaseLayerTransition();
	}

	private void CreateCrouchLayerState()
	{
		// Load Animation
		string fbxPath = AnimationPath + "Crouch.FBX";
		CreateCrouchIdle(fbxPath);
		CreateCrouchWalk(fbxPath);
	}

```

由于想要全面展示创建各个层级的状态机，子状态机，以及普通状态，融合树状态等各个知识点，这部分内容会比较繁琐，请耐心往下看：

```

/// <summary>
	/// Show how to add a basic state, And add a behaviour
	/// </summary>
	private void CreateIdle()
	{
		// Load Animation
		AnimationClip idleClip = AnimatorFactoryUtil.LoadAnimClip(AnimationPath + "Idle.FBX");

		// add tree state & set state motion
		stateIdle = baseLayerMachine.AddState("Idle", new Vector3(300f, 0f));
		stateIdle.motion = idleClip;

		// Add behaviour to state
		stateIdle.AddStateMachineBehaviour<CharacterIdleState>();

		// set to default state
		baseLayerMachine.defaultState = stateIdle;
	}

```

**CreateIdle** 展示了如何创建一个普通的动画状态并指定动画以及给该状态添加一个Behavior的过程（CharacterIdleState脚本中是一个继承自StateMachineBehaviour的空实现类）。加载动画用到了前面准备好的AnimatorFactoryUtil类。

```
	/// <summary>
	/// Show how to add a 1D tree
	/// </summary>
	private void CreateMove()
	{
		// Load Animation
		AnimationClip walkClip = AnimatorFactoryUtil.LoadAnimClip(AnimationPath + "Walk.FBX");
		AnimationClip runClip = AnimatorFactoryUtil.LoadAnimClip(AnimationPath + "Run.FBX");

		// new a tree
		BlendTree tree = new BlendTree();

		// Set blendtree parameters
		tree.name = "Move";
		tree.blendType = BlendTreeType.Simple1D;
		tree.useAutomaticThresholds = true;
		tree.minThreshold = 0f;
		tree.maxThreshold = 1f;
		tree.blendParameter = "FloatA";

		// Add clip to BlendTree
		tree.AddChild(walkClip, 0f);
		tree.AddChild(runClip, 1f);

		// Add tree to controller asset
		if (AssetDatabase.GetAssetPath(productController) != string.Empty)
		{
			AssetDatabase.AddObjectToAsset(tree, AssetDatabase.GetAssetPath(productController));
		}

		// add tree state & set state motion
		stateMove = baseLayerMachine.AddState(tree.name, new Vector3(600f, 0f));
		stateMove.motion = tree;
	}

```

**CreateMove** 展示了如何创建一维融合树。这里之所以使用AddObjectToAsset接口是参考了Unity源码中AnimatorController类中的接口**CreateBlendTreeInController**。如果直接使用后文将提及的SetStateEffectiveMotion接口或直接指定stateMove的motion属性，生成controller时一切正常，运行后融合树就失效了。猜测原因，应该是CreateBlendTreeInController里由于做了AddObjectToAsset操作保留了融合树，而使用SetStateEffectiveMotion接口或直接指定stateMove的motion属性没有这个操作。那我为什么一定要固执的使用这个方法，而不直接CreateBlendTreeInController呢？你们猜？（因为这个接口无法指定该状态在animator窗口中的位置）。

```

	/// <summary>
	/// Show how to add 2D tree
	/// </summary>
	private void CreateJump()
	{
		// Load Animation
		string fbxPath = AnimationPath + "IdleJumpUp.FBX";
		AnimationClip fallClip = AnimatorFactoryUtil.LoadAnimClip(fbxPath, "HumanoidFall");
		AnimationClip idleJumpUpClip = AnimatorFactoryUtil.LoadAnimClip(fbxPath, "HumanoidIdleJumpUp");
		AnimationClip jumpUpClip = AnimatorFactoryUtil.LoadAnimClip(fbxPath, "HumanoidJumpUp");
		AnimationClip midAirClip = AnimatorFactoryUtil.LoadAnimClip(fbxPath, "HumanoidMidAir");

		// create a tree
		BlendTree tree = new BlendTree();

		// Set blendtree parameters
		tree.name = "Jump";
		tree.blendType = BlendTreeType.FreeformDirectional2D;
		tree.useAutomaticThresholds = true;
		tree.minThreshold = 0f;
		tree.maxThreshold = 1f;
		tree.blendParameter = "FloatA";
		tree.blendParameterY = "FloatB";

		// Add clip to BlendTree
		tree.AddChild(fallClip, new Vector2(0f, 0f));
		tree.AddChild(idleJumpUpClip, new Vector2(0f, 1f));
		tree.AddChild(jumpUpClip, new Vector2(1f, 0f));
		tree.AddChild(midAirClip, new Vector2(-1f, 0f));

		// Add tree to controller asset
		if (AssetDatabase.GetAssetPath(productController) != string.Empty)
		{
			AssetDatabase.AddObjectToAsset(tree, AssetDatabase.GetAssetPath(productController));
		}

		// add tree state & set state motion
		stateJump = baseLayerMachine.AddState(tree.name, new Vector3(300f, -100f));
		stateJump.motion = tree;
	}

```

**CreateJump** 展示了如何创建一棵二维融合树。

```

	/// <summary>
	/// Show how to relate exitState and anyState
	/// </summary>
	private void CreateDeath()
	{
		AnimationClip deathClip = AnimatorFactoryUtil.LoadAnimClip(AnimationPath + "WalkTurn.FBX");
		stateDeath = baseLayerMachine.AddState("Death", new Vector3(200f, 100f));
		productController.SetStateEffectiveMotion(stateDeath, deathClip);

		// death to exit
		var exitTransition = stateDeath.AddExitTransition();
		exitTransition.AddCondition(AnimatorConditionMode.If, 0, "TriggerA");
		exitTransition.duration = 0;

		// anyState to death
		var anyTransition = baseLayerMachine.AddAnyStateTransition(stateDeath);
		anyTransition.AddCondition(AnimatorConditionMode.If, 0, "TriggerB");
		anyTransition.duration = 0;
	}

```

**CreateDeath** 展示了如何指定一个状态机的AnyState的迁移，以及Exit迁移。

```

/// <summary>
	/// Show how to add transitions
	/// </summary>
	private void SetBaseLayerTransition()
	{
		var trans = stateIdle.AddTransition(stateMove);
		trans.hasExitTime = true;
		trans.exitTime = 0.9f;
		trans.interruptionSource = TransitionInterruptionSource.Source;
		trans.duration = 0;

		trans = stateMove.AddTransition(stateIdle);
		trans.interruptionSource = TransitionInterruptionSource.Destination;
		trans.duration = 0;
		trans.AddCondition(AnimatorConditionMode.If, 0, "TriggerA");
	}

```

**SetBaseLayerTransition** 展示了如何添加状态迁移，设置exitTime，设置打断源，设置条件等。到这里，第一层动画状态机的演示到此结束。接下来我们开始创建**CrouchLayer**层的状态机。

```

/// <summary>
	/// Show how to set speed parameter, And set state motion in other way
	/// </summary>
	/// <param name="fbxPath"></param>
	private void CreateCrouchIdle(string fbxPath)
	{
		// Load Animation
		AnimationClip idleClip = AnimatorFactoryUtil.LoadAnimClip(fbxPath, "HumanoidCrouchIdle");

		stateIdle = crouchLayerMachine.AddState("Idle", new Vector3(300f, 0f));
		stateIdle.speedParameterActive = true;
		stateIdle.speedParameter = "FloatA";

		// Set state motion,the other way
		productController.SetStateEffectiveMotion(stateIdle, idleClip);
		// set to default state
		crouchLayerMachine.defaultState = stateIdle;
	}

```

**CreateCrouchIdle** 展示了如何设置动画播放速度参数，以及另一种指定动画状态的动画的方法，即使用SetStateEffectiveMotion。

```

/// <summary>
	/// Show how to add a child machine
	/// </summary>
	/// <param name="fbxPath"></param>
	private void CreateCrouchWalk(string fbxPath)
	{
		AnimatorStateMachine walkMachine = crouchLayerMachine.AddStateMachine("Walk", new Vector3(0f, 100f));
		walkMachine.entryPosition = Vector3.zero;
		walkMachine.anyStatePosition = new Vector3(0f, -200f);
		walkMachine.exitPosition = new Vector3(0f, -400f);

		AnimationClip walkClip = AnimatorFactoryUtil.LoadAnimClip(fbxPath, "HumanoidCrouchWalk");
		AnimationClip walkLeftClip = AnimatorFactoryUtil.LoadAnimClip(fbxPath, "HumanoidCrouchWalkLeft");
		AnimationClip walkRightClip = AnimatorFactoryUtil.LoadAnimClip(fbxPath, "HumanoidCrouchWalkRight");
		stateWalk = walkMachine.AddState("Walk", new Vector3(200f, 0f));
		stateWalk.motion = walkClip;
		stateWalkLeft = walkMachine.AddState("WalkLeft", new Vector3(200f, -200f));
		stateWalkLeft.motion = walkLeftClip;
		stateWalkRight = walkMachine.AddState("WalkRight", new Vector3(200f, -400f));
		stateWalkRight.motion = walkRightClip;
	}

```

**CreateCrouchWalk** 展示了如何添加一个子状态机。到这里创建状态机的各个常用知识点基本都涉及了。

#### 5 绑定状态机到Prefab
```

private void BindControllerToPrefab()
	{
		// Generate prefab first time
		var prefab = AssetDatabase.LoadAssetAtPath(ModelPath + characterName + ".fbx", typeof(GameObject)) as GameObject;
		var go = Instantiate(prefab);
		go.name = characterName + "Prefab";

		var animator = go.GetComponent<Animator>() ?? go.AddComponent<Animator>();

		animator.runtimeAnimatorController = productController;
		PrefabUtility.CreatePrefab(PrefabSavePath + go.name + ".prefab", go);
		DestroyImmediate(go);

		// If there has been a prefab, just bind controller to it.
		//var prefab = AssetDatabase.LoadAssetAtPath(PrefabSavePath + characterName, typeof(GameObject)) as GameObject;
		//var go = Instantiate(prefab) as GameObject;

		//var animator = go.GetComponent<Animator>() ?? go.AddComponent<Animator>();

		//animator.runtimeAnimatorController = productController;
		//PrefabUtility.ReplacePrefab(go, prefab);
		//DestroyImmediate(go);
	}

```

我们可以在指定的位置生成角色的Prefab，启用的代码展示第一次创建Prefab的情况。注释的代码展示绑定我们生成的AnimatorController到已存在的指定的Prefab上。来看看我们的成果：

![image](http://baizihan.com/assets/images/in-post/product_controller.png)

# 后记
写了好长，希望对你有帮助，提升创建Prefab的效率。本文的所有演示文件都在[我的Github](https://github.com/AllenKashiwa/StudyUnity)上。下次再见。