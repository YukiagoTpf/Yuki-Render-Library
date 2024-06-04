# Tips

记录URP中RenderFeature的结构 & 执行顺序 & 注意事项等
编写文档时使用 **URP 14.0.9**
仅供自己记录

# RenferFeature 结构

## 整体结构

继承自**ScriptableRendererFeature**的**MyCustomRenderFeature**类

* RenderFeature Setting设置
> 设置RenderFeature的一些参数等

* 重写的 **Create()** 方法
> 当RenderFeature参数面板修改时调用，利用类名 + 反射实例化RenderPass，定义了名称，对着色器判定，列化**ScriptableRenderPass**类，将需要的参数传入**ScriptableRenderPass**类中，最后指定需要注入的渲染Pass的位置。

* 重写的 **AddRenderPasses()** 方法
> 每帧调用，将Pass添加到渲染队列里，可以添加多个

继承自 **ScriptableRenderPass** 的 **CustomRenderPass** 类
> 实现具体渲染逻辑的地方

```C#
public calss MyCustomRenderFeature : ScriptableRendererFeature
{
	class Settings
	{...}
	publish
	class CustomRenderPass : ScriptableRenderPass
	{...}
	publish Settings m_Settings = new Settings();
	public override void Create()
	{...}
	
	public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)  
	{...}
}	
```

## ScriptableRenderPass模块

**ScriptableRenderPass** 可将其称为”最基础的渲染单元“ ，因为URP内部渲染的实现，都是基于继承**ScriptableRenderPass**来实现渲染的

官方注释： 实现了一个逻辑渲染通道，可以用来扩展通用RP渲染器

  

从类的执行的层次结构：**RenderPipelines -> ScriptableRenderer -> ScriptableRenderPass**自上而下的执行，最终执行**ScriptableRenderPass**类中的每个所谓的”生命周期函数“

1. 从总入口：RenderPipelines类的 Render函数

2. 在Render函数中RenderSingleCamera中执行 ScriptableRenderer中的”生命周期函数“

3. 在ScriptableRenderer中的Execute函数中执行 ScriptableRenderPass中的”生命周期函数“

经过上面三步，就执行完了一个**ScriptableRenderPass**的生命周期

**MyCustomRenderFeature**负责把这个 Render Pass 加到 Renderer 里面。Render Feature 可以在渲染管线的某个时间点增加一个 Pass 或者多个 Pass。
### 构造函数
> 构造函数，用来初始化RenderPass
### Setup()方法
 > 进行一些初始设置
### OnCameraSetup()方法
> 第一个被执行，可以看到在相机等数据刚准备好的时候调用，其意思很明显就是有一些数据要对RenderingData修改or填充
### Configure()方法
> 在执行渲染过程之前，Renderer 将调用此方法。如果需要配置渲染目标及其清除状态，并创建临时渲染目标纹理，那就要重写这个方法，如果渲染过程未重写这个方法，则该渲染过程将渲染到激活状态下 Camera 的渲染目标。

### Execute()方法
> 执行Pass，这是自定义渲染发生的地方。在这里可以实现渲染逻辑。
**Execute()**是**ScriptableRenderPass**类的核心方法，定义我们的执行规则；包含渲染逻辑，设置渲染状态，绘制渲染器或绘制程序网格，调度计算等等。
判定了材质是否存在，传入摄像机的数据和当前使用的**Renderer**。并且新建一个**CommandBuffer，**在这里执行，执行完毕后释放。

### Dispose()方法
> 主要用法是进行纹理释放
> 1. 在ScriptableRendererPass中定义Dispose方法 并将未释放的RTHandle调用其Release方法
>2. 在ScriptableRendererFeature中override Dispose方法 protected override void Dispose(bool disposing)并调用Pass的Dispose方法

```c#
public void Dispose()
{
	_customOpaqueTexture?.Release();
}

protected override void Dispose(bool disposing)
{
	renderObjectsPass.Dispose();
}
```
# 升级相关

## ScriptableRendererFeature 升级

### 1. SetupPass拆分

旧

```csharp
public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
{
        // The target is used before allocation
        m_CustomPass.Setup(renderer.cameraColorTarget);
         // Letting the renderer know which passes are used before allocation
        renderer.EnqueuePass(m_ScriptablePass);
}
```

新

```csharp
public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
{
        // Letting the renderer know which passes are used before allocation
        renderer.EnqueuePass(m_ScriptablePass);
}

public override void SetupRenderPasses(ScriptableRenderer renderer, in RenderingData renderingData)
{
        // The target is used after allocation
        m_CustomPass.Setup(renderer.cameraColorTarget);
}
```

### 2. 使用Dispose方法进行纹理释放

1. 在ScriptableRendererPass中定义Dispose方法 并将未释放的RTHandle调用其`Release`方法

```csharp
public void Dispose()
{
    _customOpaqueTexture?.Release();
}
```
2. 在ScriptableRendererFeature中override Dispose方法 `protected override void Dispose(bool disposing)`并调用Pass的Dispose方法

```csharp
protected override void Dispose(bool disposing)
{
    renderObjectsPass.Dispose();
}
```

### 3.  一些接口调整

- `public static readonly RenderTargetHandle CameraTarget`不再存在
使用`k_CameraTarget`(在`ScriptableRenderPass`直接使用
或在外部使用`ScriptableRenderPass.k_CameraTarget`)替代
- `scriptableRenderer.cameraColorTarget`现已废弃
使用`scriptableRenderer.cameraColorTargetHandle`替代
- `scriptableRenderer.cameraDepthTarget` 现已废弃
使用 `scriptableRenderer.cameraDepthTargetHandle`代替

## ScriptableRenderPass升级

### 1. RenderTarget & RenderTargetIdentifier 统一

现在两者统一使用新接口 `RTHandle` 

旧

```csharp
private RenderTargetHandle _cameraOpaqueTexture;
```

新

```csharp
private RTHandle _cameraOpaqueTexture;
```

### 2. Identifier - Color & Depth 拆分

旧

```csharp
RenderTargetIdentifier m_Destination;
```

新

```csharp
RTHandle m_DestinationColor;
RTHandle m_DestinationDepth;
```

### 3. 初始化RenderTexture

`renderTargetHandle.Init`+ `commandBuffer.GetTemporaryRT` 
一并替换为 `RenderingUtils.ReAllocateIfNeeded`或 `RTHandles.Alloc` 
如果为`ShadowMap`则使用`ShadowUtils.ShadowRTReAllocateIfNeeded`

在`RTHandles`中不需要设置分辨率，分辨率由`RTHandle System`统一自动控制，在申请时只需要传入`Vector2`格式的降采样参数即可，原始分辨率可传入`Vector2.One`

旧

```csharp
private RenderTargetHandle _opaqueMap;

public override void OnCameraSetup( CommandBuffer cmd, ref RenderingData renderingData )
{
    var cameraTextureDescriptor = renderingData.cameraData.cameraTargetDescriptor;
    opaqueMap.Init( _textureName );
    int width = DownSampleUtil.GetLength( cameraTextureDescriptor.width, downSample );
    int height = DownSampleUtil.GetLength( cameraTextureDescriptor.height, downSample );
    cmd.GetTemporaryRT( _opaqueMap.id, width, height, 0, FilterMode.Bilinear, RenderTextureFormat.ARGBHalf );
    this.ConfigureTarget( _opaqueMap.Identifier() );
    this.ConfigureClear( ClearFlag.All, Color.clear );
}
```

新

```csharp
private RTHandle _opaqueMap;

public override void OnCameraSetup( CommandBuffer cmd, ref RenderingData renderingData )
{
    var cameraTextureDescriptor = renderingData.cameraData.cameraTargetDescriptor;
    cameraTextureDescriptor.colorFormat = RenderTextureFormat.ARGBHalf;
    cameraTextureDescriptor.depthBufferBits = 0;
    RenderingUtils.ReAllocateIfNeeded(
        ref _opaqueMap,
        Vector2.one * 0.5f, 
        cameraTextureDescriptor,
        FilterMode.Bilinear,
        TextureWrapMode.Clamp,
        false,
        1,
        0f,
        "_OpaqueMap"
    );
    this.ConfigureTarget( _opaqueMap );
    this.ConfigureClear( ClearFlag.All, Color.clear );
}
```

> 注：和`renderTargetHandle.Init`相比，RTHandle初始化时会在name(如`"_OpaqueMap"`)后追加描述字符串，所以无法再从全局中取到名为`_OpaqueMap`的纹理，如果需要将RTHandle生成的纹理全局使用（如交给Shader），可用下述写法 `c# Shader.SetGlobalTexture("_OpaqueMap", _opaqueMap);` 另外，如果需要将现有RenderTexture作为RTHandle的纹理，可用下述写法 `c# _opaqueMap = RTHandles.Alloc(Shader.GetGlobalTexture("_OpaqueMap"));`

### 4. 释放RenderTexture

如果该RTHandle由Feature传入，则只需在`OnCameraCleanup`方法中将其置为`null`即可

```csharp
public override void OnCameraCleanup( CommandBuffer cmd )
{
    _transparentObjectsTexture = null;
    _transparentObjectsDepthAttachment = null;
}
```

如果该RTHandle由当前Pass初始化并不会被后续代码引用（如非全局Texture），则可在`OnCameraCleanup`中Release掉

旧

```csharp
public override void OnCameraCleanup( CommandBuffer cmd )
{
    cmd.ReleaseTemporaryRT( _opaqueMap.id );
}
```

新

```csharp
public override void OnCameraCleanup( CommandBuffer cmd )
{
    _opaqueMap?.Release();
}
```

## CommandBuffer.Blit替换

> 官方不再推荐在URP管线中使用 commandBuffer.Blit(Blit) 进行渲染 现在使用Blitter类实现相应功能

**不使用material全屏渲染**
旧

```csharp
cmd.Blit( _transparentObjectsTexture.Identifier(), _cameraOpaqueTexture.Identifier() );
```

新

```csharp
Blitter.BlitCameraTexture2D(cmd,_transparentObjectsTexture, _cameraOpaqueTexture);
```

**使用material全屏渲染**

旧

```csharp
Blit( cmd, tempTarget.Identifier(), target.Identifier(), _blurMaterial )
```

新

```csharp
Blitter.BlitCameraTexture(cmd, tempTarget, target, _blurMaterial, 0);
```

## Blitter使用Material时对应Shader修改

> 如果此前使用CG，需要改为HLSL

使用Blitter后，会将Source的纹理传递给名为`_BlitTexture`的Texture(也可将RTHandle传递为GlobalTexture，进而在Shader中按新名称引用，需注意纹理回收)，且在Shader中对其采样需要使用SAMPLE_TEXTURE2D方法

在Shader中需要引入以下两个HLSL文件(不能再引用UnityCG.cginc 请慎重)

```text
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
#include "Packages/com.unity.render-pipelines.core/Runtime/Utilities/Blit.hlsl"
```

> 注： _BlitTexture已在Blit.hlsl中定义，不需要重复定义

### Vertex 修改

该示例以直接采样输出全屏图为例

```HLSL
struct Attributes
{
#if SHADER_API_GLES
        float4 positionOS : POSITION;
        float2 uv : TEXCOORD0;
#else
        uint vertexID : SV_VertexID;
#endif
        UNITY_VERTEX_INPUT_INSTANCE_ID
};
struct Varyings
{
        float2 texcoord : TEXCOORD0;
        // UNITY_FOG_COORDS(1) //目前没找到可以替代的
        float4 positionCS : SV_POSITION;
        UNITY_VERTEX_OUTPUT_STEREO
};
Varyings vert(Attributes v)
{
        Varyings o;
        UNITY_SETUP_INSTANCE_ID(v);
        UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
#if SHADER_API_GLES
        float4 pos = v.positionOS;
        float2 uv  = v.uv;
#else
        float4 pos = GetFullScreenTriangleVertexPosition(v.vertexID);
        float2 uv = GetFullScreenTriangleTexCoord(v.vertexID);
#endif

        o.positionCS = pos;
        o.texcoord = uv * _BlitScaleBias.xy + _BlitScaleBias.zw;
        return o;
    }
```

### Fragment修改

```HLSL
half4 frag(Varyings i) : SV_Target
{
    half4 col = SAMPLE_TEXTURE2D(_BlitTexture, sampler_LinearRepeat, i.uv);
    return col;
}
```


# 通用Blit模板

## ScriptableRendererFeature部分

**参数定义**
```Csharp
public Shader m_shader;  
[Range(0, 10)] public float number = 0.5f;  
Material m_Material;  
private ScriptableRenderPass scriptableRenderPass;
```
**Create函数**
```Csharp
public override void Create()  
{  
    m_Material = CoreUtils.CreateEngineMaterial(m_shader);  
    m_scriptableRenderPass = new ScriptableRenderPass(m_Material);  
}
```

**AddRenderPasses + SetupRenderPasses**

AddRenderPasses : 使用此方法将Pass加入到渲染队列中
```Csharp
public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)  
{  
    //判断相机状态  
    if (renderingData.cameraData.cameraType == CameraType.Game)  
        renderer.EnqueuePass(m_SNNPass);  
} 
```
SetupRenderPasses : 对Pass的设置在SetupRenderPasses中进行
```Csharp
public override void SetupRenderPasses(ScriptableRenderer renderer, in RenderingData renderingData)  
{  
    if (renderingData.cameraData.cameraType == CameraType.Game)  
    {        
	    //给Pass内插入Color  
        m_Pass.ConfigureInput(ScriptableRenderPassInput.Color);  
        //给Pass内传递参数和RT  
        m_Pass.SetTarget(renderer.cameraColorTargetHandle,m_halfWidth);  
    }}
```
> 此处调用的m_CustomPass.Setup方法为约定的初始化方法，可以自己定义(也可叫别的名字)，一般用于将RenderTexture引用传入ScriptableRendererPass
**Dispose**
```Csharp
public override void Dispose(bool disposing) 
{
	CoreUtils.Destroy(material); 
}
```

## ScriptableRenderPass部分

**定义**

```Csharp
ProfilingSampler m_ProfilingSampler = new ProfilingSampler("SNN");  
Material m_Material;  
RTHandle m_CameraColorTarget;  
public float number = 4;
```

**构造函数**

```Csharp
public RenderPass(Material material)  
{  
    m_Material = material;  
    renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;  
}
```

**SetUp函数**

```Csharp
public void SetTarget(RTHandle colorHandle, float halfWidth)  
{  
    m_CameraColorTarget = colorHandle;  
    m_HalfWidth = halfWidth;  
}
```

**OnCameraSetup**

第一个被执行，可以看到在相机等数据刚准备好的时候调用，其意思很明显就是有一些数据要对RenderingData修改or填充

```Csharp
public override void OnCameraSetup(CommandBuffer cmd, ref RenderingData renderingData)  
{  
    //设置绘制RT  
    ConfigureTarget(m_CameraColorTarget);  
}
```

**Execute**
执行函数内大致流程
1. 获取所需数据、判空、定义cmd
2. 使用ProfilingScope把逻辑框起来以监控性能
3. 最后执行cmd后Clear

```Csharp
public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)  
{  
    var cameraData = renderingData.cameraData;  
    //叛空  
    if (cameraData.camera.cameraType != CameraType.Game)  
        return;  
    //判空  
    if (m_Material == null)  
        return;  
    CommandBuffer cmd = CommandBufferPool.Get();  
    using (new ProfilingScope(cmd , m_ProfilingSampler))  
    {        
    m_Material.SetFloat("number",m_HalfWidth);
    Blitter.BlitCameraTexture(cmd,m_CameraColorTarget,m_CameraColorTarget,m_Material,0);  
    }    
    context.ExecuteCommandBuffer(cmd);  
    cmd.Clear();  
}
```

