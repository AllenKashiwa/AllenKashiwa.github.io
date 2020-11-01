---
layout:     post
title:      "如何在Unity中创建可破碎的墙体"
date:       2020-11-02 01:07:28
---

# **效果展示**

如果你想要在自己的游戏中添加类似下图的效果，那这篇文章可能会帮到你：

![破碎效果](http://baizihan.me/assets/images/in-post/destructible_walls.gif)

# 如何做到演示效果

要实现这样的效果需要如下几个步骤：

## 1.切Mesh

墙体是一个大的整体的mesh，我们首先要将其切成大小不同的块（chunk）。这里使用了[Nvidia blast library](https://developer.nvidia.com/blast)来完成切mesh。使用非常简单，只需要传入mesh就可以得到返回的chunks。

核心代码如下：

```
public void Bake(GameObject go)
{
    NvBlastExtUnity.setSeed(seed);

    var nvMesh = new NvMesh(
        mesh.vertices,
        mesh.normals,
        mesh.uv,
        mesh.vertexCount,
        mesh.GetIndices(0),
        (int) mesh.GetIndexCount(0)
    );

    var fractureTool = new NvFractureTool();
    fractureTool.setRemoveIslands(false);
    fractureTool.setSourceMesh(nvMesh);

    Voronoi(fractureTool, nvMesh);

    fractureTool.finalizeFracturing();

    for (var i = 1; i < fractureTool.getChunkCount(); i++)
    {
        var chunk = new GameObject("Chunk" + i);
        chunk.transform.SetParent(go.transform, false);

        Setup(i, chunk, fractureTool);
        FractureUtils.ConnectTouchingChunks(chunk, jointBreakForce);
    }
}
```

## 2.添加刚体

为每个chunk添加RigidBody。此时各个刚体因为没有连接，会在重力的作用下坍塌。

![Rigidbody](http://baizihan.me/assets/images/in-post/destructible_walls/rigidbody.png)

![crumbles](http://baizihan.me/assets/images/in-post/destructible_walls/crumbles.gif)

## 3.添加固定关节

找到每个chunk相邻的所有chunk，并添加fixed joints将他们连接起来，使chunk保持在自己的位置上。

![Neighbours](http://baizihan.me/assets/images/in-post/destructible_walls/neighbours.png)

![Fixed Joints](http://baizihan.me/assets/images/in-post/destructible_walls/fixed_joints.png)

此时chunk虽然由于关节保持在位置上但是墙体会像果冻一样不停颤抖。

## 4.按需锁定和解锁刚体

为了让墙不抖动，可以设置chunk上的rigidbody的约束。

我们设计如下两个接口来动态设置chunk上刚体的属性：

```
public void Unfreeze()
{
    frozen = false;
    rb.constraints = RigidbodyConstraints.None;
    rb.useGravity = true;
    rb.gameObject.layer = LayerMask.NameToLayer("Default");
}

private void Freeze()
{
    frozen = true;
    rb.constraints = RigidbodyConstraints.FreezeAll;
    rb.useGravity = false;
    rb.gameObject.layer = LayerMask.NameToLayer("FrozenChunks");
    frozenPos = rb.transform.position;
    forzenRot = rb.transform.rotation;
}
```

默认添加rigidbody后，我们就调用Freeze，这样墙就不会想果冻一样摇晃。我们创建一个chunk连接块的图。如果一个chunk与rigidbody没有关联，我们就调用Unfreeze。

要判断一个chunk和所有rigidbody没有关联，我们获取所有块，并在邻居之间创建双向连接。递归遍历所有刚体的邻居，未遍历的块就是未与任何刚体连接。此时这些chunk应该处于自由下落状态，则调用Unfreeze。

# **资源**

[原项目地址](https://github.com/ElasticSea/destructible-walls)

如果你喜欢这个系列可以扫描下面的二维码关注我的公众号：

![Unity与图形学](http://baizihan.me/assets/images/qrcode.jpg)
