---
layout:     post
title:      "眼见为实——如何在Unity中画出碰撞范围"
date:       2017-05-07 18:30:00
---

# 碰撞器

游戏中经常需要做范围判定，常见的方式是进行物理上的碰撞检测。在Unity引擎中提供了各种类型的Collider和碰撞检测接口。其中，Collider可以在Scene窗口中看到绿色线标识的范围。例如，CapsuleCollider：

![CapsuleCollider](http://baizihan.me/assets/images/in-post/gizmos_helper/capsule_collider.png)

这样的范围标识能带来直观的感受。但是在开发过程中，范围判定不一定全是通过Collider来做检测，并且挂载Collider的物体极有可能在我们需要看到显示范围时已经被销毁。所以，我们有必要做一个能画出各种几何图形的工具。Unity中提供的GizmosDraw的功能可以实现这样一个工具。

# 效果

我们的工具要能做到以下效果：

![GizmosResult](http://baizihan.me/assets/images/in-post/gizmos_helper/gizmos_result.png)

从图中可以看出，GameObjective和GizmosDraw出的范围有出入，这是模拟Collider会随着设置的Scale，Radius，Center等值而产生偏差。而我们的工具需要保证在参数设置相同的情况下，Gizmos的结果和碰撞器的真实范围一致。

![矩形](http://baizihan.me/assets/images/in-post/gizmos_helper/cube_inspector.png)

![球形](http://baizihan.me/assets/images/in-post/gizmos_helper/sphere_inspector.png)

![胶囊体](http://baizihan.me/assets/images/in-post/gizmos_helper/capsule_inspector.png)

# 代码

由于篇幅原因，这里仅给出DrawCircleImp和DrawCapsuleImp的实现代码。完整代码欢迎大家前往我的[github](https://github.com/AllenKashiwa/StudyUnity)围观。有任何问题也可以和我交流。另外，从本项目起，Untiy版本更新至5.6.0f3。

```csharp
private static void DrawCircleImp(Vector3 center, Vector3 up, Color color, float radius)
{
    var oldColor = Gizmos.color;
    Gizmos.color = color;

    up = (up == Vector3.zero ? Vector3.up : up).normalized * radius;
    var forward = Vector3.Slerp(up, -up, 0.5f);
    var right = Vector3.Cross(up, forward).normalized * radius;
    for (var i = 1; i < 26; i++)
    {
        Gizmos.DrawLine(center + Vector3.Slerp(forward, right, (i - 1) / 25f), center + Vector3.Slerp(forward, right, i / 25f));
        Gizmos.DrawLine(center + Vector3.Slerp(forward, -right, (i - 1) / 25f), center + Vector3.Slerp(forward, -right, i / 25f));
        Gizmos.DrawLine(center + Vector3.Slerp(right, -forward, (i - 1) / 25f), center + Vector3.Slerp(right, -forward, i / 25f));
        Gizmos.DrawLine(center + Vector3.Slerp(-right, -forward, (i - 1) / 25f), center + Vector3.Slerp(-right, -forward, i / 25f));
    }

    Gizmos.color = oldColor;
}
        
private void DrawCapsuleImp(Vector3 pos, Vector3 center, Vector3 scale, CapsuleDirection direction, float radius, float height, Color color)
{
    // 参数保护
    if (height < 0f)
    {
        Debug.LogWarning("Capsule height can not be negative!");
        return;
    }
    if (radius < 0f)
    {
        Debug.LogWarning("Capsule radius can not be negative!");
        return;
    }
    // 根据朝向找到up 和 高度缩放值
    Vector3 up = Vector3.up;
    // 半径缩放值
    float radiusScale = 1f;
    // 高度缩放值
    float heightScale = 1f;
    switch (direction)
    {
        case CapsuleDirection.XAxis:
            up = Vector3.right;
            heightScale = Mathf.Abs(scale.x);
            radiusScale = Mathf.Max(Mathf.Abs(scale.y), Mathf.Abs(scale.z));
            break;
        case CapsuleDirection.YAxis:
            up = Vector3.up;
            heightScale = Mathf.Abs(scale.y);
            radiusScale = Mathf.Max(Mathf.Abs(scale.x), Mathf.Abs(scale.z));
            break;
        case CapsuleDirection.ZAxis:
            up = Vector3.forward;
            heightScale = Mathf.Abs(scale.z);
            radiusScale = Mathf.Max(Mathf.Abs(scale.x), Mathf.Abs(scale.y));
            break;
    }

    float realRadius = radiusScale * radius;
    height = height * heightScale;
    float sideHeight = Mathf.Max(height - 2 * realRadius, 0f);

    center = new Vector3(center.x * scale.x, center.y * scale.y, center.z * scale.z);
    // 为了符合Unity的CapsuleCollider的绘制样式，调整位置
    pos = pos - up.normalized * (sideHeight * 0.5f + realRadius) + center;

    Color oldColor = Gizmos.color;
    Gizmos.color = color;

    up = up.normalized * realRadius;
    Vector3 forward = Vector3.Slerp(up, -up, 0.5f);
    Vector3 right = Vector3.Cross(up, forward).normalized * realRadius;

    Vector3 start = pos + up;
    Vector3 end = pos + up.normalized * (sideHeight + realRadius);

    // 半径圆
    DrawCircleImp(start, up, color, realRadius);
    DrawCircleImp(end, up, color, realRadius);

    // 边线
    Gizmos.DrawLine(start - forward, end - forward);
    Gizmos.DrawLine(start + right, end + right);
    Gizmos.DrawLine(start - right, end - right);
    Gizmos.DrawLine(start + forward, end + forward);
    Gizmos.DrawLine(start - forward, end - forward);

    for (int i = 1; i < 26; i++)
    {
        // 下部的头
        Gizmos.DrawLine(start + Vector3.Slerp(right, -up, (i - 1) / 25f), start + Vector3.Slerp(right, -up, i / 25f));
        Gizmos.DrawLine(start + Vector3.Slerp(-right, -up, (i - 1) / 25f), start + Vector3.Slerp(-right, -up, i / 25f));
        Gizmos.DrawLine(start + Vector3.Slerp(forward, -up, (i - 1) / 25f), start + Vector3.Slerp(forward, -up, i / 25f));
        Gizmos.DrawLine(start + Vector3.Slerp(-forward, -up, (i - 1) / 25f), start + Vector3.Slerp(-forward, -up, i / 25f));

        // 上部的头
        Gizmos.DrawLine(end + Vector3.Slerp(forward, up, (i - 1) / 25f), end + Vector3.Slerp(forward, up, i / 25f));
        Gizmos.DrawLine(end + Vector3.Slerp(-forward, up, (i - 1) / 25f), end + Vector3.Slerp(-forward, up, i / 25f));
        Gizmos.DrawLine(end + Vector3.Slerp(right, up, (i - 1) / 25f), end + Vector3.Slerp(right, up, i / 25f));
        Gizmos.DrawLine(end + Vector3.Slerp(-right, up, (i - 1) / 25f), end + Vector3.Slerp(-right, up, i / 25f));
    }

    Gizmos.color = oldColor;
}

```