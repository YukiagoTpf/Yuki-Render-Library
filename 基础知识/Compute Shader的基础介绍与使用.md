## Compute Shader是什么

简单来说，就是一个能够访问GPU实现数据并行算法的可编程着色器，它独立于渲染管线之外，但是可以对GPU资源（存放在显存中）进行读取和写入操作。

> Compute shaders are shader programs that run on the GPU, outside of the normal rendering pipeline.
> They can be used for massively parallel GPGPU algorithms, or to accelerate parts of game rendering. In order to efficiently use them, an in-depth knowledge of GPU architectures and parallel algorithms is often needed; as well as knowledge of [DirectCompute](http://msdn.microsoft.com/en-us/library/windows/desktop/ff476331.aspx), [OpenGL Compute](https://www.khronos.org/opengl/wiki/Compute_Shader), [CUDA](http://en.wikipedia.org/wiki/CUDA), or [OpenCL](http://en.wikipedia.org/wiki/OpenCL).

## 理解Compute Shader

从Unity官方创建的默认ComputeShader了解Cs的基本结构和用法

```hlsl
// Each #kernel tells which function to compile; you can have many kernels  
#pragma kernel CSMain  
  
// Create a RenderTexture with enableRandomWrite flag and set it  
// with cs.SetTexture  
RWTexture2D<float4> Result;  
  
[numthreads(8,8,1)]  
void CSMain (uint3 id : SV_DispatchThreadID)  
{  
    // TODO: insert actual code here!  
  
    Result[id.xy] = float4(id.x & id.y, (id.x & 15)/15.0, (id.y & 15)/15.0, 0.0);  
}
```

### Kernel
第一行：`#pragma kernel CSMain  
CSMain为一个函数，kernel直译为核心，也就是这个Cs的核函数，最终会在GPU中被执行

一个CS中**至少要有一个kernel才能够被唤起**。声明方法即为：``#pragma kernel functionName
同时可以声明多个核函数，可以在该指令后面定义一些预处理的宏命令

### 定义

`RWTexture2D<float4> Result;

Texture2D就是二维纹理,RW代表Read & Write，就是可以读写的意思，如果只想读不想写，直接使用Texture2D类型即可
Result就是声明名，可以使用Restult[uint2(0,0)]来访问，访问值为float4

### numthreads

> 有点难理解，先摘抄，后理解

`[numthreads(8,8,1)] 

定义**一个线程组（Thread Group）中可以被执行的线程（Thread）总数量**

> 在GPU编程中，我们可以将所有要执行的线程划分成一个个线程组，一个线程组在单个流多处理器（Stream Multiprocessor，简称SM）上被执行。如果我们的GPU架构有16个SM，那么至少需要16个线程组来保证所有SM有事可做。为了更好的利用GPU，每个SM至少需要两个线程组，因为SM可以切换到处理不同组中的线程来隐藏线程阻塞（如果着色器需要等待Texture处理的结果才能继续执行下一条指令，就会出现阻塞）。
> 每个线程组都有一个各自的共享内存（Shared Memory），该组中的所有线程都可以访问改组对应的共享内存，但是不能访问别的组对应的共享内存。因此线程同步操作可以在线程组中的线程之间进行，不同的线程组则不能进行同步操作。
> 每个线程组中又是由n个线程组成的，线程组中的线程数量就是通过numthreads来定义的，格式如下：`numthreads(tX, tY, tZ)

**每个核函数前面我们都需要定义numthreads**，否则编译会报错。

其中tX，tY，tZ三个值也并不是也可随便乱填的，它们在不同的版本里有如下的约束：

|Compute Shader 版本|tZ的最大取值|最大线程数量（tX*tY*tZ）|
|---|---|---|
|cs_4_x|1|768|
|cs_5_0|64|1024|

搞清楚结构，我们就可以很好的理解下面这些与单个线程有关的参数含义：

| 参数                  | 值类型  | 含义                                                             | 计算公式                                                                                                 |
| ------------------- | ---- | -------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| SV_GroupID          | int3 | 当前线程所在的线程组的ID，取值范围为(0,0,0)到(gX-1,gY-1,gZ-1)。                   | 无                                                                                                    |
| SV_GroupThreadID    | int3 | 当前线程在所在线程组内的ID，取值范围为(0,0,0)到(tX-1,tY-1,tZ-1)。                  | 无                                                                                                    |
| SV_DispatchThreadID | int3 | 当前线程在所有线程组中的所有线程里的ID，取值范围为(0,0,0)到(gX*tX-1, gY*tY-1, gZ*tZ-1)。 | 假设该线程的SV_GroupID=(a, b, c)，SV_GroupThreadID=(i, j, k) 那么SV_DispatchThreadID=(a*tX+i, b*tY+j, c*tZ+k) |
| SV_GroupIndex       | int  | 当前线程在所在线程组内的下标，取值范围为0到tX*tY*tZ-1。                              | 假设该线程的SV_GroupThreadID=(i, j, k) 那么SV_GroupIndex=k*tX*tY+j*tX+i                                      |

### 核函数

```cpp
void CSMain (uint3 id : SV_DispatchThreadID)
{
    Result[id.xy] = float4(id.x & id.y, (id.x & 15)/15.0, (id.y & 15)/15.0, 0.0);
}
```

核函数也就是我们自己声明的函数

除了SV_DispatchThreadID这个参数以外，我们前面提到的几个参数都可以被传入到核函数当中，根据实际需求做取舍即可，完整如下：

```cpp
void KernelFunction(uint3 groupId : SV_GroupID,
    uint3 groupThreadId : SV_GroupThreadID,
    uint3 dispatchThreadId : SV_DispatchThreadID,
    uint groupIndex : SV_GroupIndex)
{
    
}
```

官方案例中的核函数是为我们Texture中下标为 id.xy 的像素赋值一个颜色
id 的类型为SV_DispatchThreadID，所以id.xy的数值计算如下

>假设该线程的SV_GroupID=(a, b, c)，SV_GroupThreadID=(i, j, k) 那么SV_DispatchThreadID=(a*tX+i, b*tY+j, c*tZ+k)
>SV_GroupID=(8, 8, 1),SV_GroupThreadID的取值范围 为（0，0，0） ~ （7，7，0）
>线程组(a,b,0)中它的取值范围为(a*8, b*8, 0)到(a*8+7, b*8+7, 0)

### C#驱动

以往的Vertex & Fragment shader我们都是给它关联到Material上来使用的，但是CS不一样，它是**由c#来驱动**的。先新建一个monobehaviour脚本，Unity为我们提供了一个**ComputeShader**的类型用来引用我们前面生成的 .compute 文件：

```csharp
public ComputeShader computeShader;
```

此外我们再关联一个Material，因为CS处理后的纹理，依旧要经过FragmentShader采样后来显示。

```csharp
public Material material;
```

接着我们可以将Unity中的**RenderTexture**赋值到CS中的RWTexture2D上，但是需要注意因为我们是多线程处理像素，并且这个处理过程是**无序**的，因此我们要将RenderTexture的**enableRandomWrite**属性设置为true，代码如下：

```csharp
RenderTexture mRenderTexture = new RenderTexture(256, 256, 16);
mRenderTexture.enableRandomWrite = true;
mRenderTexture.Create();
```

我们创建了一个分辨率为256*256的RenderTexture，首先我们要把它赋值给我们的Material，这样我们的Cube就会显示出它。然后要把它赋值给我们CS中的Result变量，代码如下：

```csharp
material.mainTexture = mRenderTexture;
computeShader.SetTexture(kernelIndex, "Result", mRenderTexture);
```

这里有一个kernelIndex变量，即核函数下标，我们可以利用FindKernel来找到我们声明的核函数的下标：

```csharp
int kernelIndex = computeShader.FindKernel("CSMain");
```

这样在我们Fragment Shader采样的时候，采样的就是CS处理过后的纹理：

```csharp
fixed4 frag (v2f i) : SV_Target
{
    // _MainTex 就是被处理后的 RenderTexture
    fixed4 col = tex2D(_MainTex, i.uv);
    return col;
}
```

> 注：此时没有将CS的处理结果从GPU回读到CPU的操作，因为处理结果直接输入到渲染管线中由Fragment Shader进行处理了。

最后就是开线程组和调用我们的核函数了，C#的ComputeShader类提供了Dispatch方法为我们一步到位：

```csharp
computeShader.Dispatch(kernelIndex, 256 / 8, 256 / 8, 1);
```