---
layout:     post
title:      "Unity预设序列化的自动化方案"
date:       2017-02-26 18:30:00
---

# 预设

在Unity中，我们会将重复使用的资源做成预设（Prefab）。预设上可以挂载脚本并在Aspector面板中指定可序列化的值便于配置。

在实际操作过程中，我们的脚本上需要配置的部分可能常常需要指定一些固定的组件。加入有这样的一个脚本：

![image](http://baizihan.com/assets/images/in-post/asset_helper/example.png)

像这样，rigBody和capsuleCollider如果在游戏运行时获取会有些消耗，而在Editor模式下又需要我们每次从Hierarchy面板拖动到Aspector面板来指定费时费力。而这一切我们可以通过脚本自动完成。我们在Monster脚本上添加如下接口：

```
public void DoSerialized()
{
	rigBody = GetComponent<Rigidbody>();
	capsuleCollider = GetComponent<CapsuleCollider>();
}
```

# Apply
Unity提供了在预设Apply时对该预设进行操作的Attribute：**InitializeOnLoadMethod**

我们在Editor目录下创建一个脚本AssetHelper，实现该接口：

```
static void StartInitializeOnLoadMethod()
{
    // 注册Apply时的回调
    PrefabUtility.prefabInstanceUpdated = delegate(GameObject instance)
    {
        if(instance)
        SaveMonsterPrefab(instance);
    };
}    

static void SaveMonsterPrefab(GameObject instance)
{
    string prefabPath = AssetDatabase.GetAssetPath(PrefabUtility.GetPrefabParent(instance));
    if(!IsMonsterPrefab(prefabPath))
        return;

    Debug.LogFormat("SaveMonsterPrefab Path = {0}", prefabPath);
    Monster comp = instance.GetComponent<Monster>();
    if (null == comp)
    {
        string msg = string.Format("{0} 缺少Monster组件", prefabPath);
        EditorUtility.DisplayDialog("Apply a Monster prefab!", msg, "OK");
        return;
    }

    comp.DoSerialized();
}


static bool IsMonsterPrefab(string path){
    if(path.Contains(MONSTER_FOLDER) && Path.GetExtension(path) == ".prefab")
        return true;

    return false;
}

```

以后创建Prefab的时候就可以直接添加一个Monster脚本，按一下Apply就行了。

# 保存

Apply的方法有个明显的缺点，那就是Prefab的改动有时候不会拖到Hierarchy中，而是直接修改，然后Ctrl+S保存，从而没有Apply过程。不用担心，即便这样Unity也有解决方案。

我们将AssetHelper类继承**UnityEditor.AssetModificationProcessor**类，并实现**OnWillSaveAssets**方法。这样就可以在资源保存时对资源做操作。实现细节如下：

```
static string[] OnWillSaveAssets(string[] paths){
    SaveMonsterPrefabs(paths);
    return paths;
}

static void SaveMonsterPrefabs(string[] paths)
{
    foreach (string path in paths)
    {
        if(!IsMonsterPrefab(path))
            continue;

        Debug.LogFormat("SaveMonsterPrefabs {0}", path);

        GameObject prefab = (GameObject)AssetDatabase.LoadAssetAtPath(path, typeof(GameObject));
        if(prefab == null)
        {
            Debug.LogWarning(string.Format("Can not load prefab {0}", path));
            continue;
        }

        GameObject go = UnityEngine.Object.Instantiate(prefab) as GameObject;
        Monster comp = go.GetComponent<Monster>();
        if (null == comp)
        {
            Debug.LogWarning(string.Format("{0} 缺少Monster组件", path));
            continue;
        }

        comp.DoSerialized();
        PrefabUtility.ReplacePrefab(go, prefab);
        UnityEngine.Object.DestroyImmediate(go);
    }
}
```

这样一来对资源的修改都会做序列化了。

# 后记
本文所有源代码依然放在[我的Github](https://github.com/AllenKashiwa/StudyUnity)上，有兴趣的朋友可以下来自行参考。