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

### UAV（Unordered Access view）

通常我们Shader中使用的资源被称作为**SRV（Shader resource view）**，例如Texure2D，它是**只读**的。但是在Compute Shader中，我们往往需要对Texture进行写入的操作，因此SRV不能满足我们的需求，而应该使用一种新的类型来绑定我们的资源，即UAV。它允许来自多个线程临时的无序读/写操作，这意味着该资源类型可以由多个线程同时读/写，而不会产生内存冲突。

前面我们提到了RWTexture，RWStructuredBuffer这些类型都属于UAV的数据类型，并且它们**支持在读取的同时写入**。它们只能在Fragment Shader和Compute Shader中被使用（绑定）。

如果我们的RenderTexture不设置enableRandomWrite，或者我们传递一个Texture给RWTexture，那么运行时就会报错：

> the texture wasn't created with the UAV usage flag set!

## 理解环节

通过学习以下仓库中Demo理解Cs各个模块
[MinimalCompute](https://github.com/cinight/MinimalCompute).

### Compute Texture

这一章主要是学习ComputeTexture的各类用法

#### FallingSand

实现一个随意绘制贴图，并能够阻挡向下掉落的沙子

![Alt Text](Textures/01_2_FallingSand.gif)

##### 代码分析

先来看Cs代码

首先是只有一个核函数

定义了
贴图（输出用）
Size：Size为C#传入Texture2D的Size
Time：Time.time
MousePos：由C#传入的鼠标位置，与贴图对应
MouseMode：用来区分不同模式，由C#传入，0绘制像素，1绘制障碍，2移除障碍
```CPP
RWTexture2D<float4> Result;  
int _Size;  
float _Time;  
float2 _MousePos;  
int _MouseMode;
```

```
[numthreads(8,8,1)]
```

核函数中主要分为两块逻辑：制作Pixel or 障碍 / Pixel的自由落体

**制作Pixel or 障碍**
如何获取鼠标所在Piexl？贴图的Size  x 鼠标所在的UV位置
```CPP

void MakeNewPixelObstacle(int2 id)  
{  
    float4 p = Result[id];  
    if( _MouseMode == 0 && p.x == 0 && p.z == 0  ) Result[id] = float4(1,1,0,1);  
    else if( _MouseMode == 1 ) Result[id] = float4(0,1,1,1);  
    else if( _MouseMode == 2 && p.x == 0 ) Result[id] = float4(0,0,0,1);  
}

//make new pixel / obstacle ----------------------  
int2 newPixelID = _MousePos*_Size;  
if( _MouseMode != 0)  
{  
    //so that the brush is thicker  
    float dist = distance(float2(id.xy),float2(newPixelID));  
    if(dist < 2.5f) MakeNewPixelObstacle(id.xy);  
}  
else  
{  
    MakeNewPixelObstacle(newPixelID);  
}
```
大致逻辑如下：
首先是绘制像素的函数
如果是绘制像素模式且目标像素的R值为0，A值为0才会在目标像素绘制黄色像素
如果是绘制障碍模式，绘制青色像素，如果是移除障碍模式且R值为0（也是障碍物的颜色），会被重置为黑色
实际执行中，通过计算目标像素和鼠标所在像素的Distance来控制笔刷大小，绘制像素模式就直接绘制即可

**Pixel的自由落体**
```CPP
//move pixels ----------------------  
int2 pID = id.xy;  
float4 p = Result[pID];  
  
if(p.x == 1)  
{  
    //move down  
    int2 direction = int2( 0 , -1 );  
    int2 pID_new = pID+direction;//*(10*p.y);  
    float4 p_new = GetResultPixel(pID_new);  
  
    //if not empty - move horizontal  
    if(p_new.x > 0 || p_new.z > 0)  
    {       
	   direction = int2( sign(random(float2(pID) + _Time)-0.5f) , -1 );  
       pID_new = pID+direction;       
       p_new = GetResultPixel(pID_new);  
    }  
    //if empty - assign  
    if(p_new.x == 0 && p_new.z == 0)  
    {       
	   Result[pID_new] = p;  
       Result[pID] = float4(0,0,0,1);  
    }
}
```
大致逻辑如下：
如果目标像素的R值为1（也就是黄色像素）进入逻辑
给原像素的ID往下位移一个像素，再获取位移后的颜色
如果这个颜色不为黑，就根据PID和时间水平给一个方向值，如果这个颜色为0，就将R值为1的目标像素颜色赋值到这个向下位移一个像素的ID，原ID设置为黑色

自己项目里写一个类似的玩玩
![Alt Text](Textures/GIF2024-5-519-54-10.gif)



### StructuredBuffer

这一张主要是学习**StructuredBuffer**的用法

**RWStructuredBuffer**
它是一个可读写的buffer，并且我们可以指定buffer中的数据类型为我们**自定义**的struct类型，不用再局限于int，float这类的基本类型

先来一个简单的案例理解StructuredBuffer

首先ComputeBuffer是可以单独使用在Shader当中的，可以脱离ComputeShader使用

在C#中定义Struct结构体和ComputeBuffer
```Csharp
struct myObjectStruct  
{  
    public Vector3 objPosition;  
};  
private ComputeBuffer _computeBuffer;  
private myObjectStruct[] _mos;
void Start ()  
{  
    _noOfObj = _myObjects.Length;  
    _mos = new myObjectStruct[_noOfObj];  
    //Initiate buffer  
    _computeBuffer = new ComputeBuffer(_noOfObj,12);
}
```
其中count代表我们buffer中元素的数量，而stride指的是每个元素占用的空间（字节）。需要注意的是**ComputeBuffer中的stride大小必须和RWStructuredBuffer中每个元素的大小一致**。
比如上述代码，一个Struct里有一个Vector3，等价float3，一个float占用4bytes，那就是12bytes

```Csharp
//Set buffer  
_computeBuffer.SetData(_mos);  
//Assign buffer to unlit shader  
Mat.SetBuffer("myObjectBuffer", _computeBuffer);

void OnDestroy()  
{  
    //Clean Buffer  
    _computeBuffer.Release();  
}
```
SetData往Buffer里面填充数据，SetBuffer传输到Shader里，记得最后用完需要Release()掉

Shader里如何使用呢？
```hlsl
struct myObjectStruct  
{  
    float3 objPosition;  
};  
StructuredBuffer<myObjectStruct> myObjectBuffer;

VS:
[unroll]  
for(int i=0; i< 2; i++) //only 2 spheres in scene  
{  
    dist = abs(distance(myObjectBuffer[i].objPosition, wvertex.xyz));  
    power = 1 - clamp(dist / _EmissionDistance, 0.0f, 1.0f);  
    o.emissionPower += power;  
}
```
需要定义一个一模一样的Struct，然后在VS or FS中循环使用

再来看一个带有ComputeShader的案例

通过摄像机绘制Scene到RT，再把RT输入到ComputeShader中计算输出到屏幕
先看ComputeShader
```Csharp
struct Particle  
{  
    float2 position;  
    float direction;  
    float intensity;  
    float4 color;  
};  
RWStructuredBuffer<Particle> particleBuffer; 
Texture2D<float4> texRef;  
uint texSizeX;  
uint texSizeY;
```
定义结构体
定义一个只读不写Texture用于接收屏幕RT
用于接收长宽的两个参数

```Csharp
uint2 uv = particleBuffer[id.x].position * uint2(texSizeX,texSizeY);  
float angle = particleBuffer[id.x].direction;  
float2 dir = float2( cos(angle) , sin(angle) );  
float intensity = 0;

for(int i=0; i<STEP; i++)  
{  
    uint2 nuv = uv + i * dir * 3.0f;  
    if(nuv.x >= texSizeX || nuv.y >= texSizeY)  
    {       
	    break;  
    }  
    float4 col = texRef[nuv];  
    float f = 1.0 - (col.r+col.g+col.b)/3.0;  
  
    intensity += f;
}
particleBuffer[id.x].intensity = intensity;  
particleBuffer[id.x].color = texRef[uv];
```
核函数大概是一个类似于滤镜的算法，这个得结合最后的渲染Shader来看
