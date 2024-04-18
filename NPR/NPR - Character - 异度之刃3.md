## Reference

[异度之刃3 CEDEC渲染分享](https://zhuanlan.zhihu.com/p/588630949)

## 效果图

| Directional Light | Point Light Right | Point Light Left
| :--: | :--: | :--: | 
| \#FFF0F0 Intensity:2 | \#95BFE7 Intensity :3 | \#E6AA9D Intensity:3.8|
| Rotation:17,-66,-18 | -- | -- |

![Alt Text](Textures/20230128212309.png)
![Alt Text](Textures/20230128212314.png)
![Alt Text](Textures/20230128212319.png)

## 制作过程

总体分三个部分，前期的资产处理，以及主要的光照处理，和一些特殊效果的Feature

### 资产处理
#### 贴图
这一部分其实也没有进行特别多的处理，从Renderdoc上弄下来的模型UV Y轴被反转了，这部分问题我直接在Shader里解决了

![Alt Text](Textures/20230128212752.png)
（因为目前只制作Mio这个角色的渲染，如果套用到其他角色还是得反转下贴图or处理下UV的）

#### 平滑法线

比较常规的一个步骤，目的是为了处理断边，一般的处理方法是把平滑后的法线数据存放在切线中，用平滑后的法线进行外扩，但是Mio她有法线贴图，需要原始的TBN数据

所以我把平滑后的法线转到切线空间存到了UV3中（其实都可以，只要不用的UV Channel都可以存）

不转切线空间的话骨骼动画会出错

![Alt Text](Textures/20230128212838.png)

拿一个插件的平滑法线工具改的，原版并不支持蒙皮Mesh把平滑法线存到UV中
![Alt Text](Textures/20230128214108.png)

### 光照

#### 基于Ramp的PBR Toon

这一部分基本就是基于URP Lit Shader魔改的PBR效果，在URP自带的LightingPhysicallyBased计算平行光光照函数中，Diffuse是通过普通的NdotL乘以灯光的颜色和衰弱，最后与计算出来的Specular来相乘来获取平行光光照后的manLightColor，那我们直接介入这个环节，将NdotL的结果来采样RampTex，再用这个来去乘以灯光的颜色和衰弱，也就是**DirectionRadiance**

```hlsl
half halfLambert = dot(normalWS, lightDirectionWS)*0.5+0.5;
float3 RampDiffuse = tex2D(DiffuseRamp,float2(saturate(halfLambert - ShadowRange), 0.5));
```

这时候就会遇到一个问题，Ramp图是偏亮的，计算出来的DirectionRadiance相比于其他的PBR物体会亮不少，就会过曝

我这里使用的是和XB3一样的方案，将光照结果转换成亮度后，通过参数来控制

```hlsl
half ColorToLuminance(float3 diffuse)
{
    return 0.2126 * diffuse.r + 0.7152 * diffuse.g + 0.0722 * diffuse.b;
}
float DirectionLuminance  = ColorToLuminance(DirectionRadiance);
DirectionRadiance *= min(ToonCtrl.Exposure,DirectionLuminance)/(DirectionLuminance + ToonCtrl.Eplision);
```

![Alt Text](Textures/20230128214244.png)


之后的GI、直接光的高光等就按照正常的PBR算法进行计算即可，因为本身Mio的AO就不是非常的重，所以结果也挺干净的

效果如下图

![Alt Text](Textures/20230128214301.png)
![Alt Text](Textures/20230128214308.png)

![Alt Text](Textures/20230128214314.png)
几张自己画的Ramp图，没有画的很细，也没考虑昼夜交换的情况，XB3中的Ramp非常非常非常复杂

#### 边缘光

![Alt Text](Textures/20230128214351.png)

最基本的菲涅尔边缘光，分为两个部分，亮部和暗部，亮部的颜色由环境光来提供，暗部的颜色由所有照到角色的灯光中亮度最亮（其实可以和主光的颜色进行一个混合）的那一个来提供
![Alt Text](Textures/20230128214410.png)
![Alt Text](Textures/20230128214419.png)


#### 两灯混合（平行光+1个点光）

我们假设环境里有两个灯光，1个平行光+1个点光，许多ToonShading把点光源都作为染色作用，XB3里不一样

首先先计算平行光下的Diffuse
![Alt Text](Textures/20230128214503.png)


忽视掉点光源的衰弱，计算点光源的Diffuse
![Alt Text](Textures/20230128214514.png)

再通过之前调节曝光使用的ColorToLuminance函数把点光源照射下在像素上的亮度计算出来，以此为权重，进行平行光源和点光源之间的混合

![Alt Text](Textures/20230128214523.png)

**Lerp（平行光结果，点光结果，点光结果的亮度）**

![Alt Text](Textures/20230128214537.png)

有一个比较特殊的地方，XB3并没有直接拿点光源的数据进行光照计算，而是通过一套混合公式（会考虑到一点平行光的因素）来计算出点光的方向、颜色以及亮度，我暂时没细究其中的影响，直接用就是了

```hlsl
half AdditionalLightingMix(Light Direction,Light Additional,out Light AdditionalLight)
{
    float Luminance_Direction = ColorToLuminance(Direction.color);
    float Luminance_Additional = ColorToLuminance(Additional.color);
    AdditionalLight.color =
        (Additional.distanceAttenuation * Additional.color * Luminance_Additional + 0.2 * Direction.color * Luminance_Direction)
        /
        (Additional.distanceAttenuation * Luminance_Additional + 0.2 * Luminance_Direction);
    float3 AdditionalLightDir =
        (Additional.distanceAttenuation * Luminance_Additional * Additional.direction + 0.2 * Luminance_Direction * Direction.direction)
        /
        (Additional.distanceAttenuation * Luminance_Additional + 0.2 * Luminance_Direction);
    AdditionalLight.direction = normalize(AdditionalLightDir);
    AdditionalLight.distanceAttenuation = 1 ;
    AdditionalLight.shadowAttenuation = 1 ;
    AdditionalLight.layerMask = DEFAULT_LIGHT_LAYERS;
    return 0;
}
```

0.2为与主光源的混合权重，可自定义

![Alt Text](Textures/20230128214607.png)

![Alt Text](Textures/20230128214614.png)

![Alt Text](Textures/5da4eb4b-d565-4324-8fa3-7b803748adb4.gif)

![Alt Text](Textures/8d75b49c-aa1d-485f-a8c8-47ce9ff36525.gif)

#### 多光混合（平行光+N个点光）

分享PPT中并没有提到超过2灯情况后该怎么处理，可能是为了最后的光照结果不是那么的脏，毕竟是卡通渲染

我这里还是按照两灯的混合方式来进行多光的混合

**Lerp（上个混合的结果，第N个点光的光照结果，光照结果的亮度）**

![Alt Text](Textures/20230128214730.png)

![Alt Text](Textures/20230128214734.png)

#### 头发高光

这个XB3的做法是把头发的法线改成了球法线，然后使用Matcap的方法去制作

我一开始是通过比较常规的各向异性，但经过SNN处理之后显得太碎了一点......用了官方做法之后就好很多了...

但这一部分我暂时还没放到光照计算里，所以不会被光照影响，算是之后的TODO项

![Alt Text](Textures/d2d483ee-24ed-450e-a8c7-bdfef3869c4d.gif)

眼睛暂时没做任何处理，直接就是Albedo贴图贴上去

### Feature

#### LayerMask

这个Feature就和名字一样，主要是根据Layer来绘制Mask，用于后面的一些效果制作，我设定了Hair和Face的Layer（其实也可以用于后面的皮肤额外Bloom，但我这里并没有这么做）

![Alt Text](Textures/20230128215634.png)

![Alt Text](Textures/20230128215641.png)

#### SNN

这一部分我就不过多描述了，之前的文章写的相对比较清楚，就是在之前的基础上，SNN的shader里采样了LayerMask这张图来做到只对头发进行处理

SNN具体说明文档
[[Unity Engine/Non-Photorealistic Rendering/PostProcessing/PostProcessing - SNN Filter]]

效果

![Alt Text](Textures/20230128221154.png)![Alt Text](Textures/20230128221200.png)

#### HairShadow

我这里用的是RenderFeature的方法

同样是在脸部的Shader中，去采样LayerMask这张图，并根据View空间下的平行光方向进行采样，由PosNDC.w来控制偏移距离

![Alt Text](Textures/20230128221600.png)

<font color = yellow>此处要补一个HairShadow的具体文档</font>

#### EyeBrowOut

非常标准的做法，眉毛两个Pass

ZTest LEqual正常绘制，ZTest GEqual负责渲染被头发遮挡住的头发，RenderObject来启用

![Alt Text](Textures/20230128221737.png)

#### SkinBloom

在XB3中，MonoLith把人物的皮肤Part额外加了一个Bloom

![Alt Text](Textures/20230128221753.png)

这个一加，Mio就显得非常水灵了（像是在发光）

![Alt Text](Textures/20230128221806.png)

<font color = yellow>此处要补一个SelectiveBloom的具体文档</font>

### 点睛之笔

鼻尖和嘴唇的这个高光我感觉对整个人物的渲染有比较大的影响，加了之后显得更加可爱了，也没有太大的开销

![Alt Text](Textures/20230128221818.png)





