## 流程

1. 利用C#的**ComputeBuffer**将所有物体的localToWorldMatrix传递到ComputeShader的**StructuredBuffer**中
2. 将包围盒的八个顶点从ObjectSpace转换到WorldSpace，通过ComputeShader判断是否在视锥体内，在的话就把相对应的localToWorldMatrix传入ComputeBuffer
3. 渲染被剔除之后的Mesh

## 实操

#### 效果
![Alt Text](Textures/GIF2024-5-619-54-34.gif)
#### C#阶段准备数据
为了计算物体的包围盒是否在视锥体中，需要几个数据：
- Mesh的Local包围体八个顶点数据
> 这个不需要传入，直接在CS中计算即可（也可以通过脚本计算传入）
- Mesh所有实例的LocaltoWorldMatrix
> ```Csharp
> for (int i = 0; i < instanceCount; i++) 
> {
> 	......
> 	localToWorldMatrixs.Add(Matrix4x4.TRS(positions[i], Quaternion.identity, new Vector3(size, size, size)));
> 	//Matrix4x4 localToWorldMatrix = Matrix4x4.TRS(position, quaternion, scale)
> }
> ```
- 物体总数
>```Csharp
>compute.SetInt("instanceCount", instanceCount);
>```
- 视锥体的八面数据
> ```Csharp
> Vector4[] planes = GetFrustumPlane(mainCamera);
> compute.SetVectorArray("planes", planes);
> ```
#### ComputeShader计算
计算是否在视锥体内，并把符合要求的LocalToWorld矩阵通过**AppendStructuredBuffer**存入输出的Buffer中
输出之后将剔除之后的数量存入argsBuffer
```Csharp
ComputeBuffer.CopyCount(cullResult, argsBuffer, sizeof(uint));
```
#### DrawMeshInstanced
为了绘制成千上百个相同的Mesh，我们需要使用GPU Instancing
[Graphics.DrawMeshInstancedIndirect](https://docs.unity3d.com/ScriptReference/Graphics.DrawMeshInstancedIndirect.html).可以帮助我们绕过GPU to CPU的步骤，只需要把ComputeShader处理之后的结果直接放入渲染管线中渲染
官方文档中完整的函数如下

```Csharp
DrawMeshInstancedIndirect(Mesh mesh,int submeshIndex,Material mat,Bounds bounds,ComputeBuffer bufferwithArgs,int argsOffset = 0, MaterialPropertyBlock properties = null, Rendering.ShadowCastingMode castShadows = ShadowCastingMode.On, bool receiveShadows = true, int layer = 0, Camera camera = null, Rendering.LightProbeUsage lightProbeUsage = LightProbeUsage.BlendProbes, LightProbeProxyVolume lightProbeProxyVolume = null)
```

官方案例中使用方法如下：
```Csharp
Graphics.DrawMeshInstancedIndirect(instanceMesh, subMeshIndex, instanceMaterial, new Bounds(Vector3.zero, new Vector3(100.0f, 100.0f, 100.0f)), argsBuffer);
```
其他的都好理解，argsBuffer是什么？
> Buffer with arguments, bufferWithArgs, has to have five integer numbers at given argsOffset offset: index count per instance, instance count, start index location, base vertex location, start instance location.
argsBuffer就是一个包含了每个instance的索引数、instance数量、基本顶点位置、起始instance数量位置
```Csharp
args[0] = (uint)instanceMesh.GetIndexCount(subMeshIndex);  
args[1] = (uint)instanceCount;  
args[2] = (uint)instanceMesh.GetIndexStart(subMeshIndex);  
args[3] = (uint)instanceMesh.GetBaseVertex(subMeshIndex);
```

----
## Hi-Z遮挡