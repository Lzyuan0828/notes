## 渲染管线
### 通用渲染管线
- "RenderPipeline" = "UniversalPipeline"
- 
### 内置渲染管线
- "RenderPipeline" = "Built-in Render Pipeline"
- Unity默认的渲染管线
### 高清渲染管线
- "RenderPipeline" = "High Definition Render Pipeline"


## 光照模型

## 计算明暗面
dot : a·b = |a|·|b|·cos<a,b>
光照方向向量 · 法线向量
dot(i.normal,lightDir) 计算出的结果形象来讲是光线和法线的夹角cos值，原理来说夹角90°时，光照强度最大，因此这个结果越大，则光线强度越大。