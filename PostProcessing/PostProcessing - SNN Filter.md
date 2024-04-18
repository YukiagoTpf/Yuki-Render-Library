## Reference

![[Reference/SNN Filter Paper.pdf]]

[SNN/Kuwahara Filter](https://www.shadertoy.com/view/MlyfWd)

## 研究背景

在游玩一些日本游戏的时候，发现日本厂商在对风格化的探索中，开始使用一种特殊的后处理滤镜，比较典型的就是破晓传说

![Alt Text](Textures/20230128220415.png)

![Alt Text](Textures/20230128220419.png)

破晓传说在整个画面中都大量使用了这些滤镜效果（人物+场景），得到了一个风格特别水彩的画面，他们自己称之为“Atmos Shader”

（但个人认为这种大范围的使用会让画面有点糊在一起的感觉）

最近游玩的异度之刃3对于这种滤镜的使用让我眼前一亮，它并没有大批量使用这种滤镜，而是针对人物渲染中的头发，单独使用了这个滤镜，得到了一个我觉得非常有意思的效果

![Alt Text](Textures/20230128220431.png)

![Alt Text](Textures/20230128220435.png)

这个滤镜让头发的明暗交界线不是那么的明显，增强了笔触感，也让高光有了一种独特的质感。

## Filter算法

![Alt Text](Textures/20230128220459.png)

### Kuwahara Filter

和卷积一样，需要一个核，不过需要用4个核
![Alt Text](Textures/43406e2d-3689-48d4-8312-08ebd82945b4.gif)
**算法**
1.  计算一个像素四周每四个核中颜色的平均值
2.  对于每个核，计算方差，这主要是测量每个核中颜色值的变化大小
3.  我们找到方差最低的核，让这个像素的值等于这个卷积核的平均值
![Alt Text](Textures/6cfb9b71-53a1-499b-9f74-9ade1408733e.gif)
Kuwahara算法最初用在了图像的去噪上，比起模糊算法更能保持边缘的锐利性
保持边缘的平滑特性还给图像带来了一种手绘风。
因为，手绘的画风通常就是边缘较硬而且不会产生很大（画面）噪声的，并且单个物体的颜色细节并不会非常丰富，所以Kuwahara通常还使用在了风格化中

![Alt Text](Textures/20230128220632.png)

### SNN Filter

比起Kuwahara滤镜，SNN从更平滑和更多边缘保留中保持了一个平衡，没有Kuwahara滤镜那么锐利的边缘保持，会更像是水彩的感觉，而且更会保持目标像素本身的颜色

算法比起Kuwahara滤镜也相当不一样

![Alt Text](Textures/20230128220655.png)

**算法**
拿一个3X3的核为例，SNN就是把目标像素四周的对角像素进行对比，选择颜色数值偏差小，比如N、S像素对比，N比起S更接近目标像素的颜色值，那就选择N，然后依次对比完剩下的像素，最后将四个选择后的数值加起来计算平均值，输出即可
![Alt Text](Textures/20230128220719.png)
![Alt Text](Textures/20230128220723.png)
![Alt Text](Textures/20230128220728.png)
![Alt Text](Textures/20230128220731.png)

## 落地使用

我这里的使用场景是还原异度之刃3中对于人物头发特殊效果渲染的还原

最终效果大致这样

![Alt Text](Textures/20230128220807.png)

除了一些细节上的问题，大致上是还原了游戏中类厚涂的头发效果

我实现的较为简单，做了一个全屏的后处理，游戏中是将头发绘制完成之后扣出来单独做了一个后处理

创建一个RenderFeature

![Alt Text](Textures/20230128220819.png)

创建RT输入Shader

Shader中进行图像的处理

```hlsl
float3 SNN( float2 ScennUV,TEXTURE2D(mainTexture),SAMPLER(mainTextureSamper) )
{
    
 float2 src_size = _ScreenParams.xy;
    float2 inv_src_size = 1.0f / src_size;
    float2 uv = ScennUV;
    
    float3 c0 = SAMPLE_TEXTURE2D(mainTexture,mainTextureSamper,uv);
    
    float4 sum = float4(0.0f, 0.0f, 0.0f, 0.0f);
    
    for (int i = 0; i <= _half_width; ++i) {
        float3 c1 = SAMPLE_TEXTURE2D(mainTexture,mainTextureSamper, uv + float2(+i, 0) * inv_src_size).rgb;
        float3 c2 = SAMPLE_TEXTURE2D(mainTexture,mainTextureSamper, uv + float2(-i, 0) * inv_src_size).rgb;
        
        float d1 = CalcDistance(c1, c0);
        float d2 = CalcDistance(c2, c0);
        if (d1 < d2) {
            sum.rgb += c1;
        } else {
            sum.rgb += c2;
        }
        sum.a += 1.0f;
    }
  for (int j = 1; j <= _half_width; ++j) {
     for (int i = -_half_width; i <= _half_width; ++i) {
            float3 c1 = SAMPLE_TEXTURE2D(mainTexture,mainTextureSamper, uv + float2(+i, +j) * inv_src_size).rgb;
            float3 c2 = SAMPLE_TEXTURE2D(mainTexture,mainTextureSamper, uv + float2(-i, -j) * inv_src_size).rgb;
            
            float d1 = CalcDistance(c1, c0);
            float d2 = CalcDistance(c2, c0);
            if (d1 < d2) {
               sum.rgb += c1;
            } else {
                sum.rgb += c2;
            }
            sum.a += 1.0f;
  }
    }
    return sum.rgb / sum.a;
}
```

最后输出

后续其实可以学习破晓传说中，对SNN的核进行长宽的控制，在加上一些锐化的处理，效果应该会更接近破晓传说中的风格化处理。