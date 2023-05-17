# 渲染管线
### 通用渲染管线
- "RenderPipeline" = "UniversalPipeline"
### 内置渲染管线
- "RenderPipeline" = "Built-in Render Pipeline"
- Unity默认的渲染管线
### 高清渲染管线
- "RenderPipeline" = "High Definition Render Pipeline"


# 光照模型
## 计算明暗面
dot : a·b = |a|·|b|·cos<a,b>
光照方向向量 · 法线向量
dot(i.normal,lightDir) 计算出的结果形象来讲是光线和法线的夹角cos值，原理来说夹角90°时，光照强度最大，因此这个结果越大，则光线强度越大。

## 漫反射
- 漫反射计算公式C(diffuse) = (C(light)* m(diffuse))max(0,n·I)
- n是表面法线，I是*指向光源*的单位矢量，m(diffuse)是材质的漫反射颜色，C(light)是光源颜色
- 目前使用半兰伯特模型，公式是diffuse = C(light)* m(diffuse)* α(n·I)+β
- 相比较于原兰伯特模型，舍去了防止max操作，增加了一个缩放和偏移，通常情况下缩放和偏移值都是0.5
```
	float lightAtenuation = dot(worldLightDir, worldNormal) * 0.5 + 0.5;
```
- 原理其实就是上面的计算明暗面的原理，得到的值越大，证明光照强度越大
```
	half3 diffuse = light.color * _BaseColor.rgb * lightAtenuation;
```

## 计算阴影
- 在URP中得到阴影效果，首先需要声明一个新的PASS,例子如下
```
Pass
        {
            Tags
            {
                "LightMode" = "ShadowCaster"
            }

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct appdata
            {
                float4 vertex : POSITION;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;

            v2f vert(appdata v)
            {
                v2f o;
                o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
                return o;
            }

            float4 frag(v2f i) : SV_Target
            {
                float4 color;
                color.xyz = float3(0.0, 0.0, 0.0);
                return color;
            }
            ENDHLSL
        }
	
```
- 这个PASS的作用是通过"LightMode" = "ShadowCaster"在渲染的时候声明出来，这个物体参与阴影计算
- 但是这时候物体本身自己不会显示其他物体的阴影，我们需要再次声明
```
 	#pragma multi_compile _ _MAIN_LIGHT_SHADOWS
    #pragma multi_compile _ _MAIN_LIGHT_SHADOWS_CASCADE
```
## 高光反射
### Phong模型
```hlsl
	fixed3 reflectDir = normalize(reflect(-litDir, worldNormal));
	fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - mul(unity_ObjectToWorld, v.vertex).xyz);
	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)), _Gloss);
```
### Blinn - Phong 模型
```hlsl
	fixed3 viewDir = normalize(UnityWorldSpaceViewDir(worldPos));
    fixed3 halfDir = normalize(viewDir + litDir);
    fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(worldNormal, halfDir)), _Gloss);
```

### 卡通风格的高光渲染
原理是把计算出的高光值和阈值进行比较，如果小于该阈值，则高光反射系数为0
```hlsl
	float spec = dot(worldNormal,worldHalfDir);
	spec = step(threshold,spec);
```
这样的做法会产生锯齿，原因是暴力突变到0-1
抗锯齿做法如下：
```hlsl
	float spec = dot(worldNormal,worldHalfDir);
	float w = fwidth(spec) * 2.0;
	spec = lerp(0,1,smoothstep(-w,w,spec-threshold));
```
***！！！spec的精度如果是half或者更低，就会出现边缘粗糙的感觉！！！***
- 另一种做法
- 然后是高光specular的计算。用cg自带的reflect函数计算出反向的反射光方向，用正向的反射光方向与视角方向点乘得vDotRefl（与phong模型相似）。然后取样高光贴图值与_Glossiness相乘赋值给smoothness（贴图控制高光形状，_Glossiness控制高光大小）。使用step函数来对高光值取0或1（也就得到了硬的成块的高光效果）乘以_SpecularColor高光颜色，然后赋值给specular：
```
	float3 refl = reflect(lightDir, normal);
    float vDotRefl = dot(viewDir, -refl);
    float smoothness = tex2D(_SpecularMap, i.uv.zw).x * _Glossiness;
    float3 specular = _SpecularColor.rgb * step(1 - smoothness, vDotRefl);
```



# 法线贴图
### URP中使用法线贴图
- 目前使用的法线贴图 都是切线空间下的 因此我们需要在顶点着色器中完成对灯光、视角的转换
- 模型空间->切线空间矩阵求法：
- 因为知道切线相对模型空间的向量，因此将切线空间的三轴按照列排布即为  切线空间->模型空间的矩阵
- 因为切线空间是正交矩阵，因此转置矩阵等于逆矩阵，那么只需要将 切线空间->模型空间转置（按照行排布），即可得到 模型空间->切线空间
```
	_BumpMap ("Normal Map",2D) = "bump" {}
    _BumpScale("Scale", Float) = 1.0
```
```
		v2f vert(a2v v)
            {
                v2f o;
                o.pos = TransformObjectToHClip(v.vertex);
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
                float3 binormal = cross(v.normal, v.tangent.xyz) * v.tangent.w;
                // tangent、binormal、normal为模型坐标系下的表示
                // 转置摆放后（按行摆放，按列摆放的话为 切线空间->模型空间）即为 模型空间->切线空间 的矩阵
                float3x3 rotation = float3x3(v.tangent.xyz, binormal, v.normal);
                //TANGENT_SPACE_ROTATION;
                // URP 中没有 ObjSpaceLightDir
                half3 objectSpaceLightDir = TransformWorldToObjectDir(_MainLightPosition.xyz);
                o.tangentLightDir = mul(rotation, objectSpaceLightDir).xyz;
                half3 objectSpaceViewDir = half3(TransformWorldToObject(_WorldSpaceCameraPos.xyz) - v.vertex.xyz);
                o.tangentViewDir = mul(rotation, objectSpaceViewDir).xyz;
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                return o;
            }
```




# alpha贴图
感觉是 简单一张贴图使用alpha通道就可以。
具体可以参照透明度测试


# URP中的TAGS
### "LightMode"
- DepthOnly 遇到了一个问题是，在shader中没有声明DepthOnly的Pass的时候，在Scene视角下，似乎读取不到深度。声明后变为正常。
    + 官方文档对于DepthOnly的描述如下：The Pass renders only depth information from the perspective of a Camera into a depth texture.

## 后处理
### Unity中的后处理
[Unity中的后处理](https://johnyoung404.github.io/2019/12/13/Unity%E4%B8%AD%E7%9A%84%E5%90%8E%E5%A4%84%E7%90%86-post-processing/)
### 如何扩展Unity URP的后处理Volum组件
[如何扩展UnityURP的后处理Volum组件](https://www.zhihu.com/tardis/zm/art/161658349?source_id=1003)
- local化URP源码修改
### renderFeature的顺序
```
 public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
        {
            m_ScriptablePass.renderPassEvent = RenderPassEvent.BeforeRenderingPostProcessing;
            renderer.EnqueuePass(m_ScriptablePass);
        }
```
通过 RenderPassEvent定义顺序

# Blend Type
Blend SrcAlpha OneMinusSrcAlpha // 传统透明度
Blend One OneMinusSrcAlpha // 预乘透明度
Blend One One // 加法
Blend OneMinusDstColor One // 软加法
Blend DstColor Zero // 乘法
Blend DstColor SrcColor // 2x 乘法








# 问题小记
- 高光生成区域不规则，和例图差距较大
    + 比对后发现计算法线时候没有归一化处理half3 worldNormal = i.worldNormal/half3 worldNormal = normalize(i.worldNormal);
    + 这个处理放在顶点着色器中应该也是不对的，因为到片元内是插值，归一化会失去作用。
- 高光区域边缘粗糙
    +   计算值的精度问题