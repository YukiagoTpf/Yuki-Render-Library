# Tools

### 光滑法线生成
1. 工具生成
可以存放在VertexColor/UV3/UV4中

![Alt Text](20230128210110.png)

2. 直接写入fbx

[【Job/Toon Shading Workflow】自动生成硬表面模型Outline Normal](https://zhuanlan.zhihu.com/p/107664564)

我在原文的代码基础上修改了一部分，写入了模型的UV2中（因为大部分卡渲都是没有使用UV2，而部分使用了顶点色，保留顶点色的数据比较有必要）
```C#

Vector3 smoothedNormals = Vector3.zero;  
for (int i = 0; i < result.Length; i++)  
{  
    if (result[i][vertrx[index]] != Vector3.zero)  
        smoothedNormals += result[i][vertrx[index]];  
    else  
        break;  
}

smoothedNormals = smoothedNormals.normalized;  
  
var binormal = (Vector3.Cross(normals[index], tangents[index]) * tangents[index].w).normalized;  
  
var tbn = new Matrix4x4(  
    tangents[index],  
    binormal,    normals[index],  
    Vector4.zero);  
tbn = tbn.transpose;  
  
var bakedNormal = tbn.MultiplyVector(smoothedNormals).normalized;

Vector2 smoothnormaluv3 = new Vector2();  
smoothnormaluv3 = bakedNormal;  
uv3[index] = smoothnormaluv3;

```

**Shader内Unpack**
```hlsl
float3 bakedNormal = v.uv2.xyz;  
bakedNormal.z = sqrt(1.0 - saturate(dot(bakedNormal.xy, bakedNormal.xy)));  
bakedNormal = normalize(bakedNormal);  
float3 uv2Normal = bakedNormal;
// 此数据为切线数据
```

# Shader

### 法线背面外扩

```hlsl
VS：
float4 position_cs = mul(UNITY_MATRIX_MV, v.vertex);
position_cs = position_cs / position_cs.w;
//Z-Offset
float3 viewDir = normalize(position_cs.xyz);
position_cs.xyz = position_cs.xyz + viewDir * _MaxOutlineZOffset;
float4 clipPosition = mul(UNITY_MATRIX_P, position_cs);
float2 offset = normalize(clipNormal.xy) * _ScreenResolution.xy  * _OutlineFactor * _OutlineWidth * clipPosition.w* 2;
clipPosition.xy += offset;
o.pos = clipPosition;
return o;

PS：
return _OutlineColor;//Base
```