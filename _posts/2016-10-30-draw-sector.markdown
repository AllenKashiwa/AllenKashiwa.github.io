---
layout:     post
title:      "Unity——来个可变角度的扇形技能指示器"
date:       2016-10-30 20:30:00
---

# 技能指示器

如果你的游戏有战斗系统，那极有可能会做成技能系统，如果你的游戏有技能系统，那极有可能需要提示当前技能的攻击范围，以便玩家预判命中或者闪避这次攻击。

具体的形式如这样的：

![image](http://baizihan.me/assets/images/in-post/circle_indicator.png)

这样的：

![image](http://baizihan.me/assets/images/in-post/arrow_indicator.png)

还有这样的：

![image](http://baizihan.me/assets/images/in-post/sector_indicator.png)

**矩形**（箭头也算是矩形的）和**圆形**比较好弄，可以绘制好贴图用一个面片搞定。大小通过缩放面片实现。麻烦的是**扇形**。因为技能肯定不单一，扇形的角度就会有变化。不可能每种角度的扇形都做个特效，这样游戏资源会变多而且策划不能随意调整角度。所以，这个时候程序员们就得解决这个问题，让这样的范围指示器自动生成，避免繁重的美术制作。

网络上常见的方式是生成面片，这种方式的优点是顶点高度可以通过地形改变，在有高度的地形中显示比较自然，缺点是计算复杂，想要产生更平滑的效果需要更多的顶点。网上有不少这种生成面片的文章，大家搜来看看。而我今天要说的这种是通过shader来绘制扇形。

# Shader实现版

先来看看最终的效果：

![image](http://baizihan.me/assets/images/in-post/my_own_indicator.png)

从左到右分别是纯色版，带透明度渐变版和实用贴图版。

下面说说制作步骤：

1. 创建一个Plane
2. 创建一个Material
3. 创建一个Shader，内容如下：

```
Shader "Custom/Indicator" {
    Properties {  
        _MainTex("Main Texture", 2D) = "white" {}
        _Color ("Color", Color) = (0.17,0.36,0.81,0.0)
        _Angle ("Angle", Range(0, 360)) = 60
        _Gradient ("Gradient", Range(0, 1)) = 0
    }

    SubShader {
    Tags { "Queue"="Transparent" "RenderType"="Transparent" "IgnoreProjector"="True" }
         Pass {
            ZWrite Off
            Blend SrcAlpha OneMinusSrcAlpha
            CGPROGRAM
 
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            sampler2D _MainTex;
            float4 _Color;
            float _Angle;
            float _Gradient;
 
            struct fragmentInput {
                float4 pos : SV_POSITION;
                float2 uv : TEXTCOORD0;
            };

            fragmentInput vert (appdata_base v)
            {
                fragmentInput o;

                o.pos = mul (UNITY_MATRIX_MVP, v.vertex);
                o.uv = v.texcoord.xy;

                return o;
            }
 
            fixed4 frag(fragmentInput i) : SV_Target {
            	// 离中心点的距离
                float distance = sqrt(pow(i.uv.x - 0.5, 2) + pow(i.uv.y - 0.5, 2));
                // 在圆外
                if(distance > 0.5f){
                    discard;
                }
                // 根据距离计算透明度渐变
                float grediant = (1 - distance - 0.5 * _Gradient) / 0.5;
                // 正常显示的结果
                fixed4 result = tex2D(_MainTex, i.uv) * _Color * fixed4(1,1,1, grediant);
                float x = i.uv.x;
                float y = i.uv.y;
                float deg2rad = 0.017453;	// 角度转弧度
                // 根据角度剔除掉不需要显示的部分
                // 大于180度
                if(_Angle > 180){
                    if(y > 0.5 && abs(0.5 - y) >= abs(0.5 - x) / tan((180 - _Angle / 2) * deg2rad))
                        discard;// 剔除
                }
                else    // 180度以内
                {
                    if(y > 0.5 || abs(0.5 -y) < abs(0.5 - x) / tan(_Angle / 2 * deg2rad))
                        discard;
                }
                return result;
            }

            ENDCG
        }
    }  
    FallBack "Diffuse"
}
```

4. 将Shader赋予Material，将Material赋予Plane
5. Done

简单吧？

Shader的内容并不复杂，简单来说就是通过UV坐标进行剔除操作。代码注释应该也比较清楚了，大家玩玩看吧。本次依然放出示例工程到[我的 Github](https://github.com/AllenKashiwa/StudyUnity) ，欢迎大家围观。