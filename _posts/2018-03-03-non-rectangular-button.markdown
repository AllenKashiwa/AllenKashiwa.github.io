---
layout:     post
title:      "使用Mono.Cecil实现IL代码注入"
date:       2018-03-03 23:15:00
---

# 前言

UGUI中的按钮默认是矩形的，若要实现非矩形按钮该怎么做呢？比如这样的按钮：

![image](http://baizihan.me/assets/images/in-post/non_rectangular_button/non_rect_button.png)

本文将介绍两种实现方式供大家选择。

# 使用alphaHitTestMinimumThreshold

Image类的alphaHitTestMinimumThreshold是一个浮点值，Raycast检测时只有图片中高于该值的部分会抛出点击事件。因此我们可以使用一张alpha通道的值高于该设置值的Sprite用于自定义按钮的点击相应区域。

我们准备一张点击区域alpha高于某值，非点击区域alpha低于某值的Sprite用于Button的Image组件的Sprite。然后给这个Button挂上如下脚本组件即可：

```
using UnityEngine;
using UnityEngine.UI;

public class AlphaButton : MonoBehaviour
{
    public float alphaThreshold = 0.1f;

    void Start()
    {
        GetComponent<Image>().alphaHitTestMinimumThreshold = alphaThreshold;
    }
}

```

但这种方法有几个问题：

1. 由于是代码中需要读取图片的alpha值用于比较，因此图片在导入时需要开启Readable/Write Enable，这样会使运行时贴图大小翻倍，内存中会额外存储一份贴图数据，增大内存开销。

2. 如果是点击区域内部需要有一些低于设置值的透明样式则无法满足。

3. 点击区域的调整需要修改图片资源，十分不便。

如果可以接受这些缺点，可以使用这个方法。

# 使用IsRaycastLocationValid

通过继承Image并重写IsRaycastLocationValid方法可以自定义按钮的可点击区域。

将如下代码放置于项目中：

```
using UnityEngine;
using UnityEngine.UI;
#if UNITY_EDITOR
using UnityEditor;

#endif
[RequireComponent(typeof(PolygonCollider2D))]
public class NonRectangularButtonImage : Image
{
    private PolygonCollider2D areaPolygon;

    protected NonRectangularButtonImage()
    {
        useLegacyMeshGeneration = true;
    }

    private PolygonCollider2D Polygon
    {
        get
        {
            if (areaPolygon != null)
                return areaPolygon;

            areaPolygon = GetComponent<PolygonCollider2D>();
            return areaPolygon;
        }
    }

    protected override void OnPopulateMesh(VertexHelper vh)
    {
        vh.Clear();
    }

    public override bool IsRaycastLocationValid(Vector2 screenPoint, Camera eventCamera)
    {
        return Polygon.OverlapPoint(eventCamera.ScreenToWorldPoint(screenPoint));
    }

#if UNITY_EDITOR
    protected override void Reset()
    {
        base.Reset();
        transform.localPosition = Vector3.zero;
        var w = rectTransform.sizeDelta.x * 0.5f + 0.1f;
        var h = rectTransform.sizeDelta.y * 0.5f + 0.1f;
        Polygon.points = new[]
        {
            new Vector2(-w, -h),
            new Vector2(w, -h),
            new Vector2(w, h),
            new Vector2(-w, h)
        };
    }
#endif
}
#if UNITY_EDITOR
[CustomEditor(typeof(NonRectangularButtonImage), true)]
public class CustomRaycastFilterInspector : Editor
{
    public override void OnInspectorGUI()
    {
    }
}

public class NonRectAngularButtonImageHelper
{
    [MenuItem("GameObject/UI/NonRectangularButtonImage")]
    public static void CreateNonRectAngularButtonImage()
    {
        var goRoot = Selection.activeGameObject;
        if (goRoot == null)
            return;

        var button = goRoot.GetComponent<Button>();

        if (button == null)
        {
            Debug.Log("Selecting Object is not a button!");
            return;
        }

        // 关闭原来button的射线检测
        var graphics = goRoot.GetComponentsInChildren<Graphic>();
        foreach (var graphic in graphics)
        {
            graphic.raycastTarget = false;
        }

        var polygon = new GameObject("NonRectangularButtonImage");
        polygon.AddComponent<PolygonCollider2D>();
        polygon.AddComponent<NonRectangularButtonImage>();
        polygon.transform.SetParent(goRoot.transform, false);
        polygon.transform.SetAsLastSibling();
    }
}

#endif
```

这段代码大部分参考自雨松大神的这篇文章：

[UGUI研究院之不规则按钮的响应区域（十四）](http://www.xuanyusong.com/archives/3492)

还额外写了一个自动添加组件和设置raycastTarget属性的菜单项。创建完一个普通的按钮后，右键执行命令：

![image](http://baizihan.me/assets/images/in-post/non_rectangular_button/cmd.png)

这将自动创建一个名为“NonRectangularButtonImage”的子节点，并添加一个同名的脚本组件和一个PolygonCollider2D组件。编辑PolygonCollider2D组件即可设置按钮的点击区域，调整起来也十分方便，既简单又节省内存。

[我的Github](https://github.com/AllenKashiwa/StudyUnity/tree/master/NonRectangularButton)中这两种方式都有实现，供大家参考：

![image](http://baizihan.me/assets/images/in-post/non_rectangular_button/project_result.png)

共三组按钮，点击后可以在Console窗口中看到响应Log。

第一组是没有任何处理的普通按钮，由于在Hierarchy中RightButton在下，点击Left的右下角还是右边按钮响应，用于对照。

第二组使用了设置alphaHitTestMinimumThreshold的方式。

第三组使用了重写IsRaycastLocationValid的方式，并故意调整了Button在Hierarchy中的顺序。

如果可以，也希望大家点个Star。

# 参考

[使用alphaHitTestMinimumThreshold的方式](https://answers.unity.com/questions/821613/unity-46-is-it-possible-for-ui-buttons-to-be-non-r.html)

[UGUI研究院之不规则按钮的响应区域（十四）](http://www.xuanyusong.com/archives/3492)

[使用mask的方式](https://forum.unity.com/threads/none-rectangle-shaped-button.263684/)

[Image.alphaHitTestMinimumThreshold](https://docs.unity3d.com/ScriptReference/UI.Image-alphaHitTestMinimumThreshold.html)

[ICanvasRaycastFilter.IsRaycastLocationValid](https://docs.unity3d.com/ScriptReference/ICanvasRaycastFilter.IsRaycastLocationValid.html)