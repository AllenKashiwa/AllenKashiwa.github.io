---
layout:     post
title:      "Unity5.x新的AssetBundle机制03——下载与使用"
date:       2016-03-22 14:47:00
---

# 前言

![unity_cover.png](/assets/images/in-post/study_assetbundle/unity_cover.png)

本节我们来讲讲AssetBundle的下载与使用。在第一节[Unity5.x新的AssetBundle机制01——构建](http://www.jianshu.com/p/d5d0a70c5626)中,我们已经准备好了需要的AssetBundles。本节我们就来使用这些保存好的资源。

# 实践

有两种方法可以下载AssetBundles:

## 非缓存式。

new一个WWW对象的方式，AssetBundles不会保留在本地设备的unity缓存文件夹中。
我们使用这种方式来实践一下。打开第一节中已经准备好的工程。新建Scripts文件夹，并在其中新建脚本：

NonCacheBundle.cs

```
using System;
using UnityEngine;
using System.Collections;
class NonCacheBundle : MonoBehaviour
{
	//根据平台，得到相应的路径
	public static readonly string BundleURL =
#if UNITY_ANDROID
		"jar:file://" + Application.dataPath + "!/assets/MyAssetBundles/shape/cube";
#elif UNITY_IPHONE
		Application.dataPath + "/Raw/MyAssetBundles/shape/cube";
#elif UNITY_STANDALONE_WIN || UNITY_EDITOR
		//我们将加载以前打包好的cube
 "file://" + Application.dataPath + "/MyAssetBundles/shape/cube";//由于是编辑器下，我们使用这个路径。
#else
        string.Empty;
#endif

	//还记得吗？在cube这个AssetBundle中有两个资源，Cube1和Cube2
	private string AssetName = "Cube1";

	IEnumerator Start()
	{
		// 从URL中下载文件，不会存储在缓存中。
		using (WWW www = new WWW(BundleURL))
		{
			yield return www;
			if (www.error != null)
				throw new Exception("WWW download had an error:" + www.error);
			AssetBundle bundle = www.assetBundle;
			GameObject cube = Instantiate(bundle.LoadAsset(AssetName)) as GameObject;
			cube.transform.position = new Vector3(0f, 0f, 0f);
			// 卸载加载完之后的AssetBundle，节省内存。
			bundle.Unload(false);

		}//由于使用using语法，www.Dispose将在加载完成后调用，释放内存
	}
}
```

将代码多拽到Hierarchy视口中的MainCamera物体中，运行游戏，可以看到cube1被加载到场景中。

## 缓存方式

使用[WWW.LoadFromCacheOrDownload](http://docs.unity3d.com/ScriptReference/WWW.LoadFromCacheOrDownload.html)接口。AssetBundles将保存在本地设备的Unity的缓存文件夹中。WebPlayer 有50MB的缓存上限，PC/Mac/Android/IOS应有有4 GB的缓存上限。这种方式也是加载AssetBundle推荐的方式。我们使用这种方式来实践一下。在Scripts文件夹中新建脚本：

CacheBundle.cs

```
using System;
using UnityEngine;
using System.Collections;

public class CacheBundle : MonoBehaviour
{
	public static readonly string BundleURL =
#if UNITY_ANDROID
		"jar:file://" + Application.dataPath + "!/assets/MyAssetBundles/shape/cube";
#elif UNITY_IPHONE
		Application.dataPath + "/Raw/MyAssetBundles/shape/cube";
#elif UNITY_STANDALONE_WIN || UNITY_EDITOR
		//我们将加载以前打包好的cube
 "file://" + Application.dataPath + "/MyAssetBundles/shape/cube";//由于是编辑器下，我们使用这个路径。
#else
        string.Empty;
#endif

	//还记得吗？在cube这个AssetBundle中有两个资源，Cube1和Cube2
	private string AssetName = "Cube2";

	//版本号
	public int version;

	void Start()
	{
		StartCoroutine(DownloadAndCache());
	}

	IEnumerator DownloadAndCache()
	{
		// 需要等待缓存准备好
		while (!Caching.ready)
			yield return null;

		// 有相同版本号的AssetBundle就从缓存中获取，否则下载进缓存。
		using (WWW www = WWW.LoadFromCacheOrDownload(BundleURL, version))
		{
			yield return www;
			if (www.error != null)
				throw new Exception("WWW download had an error:" + www.error);
			AssetBundle bundle = www.assetBundle;
			GameObject cube = Instantiate(bundle.LoadAsset(AssetName)) as GameObject;
			cube.transform.position = new Vector3(1.5f, 0f, 0f);
			// 卸载加载完之后的AssetBundle，节省内存。
			bundle.Unload(false);

		} //由于使用using语法，www.Dispose将在加载完成后调用，释放内存
	}
}
```

同样将该脚本拖动到MainCamera中，运行游戏，可以看到Cube2被加载进场景中。这是最终运行效果：

![run_assetbundle.png](/assets/images/in-post/study_assetbundle/run_assetbundle.png)

当我们访问www对象的.assetBundle属性时，下载好的数据被解压并创建了AssetBundle对象。此时就可以加载该AssetBundle中的所有资源。

## 使用异步方法

前两个例子中，我们都使用了AssetBundle类中的LoadAsset接口，这是一个同步的方法。这里提供一个异步的示例：

LoadAsyncBundle.cs

```
using UnityEngine;
using System;
using System.Collections;

public class LoadAsyncBundle : MonoBehaviour
{

	//根据平台，得到相应的路径
	public static readonly string BundleURL =
#if UNITY_ANDROID
		"jar:file://" + Application.dataPath + "!/assets/MyAssetBundles/shape/cube";
#elif UNITY_IPHONE
		Application.dataPath + "/Raw/MyAssetBundles/shape/cube";
#elif UNITY_STANDALONE_WIN || UNITY_EDITOR
 "file://" + Application.dataPath + "/MyAssetBundles/shape/sphere";
#else
        string.Empty;
#endif

	private string AssetName = "Sphere1";

	IEnumerator Start()
	{
		while (!Caching.ready)
			yield return null;
		using (WWW www = new WWW(BundleURL))
		{
			yield return www;
			if (www.error != null)
				throw new Exception("WWW download had an error:" + www.error);
			AssetBundle bundle = www.assetBundle;

			// 异步加载
			AssetBundleRequest request = bundle.LoadAssetAsync(AssetName, typeof(GameObject));

			// 等待加载完成
			yield return request;

			// 获取加载好的对象引用
			GameObject prefab = request.asset as GameObject;
			GameObject sphere = Instantiate(prefab);
			sphere.transform.position = new Vector3(0f, 1.5f, 0f);
			// 卸载加载完之后的AssetBundle，节省内存。
			bundle.Unload(false);

		}//由于使用using语法，www.Dispose将在加载完成后调用，释放内存
	}
}
```

至此，Unity5中新的AssetBundle机制的使用已经大致讲完啦。当然还有许多细节无法赘述，大家可以通过查看官方文档继续了解。有问题欢迎和我一起讨论。我已经将本工程上传到我的github，以供大家参考。请使用assetbundle这个分支。

---
参考链接：

http://docs.unity3d.com/Manual/DownloadingAssetBundles.html
http://docs.unity3d.com/Manual/LoadingAssetBundles.html