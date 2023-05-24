[shader-example-记录](..\..\srp-urp\shader-example\shader-example-记录.md)


## 坐标转换
[view 坐标系 到 clip 坐标系  左右手坐标系都不一样！！！ ](..\..\..\渲染-Shader\shader-相关\Shader入门精要-阅读笔记.md)
// space at the end of the variable name. Example: NormalWS
- OS: object space

- WS: world space 世界坐标系空间

- VS: view space  观察空间

<!-- - RWS: Camera-Relative world space. A space where the translation of the camera have already been substract in order to improve precision -->

- CS 也 写成 HCS : Homogenous clip spaces  齐次裁剪空间（把矢量扩展到四维空间下）   **在顶点着色器里面完成这个操作**
unity 里是 [-1, 1]  前提是要除下 w
 
- TS: tangent space

- TXS: texture space

- positionSTS     position in shadow texture space

- NDC: Normalized Device Coordinates   
**NDC 貌似没有标准的范围， unity 里面感觉应该算 [0, 1]**

Vertex Shader的输出在Clip Space，**接着GPU会做透视除法变到NDC**。
这之后GPU还有一步，应用视口变换，转换到Screen Space，输入给Fragment Shader
(Vertex Shader) => Clip Space => (透视除法) => NDC => (视口变换) => Screen Space => (Fragment Shader)


-------------------------------

**ShaderVariablesFunctions.hlsl  里面有很多计算函数**
- 包括算 positionNDC_or_positionProjected 
- 算法线
- 等等等

## 计算
```csharp
Windows 上： UNITY_REVERSED_Z  是被定义了的！




// 世界坐标系  世界距离 float3!
float3 positionWS = TransformObjectToWorld(positionOS);  // mul(GetObjectToWorldMatrix(), float4(positionOS, 1.0)).xyz;
// 相机坐标系  世界距离 float3!
float3 positionVS = TransformWorldToView(input.positionWS); // mul(GetWorldToViewMatrix(), float4(positionWS, 1.0)).xyz;

/*
单位长度为 2 的立方体里面的点，原点是立方体的圆心，因为它是齐次坐标系，所表示的真实点的坐标是需要 / w 的！
vert 得到的是裁剪坐标系 (x', y', z', 1)   其中 x', y' z' 的范围是 [-1, 1] ， w 之所以是 1 是因为它是个点，所以补 1
但是它 齐次坐标系所以： (x', y', z', 1)   ===   (w*x', w*y', w*z', w*1)
最终存的数值就是: (w*x', w*y', w*z', w)

在 vert 中， 摄像机是原点(长度为 2 的立方体的中间)，且 y 从上 往下递增！！！
x: [-n 左边, n 右边] 世界距离   范围 [-w, w]
    x/w = -1 是左边缘，x/w = 0 是中心，x/w = 1 是右边缘
y: [-n 上边, n 下边] 世界距离   范围 [-w, w]
    对于沿垂直方向的 y/w 也是如此
z: [-n 远, n 近]     世界距离   范围 [-w, w]  非线性！！！
    z分量应用了不同的修饰，分别在 z = -1 和 z = 1 处处理对近平面和远平面的剪裁。这些极端之间的值是非线性分布的，最终会映射到我们用于对片段进行深度测试并最终写入深度缓冲区的值
    这种非线性 z 对于深度缓冲很有用，但对我们的目的来说不是很直观，这就是为什么 Unity 包含宏和着色器节点，它们会自动将其解码回线性的世界空间/眼睛空间深度，以方便我们使用。
w: [-n, n 离摄像机远的为正] 世界距离 
    它是被绘制像素的世界空间深度，从相机沿其观察轴测量。（也称为眼空间深度）


在 frag 中， 原点是左下角 （x 从左到右 递增， y 从下到上递增）
因为是 SV_POSITION 语义，所以会被自动转换 ：
Vertex Shader的输出在Clip Space，接着GPU会做透视除法变到NDC。这之后GPU还有一步，应用视口变换，转换到Screen Space，输入给Fragment Shader
(Vertex Shader) => Clip Space => (透视除法) => NDC => (视口变换) => Screen Space => (Fragment Shader)

x: [0 左边, 1600 右边] 屏幕像素
y: [0 下边, 900 上边] 屏幕像素
z: [0 远, 1 近]        
w: [-n, n 离摄像机远的为正] 世界距离  和 vert 中的值一样！ 
*/
float4 positionCS = TransformWorldToHClip(input.positionWS); // mul(GetWorldToHClipMatrix(), float4(positionWS, 1.0)); 

/*
在 frag 中，不会被转义，和 vert 中的 positionCS 一样。
但是不在可视范围内的元素，都会被丢弃。那么 w 之类的范围，就是会被限制 [near 正数, far 正数] 里面
可以通过专门存一个变量传给 frag
*/
float4 tempHCS = positionCS


/*
在顶点着色器里计算
就是将 齐次坐标系点 positionCS(x, y)  转到 [0*w, 1*w] 的范围（ 需要自己手动 / w 才会变成[0, 1]的范围 ）， zw 保持不变

原点是左下角  （x 从左到右 递增， y 从下到上递增）
positionNDC、 positionProjected 貌似指的的是同一个东西
x: [0 左边, n 右边] 这个 n 貌似就是立方体的一半吧？原来是 [-n, n] 现在变成了 [0, n]
y: [0 下边, n 上边] 上下 反转了，世界距离
z: 和 tempHCS 一样
w: 和 tempHCS 一样

x,y /  w 之后，就变成 了 [0, 1] ，可以用于采样屏幕贴图！！！
z / w 之后 应该也是变成了 [0 离摄像机远, 1 距离摄像机近] ，但是因为精度分布的原因，所以画面可能还是黑的，可以通过调节近平面 来改善！
*/
float4 ndc = OUT.positionCS * 0.5f; // 得到   x:[-0.5w, -0.5w], y:[-0.5w, -0.5w], z:[-0.5w, -0.5w], w:0.5w
// 这里的 _ProjectionParams.x 是 -1 ！！！    (-1 if projection is flipped)
// 在 unity urp 里 projectedPosition 貌似和 positionNDC 是同一个东西。。。
OUT.positionNDC_or_positionProjected.xy = float2(ndc.x, ndc.y * _ProjectionParams.x) + ndc.w;   // 得到 x:[0w, 1w]  y:[0w, 1w]
OUT.positionNDC_or_positionProjected.zw = OUT.positionCS.zw;   // 得到 z:[-w, w], w:w

////////////////////////////// 上面部分，都是在 vert 中计算 ///////////////////////////////////////


// frag 里面调用
// projectedPosition (x/w: [0, 1], y/w: [0, 1], z/w:[-1, 1], w: 齐次坐标系里，用于除数)
float CustomParticlesSoft(float near, float invFar, float4 projectedPosition)
{
    // float invFar = 1/(far - near);
    float fade = 1;
    if (near > 0.0 || invFar > 0.0)
    {
        // [0, 1]
        float rawDepth = SAMPLE_TEXTURE2D_X(_CameraDepthTexture, sampler_CameraDepthTexture, UnityStereoTransformScreenSpaceTex(projectedPosition.xy / projectedPosition.w)).r;
        /* 
        世界距离
        ProjectZ 和 eyeDepth 并不完全相等， projZ 相等，只能说明是在同一平面，但是 eyeDepth 是平面中间的最小
        LinearDepthToEyeDepth 用于算正交的深度  里面处理 UNITY_REVERSED_Z 情况   , _ZBufferParams 本身已经区分了 UNITY_REVERSED_Z， 所以 LinearEyeDepth 里不需要处理
        */
        float sceneZ = (unity_OrthoParams.w == 0) ? LinearEyeDepth(rawDepth, _ZBufferParams) : LinearDepthToEyeDepth(rawDepth);
        float thisZ = LinearEyeDepth(projectedPosition.z / projectedPosition.w, _ZBufferParams);
        fade = saturate( invFar * ((sceneZ - near) - thisZ));
    }
    return fade;
}

// Z buffer to linear depth.
// Does NOT correctly handle oblique view frustums.
// Does NOT work with orthographic projection.
// zBufferParam = { (f-n)/n, 1, (f-n)/n*f, 1/f } // ？？？为什么两处 zBufferParam 的定义不一样？？？ 其实是一样的。。。。下面列了下换算流程
// 输入 depth  [0, 1] 的范围是非线性的！ 具体可以看 原理-深度.md
// 返回的结果是 世界距离: [near, far]    （depth == 0  则返回  f）  （depth == 1  则返回 n）
float LinearEyeDepth(float depth, float4 zBufferParam)
{
    return 1.0 / (zBufferParam.z * depth + zBufferParam.w);
}

// Values used to linearize the Z buffer (http://www.humus.name/temp/Linearize%20depth.txt)
// x = 1-far/near
// y = far/near
// z = x/far    
// w = y/far  ===> 1/near
// or in case of a reversed depth buffer (UNITY_REVERSED_Z is 1)
// x = -1+far/near  ==> -n/n + f/n ==> (f - n)/n
// y = 1
// z = x/far   ==>  (-1 + f/n)/f  ==> -1/f + 1/n  ==>  -n/fn + f/fn  ==>  (f-n)/fn
// w = y/far   ==? 1/far
float4 _ZBufferParams;  // 这个和 LinearEyeDepth 里的注释 不一样。感觉是这里的数字不准。

// x = 1 or -1 (-1 if projection is flipped)
// y = near plane
// z = far plane
// w = 1/far plane
float4 _ProjectionParams;
```

### 采样 _CameraOpaqueTexture  _CameraDepthTexture


```js
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/DeclareDepthTexture.hlsl"
// DeclareOpaqueTexture.hlsl
// DeclareNormalsTexture.hlsl
float2 uv = IN.positionHCS.xy / _ScaledScreenParams.xy;  // 得到 0， 1 的 屏幕uv位置
// 方式1
SampleSceneDepth(uv);

// 或者
TEXTURE2D_X_FLOAT(_CameraOpaqueTexture);
SAMPLER(sampler_CameraOpaqueTexture);
half4 OUT = SAMPLE_TEXTURE2D_X(_CameraOpaqueTexture, sampler_CameraOpaqueTexture, uv);
```



### [Object Position](https://docs.unity3d.com/Packages/com.unity.shadergraph@12.0/manual/Object-Node.html)
unity的世界变化矩阵最后一列是存的Transform里的Position，所以我们可以在shader里提取这部分数据做一些计算，下面是unity支持的几种写法：


```js
#define SHADERGRAPH_OBJECT_POSITION UNITY_MATRIX_M._m03_m13_m23

GetObjectToWorldMatrix()
float3 center = float3(unity_ObjectToWorld[0].w, unity_ObjectToWorld[1].w, unity_ObjectToWorld[2].w);
float3 center = float3(unity_ObjectToWorld._m03, unity_ObjectToWorld._m13, unity_ObjectToWorld._m23);
float3 center = mul(unity_ObjectToWorld , float(0,0,0,1)).xyz;
float3 center = unity_ObjectToWorld._14_24_34;
```
### Object Scale
```js
float3 _Object_Scale = float3(length(float3(UNITY_MATRIX_M[0].x, UNITY_MATRIX_M[1].x, UNITY_MATRIX_M[2].x)),
                             length(float3(UNITY_MATRIX_M[0].y, UNITY_MATRIX_M[1].y, UNITY_MATRIX_M[2].y)),
                             length(float3(UNITY_MATRIX_M[0].z, UNITY_MATRIX_M[1].z, UNITY_MATRIX_M[2].z)));
```

---------------

## 法线计算 相关
[图形|基础|顶点法线、法线贴图、法线融合](https://zhuanlan.zhihu.com/p/533942130)

```js
TransformObjectToWorldDir
TransformWorldToViewDir(real3 dirWS, bool doNormalize = false)
TransformWorldToHClipDir


float3 TransformObjectToWorldNormal(float3 normalOS, bool doNormalize = true)
TransformWorldToObjectNormal




```




-----------------------------------------------------------------------------

## Unity 的内置矩阵

### UNITY_MATRIX_IT_MV 
可以把法线从模型空间变换到观察空间
float3 scale = float3(length(unity_ObjectToWorld._m00_m10_m20), length(unity_ObjectToWorld._m01_m11_m21), length(unity_ObjectToWorld._m02_m12_m22));
float3 norm = normalize(mul((float3x3)UNITY_MATRIX_IT_MV, IN.normal)) * scale;


---------------------------










<!-- 
之前的理解有点偏差


```CS
Windows（Direct3D 11.0）上： UNITY_REVERSED_Z  是被定义了的！

！！！可以用 csharp 写一下这个转换步骤，这样就很清楚了！！！

// 世界距离 float3!
float3 positionWS = TransformObjectToWorld(positionOS);  // mul(GetObjectToWorldMatrix(), float4(positionOS, 1.0)).xyz;
// 世界距离 float3!
float3 positionVS = TransformWorldToView(input.positionWS); // mul(GetWorldToViewMatrix(), float4(positionWS, 1.0)).xyz;

/*
vert 得到的结果是 float4 世界距离，里面的 w 可以给后面的 positionProjected 做除法运算，来得到 [0, 1] 范围的值！！！！！

在 vert 中， 摄像机是原点，且 y 从上 往下递增！！！
x: [-n 左边, n 右边] 世界距离
y: [-n 上边, n 下边] 世界距离
z: [-n 离摄像机远, n 距离摄像机近] 世界距离  貌似是算的 距离 near far 的距离？ 会受 near far 影响颜色。z 应该是从 near 开始算的
w: [-n, n 离摄像机远的为正] 世界距离  不受 near far 影响颜色

在 frag 中， 原点是左下角 （x 从左到右 递增， y 从下到上递增）
因为是 SV_POSITION 语义，所以会被自动转换 
x: [0 左边, 1600 右边] 屏幕像素
y: [0 下边, 900 上边] 屏幕像素
z: [0 距离摄像机远, 1 距离摄像机近]    
w: [-n, n 离摄像机远的为正] 世界距离  和 vert 中的值一样！ 也许也不是距离（但是的确离摄像机越远，这个值越大），待会看了视频，再确认
*/
！！！ 应该是一个立方体 吧！
float4 positionCS = TransformWorldToHClip(input.positionWS); // mul(GetWorldToHClipMatrix(), float4(positionWS, 1.0)); 

/*
在 frag 中，不会被转义，和 vert 中的 positionCS 一样。
但是不在可视范围内的元素，都会被丢弃。那么 w 之类的范围，就是会被限制 [near 正数, far 正数] 里面
*/
float4 tempHCS = positionCS


/*
本身应该已经是一个立方体了？！
原点是左下角  （x 从左到右 递增， y 从下到上递增）
positionNDC、 positionProjected 貌似指的的是同一个东西
x: [0 左边, n 右边] 世界距离 （不是01这么小的距离）， 这个 n 貌似就是立方体的一半吧？原来是 [-n, n] 现在变成了 [0, n]
y: [0 下边, n 上边] 上下 反转了，世界距离
z: 和 tempHCS 一样
w: 和 tempHCS 一样

x,y /  w 之后，就变成 了 [0, 1] ，可以用于采样屏幕贴图！！！    hlsl 里面有个【废弃的】方法 ComputeScreenPos 也可以看看
z / w 之后 应该也是变成了 [0 离摄像机远, 1 距离摄像机近] ，但是因为精度分布的原因，所以画面可能还是黑的，可以通过调节近平面 来改善！
*/
float4 ndc = input.positionCS * 0.5f;
// 这里的 _ProjectionParams.x 是 -1 ！！！    (-1 if projection is flipped)
// 在 unity urp 里 projectedPosition 貌似和 positionNDC 是同一个东西。。。
input.positionNDC_or_positionProjected.xy = float2(ndc.x, ndc.y * _ProjectionParams.x) + ndc.w;  
input.positionNDC_or_positionProjected.zw = input.positionCS.zw;  

////////////////////////////// 上面部分，都是在 vert 中计算 ///////////////////////////////////////


// frag 里面调用
// projectedPosition (x: [0, n] 世界距离, y: [0, n] 世界距离, z:[世界距离], w: 世界距离)
float CustomParticlesSoft(float near, float invFar, float4 projectedPosition)
{
    // float invFar = 1/(far - near);
    float fade = 1;
    if (near > 0.0 || invFar > 0.0)
    {
        // [0, 1]
        float rawDepth = SAMPLE_TEXTURE2D_X(_CameraDepthTexture, sampler_CameraDepthTexture, UnityStereoTransformScreenSpaceTex(projectedPosition.xy / projectedPosition.w)).r;
        /* 
        世界距离
        ProjectZ 和 eyeDepth 并不完全相等， projZ 相等，只能说明是在同一平面，但是 eyeDepth 是平面中间的最小
        LinearDepthToEyeDepth 用于算正交的深度  里面处理 UNITY_REVERSED_Z 情况   , _ZBufferParams 本身已经区分了 UNITY_REVERSED_Z， 所以 LinearEyeDepth 里不需要处理
        */
        float sceneZ = (unity_OrthoParams.w == 0) ? LinearEyeDepth(rawDepth, _ZBufferParams) : LinearDepthToEyeDepth(rawDepth);
        // 世界距离
        float thisZ = LinearEyeDepth(projectedPosition.z / projectedPosition.w, _ZBufferParams);
        fade = saturate( invFar * ((sceneZ - near) - thisZ));
    }
    return fade;
}


// Z buffer to linear depth.
// Does NOT correctly handle oblique view frustums.
// Does NOT work with orthographic projection.
// zBufferParam = { (f-n)/n, 1, (f-n)/n*f, 1/f }
float LinearEyeDepth(float depth, float4 zBufferParam)
{
    return 1.0 / (zBufferParam.z * depth + zBufferParam.w);
}

// Values used to linearize the Z buffer (http://www.humus.name/temp/Linearize%20depth.txt)
// x = 1-far/near
// y = far/near
// z = x/far    
// w = y/far   1/near
// or in case of a reversed depth buffer (UNITY_REVERSED_Z is 1)
// x = -1+far/near
// y = 1
// z = x/far
// w = 1/far
float4 _ZBufferParams;

// x = 1 or -1 (-1 if projection is flipped)
// y = near plane
// z = far plane
// w = 1/far plane
float4 _ProjectionParams;
```
 -->
