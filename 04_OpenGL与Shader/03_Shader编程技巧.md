# 03 Shader 编程技巧

> 本文由 `Shader/` 下的 11 篇笔记重组而成，覆盖编译控制、并行性思维、纹理与采样器、Instancing、绘制线、翻转书、全景天空盒与性能陷阱。语法基础见 `02_GLSL语法与内建函数`。

## 一、Shader Model 与变体

### 1.1 Shader Model 是什么

Shader Model（SM）是微软与 GPU 厂商共同约定的**着色器能力规范**，规定某一代硬件/驱动上可用的指令数量、寄存器数量、分支能力、纹理指令、计算着色器等。它和 .NET 的版本号性质类似：只是「这一档能支持哪些特性」的清单。

| 版本 | 代表 API | 关键能力 |
| --- | --- | --- |
| SM 2.0 | D3D9 | 指令槽有限，以静态分支为主，MRT 支持受限 |
| SM 3.0 | D3D9c | 动态分支、顶点纹理拾取、更长的指令序列 |
| SM 4.0 | D3D10 | 统一着色器架构、整数指令、纹理数组、几何着色器 |
| SM 5.0 | D3D11 | 计算着色器、 tessellation、UAV、结构化缓冲 |
| SM 6.x | D3D12 | Wave  intrinsics、64 位整数、光追、mesh shader |

几个常见误解：

- SM 是**针对 HLSL/D3D** 的规范，GLSL 由 Khronos 独立版本化，两者不是同一套编号，不能一一对应。
- 但 SM 事实上**约束了同时代 GLSL 的实现下限**：硬件要同时满足两边的能力，所以「SM 3.0 级别」的 GPU 上 GLSL 也基本停在对应特性档位。
- GLSL 也可能**超出**同代 SM 的设计（例如某些扩展），所以不能反过来用 SM 推断 GLSL 一定支持什么。

工程上的意义只有一个：**你的 shader 能跑在哪些设备上**。Unity 用 `#pragma target` 把这件事显式化。

### 1.2 Unity 的编译目标与变体

```hlsl
#pragma target 2.0    // GLES2 / D3D9 级别
#pragma target 3.0    // GLES3 / D3D11 基础
#pragma target 3.5    // 支持 DX11 风格 texture/sampler 分离
#pragma target 4.5    // 计算、结构化缓冲、DOTS instancing 常用档位
```

变体（variant）是「同一份 shader 代码 + 不同宏组合」编译出的多个程序。声明方式：

```hlsl
#pragma multi_compile _ _MAIN_LIGHT_SHADOWS _MAIN_LIGHT_SHADOWS_CASCADE
#pragma shader_feature _ _ALPHATEST_ON
#pragma multi_compile_instancing
#pragma multi_compile _ DOTS_INSTANCING_ON
```

| 指令 | 行为 |
| --- | --- |
| `multi_compile` | 全组合都编译，运行时用 `EnableKeyword` 切换；关键词会被打包进构建 |
| `shader_feature` | 只编译材质实际用到的组合，未被任何材质引用的变体在打包时被剔除 |
| `multi_compile_local` / `shader_feature_local` | 关键词作用域限制在本 shader，避免全局关键词互相干扰 |
| `multi_compile_fragment` | 只影响片元阶段的变体（减少顶点变体） |

变体爆炸的代价是**构建时间、包体、内存**，而不是运行时性能。经验：

- 用 `shader_feature` 表达「材质级开关」，用 `multi_compile` 表达「全局/管线级开关」。
- 能用 uniform 分支解决的（比如一个 `_UseNormalMap` 浮点），不要用 keyword，尤其当分支两侧代价接近时。
- 但 uniform 分支**不会**剔除代码，寄存器压力和采样器数量仍然按最坏情况计。

## 二、编译控制：branch / flatten / unroll / loop

这些是给 HLSL 编译器的**提示标签**（attribute），不是语义。

### 2.1 if：`branch` 与 `flatten`

```hlsl
[branch]
if (cond) { A(); } else { B(); }

[flatten]
if (cond) { A(); } else { B(); }
```

- `branch`：生成真正的跳转指令，按条件只执行一侧。省指令，但会产生分支开销与 warp 内发散（同一 warp 内两种结果都要串行执行两侧）。
- `flatten`：两侧都执行，最后按条件选择结果（谓词化 / 选择指令）。没有跳转，但机器码更多、寄存器压力更大。

关键结论：

- `flatten` **不是**「静态分支」。它仍然依赖运行时条件，只是把控制流变成数据流；因此两侧的**代价都会付出**。原笔记里「flatten 让编译器认为是静态分支」的说法不准确——真正让流变为一致（uniform）的是「条件本身是 uniform」或「循环被展开」。
- 分支体很小（几条指令）时 `flatten` 更快；分支体很大且通常只走一侧时 `branch` 更快。
- 条件是 uniform（同一 draw 内所有像素一致）时，编译器通常能自动提出去，等价于没有分支。
- 老平台（GLES2）没有真正的动态分支，编译器**总是** flatten。

### 2.2 for：`loop` 与 `unroll`

```hlsl
[loop]
for (uint i = 0; i < n; ++i) { ... }     // 不展开，保留循环

[unroll]
for (uint i = 0; i < 4; ++i) { ... }     // 完全展开

[unroll(4)]
for (uint i = 0; i < n; ++i) { ... }     // 最多展开 4 次，剩余部分仍循环
```

- 不写标签时由编译器决定：次数已知且很小的循环通常会被自动展开；`[loop]` 显式禁止展开，`[unroll]` 显式要求展开。原笔记「`for` 默认是 loop」的说法只在部分编译器/平台上成立。
- 展开的收益：去掉循环计数与跳转、把「变量下标」变成常量下标（这对采样器数组/常量数组很关键）、让编译器做跨迭代优化。
- 展开的代价：机器码膨胀、编译时间变长、指令缓存命中率下降;循环次数大时反而更慢。
- 次数在编译期不可知时不要 `unroll`，否则只能部分展开或直接失败。

### 2.3 组合策略

经验上的搭配是「动态配动态、静态配静态」：

- `loop` + `branch`：次数不定、每次可能要跳过大量工作（如遍历灯光、heightfield）。
- `unroll` + `flatten`：次数固定且很少（如 4 个采样桶、2 个光源），全展开后用选择指令更划算。

UE4 的代码里这种配对非常明显，可作为阅读参考。

### 2.4 动态分支里为什么不能 `tex2D`

```hlsl
[ branch ]
if (cond) { color = tex2D(_MainTex, uv); }   // 可能编译报错
```

原因：GPU 以 2×2 像素的 quad 为单位着色，隐式 LOD 需要 `dFdx/dFdy` 求 UV 在屏幕上的变化率。当 quad 内像素走了不同分支时，导数没有定义，硬件无法选 mip。

解决办法（按优先级）：

1. 用显式 LOD：`tex2Dlod(_MainTex, float4(uv, 0, 0))`；Unity 里 `SAMPLE_TEXTURE2D_LOD`。
2. 用显式梯度：`tex2Dgrad(_MainTex, uv, ddx(uv), ddy(uv))`。
3. 让分支变成一致流：条件为 uniform，或把循环 `unroll` 到编译期可知。
4. 用 `flatten` 把两侧都执行——采样对所有像素走同一路径，导数可计算；但代价是没走的那侧也采样了。（谓词化后导数是否在所有编译器上都被认为合法，官方文档没有明确保证，属经验做法，**待核实**）

第 4 条只是「能编译且正确」，性能上未必划算：纹理采样很贵，如果大部分像素不需要采样，`branch` + `tex2Dlod` 通常更快。

原笔记里「`branch` 下不能采样 mipmap，只能显式采样 level 0」应更准确地表述为：**不能使用依赖隐式导数的采样**；`tex2Dlod` 可以指定任意 level，不局限于 0。

### 2.5 一段真实风格的示例

```hlsl
UNROLL
for (uint k = 0; k < Radius; k++)
{
    float bucketDistance = radiusMultiplier * (float(k + 1) * COC_TILE_SIZE) - COC_TILE_SIZE;

    FLATTEN
    if (-outBuckets[k].fgdMinCoc * cocRadiusToBucketDistanceUpperBound > bucketDistance)
    {
        outSample.fgdMinCoc = min(outSample.fgdMinCoc, outBuckets[k].fgdMinCoc);
    }

    FLATTEN
    if (outBuckets[k].bgdMaxCoc * cocRadiusToBucketDistanceUpperBound > bucketDistance)
    {
        outSample.bgdMaxCoc = max(outSample.bgdMaxCoc, outBuckets[k].bgdMaxCoc);
    }
}

LOOP
for (uint i = 0; i < numHeightfields; i++)
{
    float3 localPos = mul(float4(interpolatedPos, 1), worldToLocal[i]).xyz;
    float2 uv = getHeightfieldUV(i, localPos.xy, minMaxUV);

    BRANCH
    if (all(uv > minMaxUV.xy) && all(uv < minMaxUV.zw))
    {
        float3 shadingPos = getHeightfieldWorldPosition(i, localPos.xy, uv);
        if (interpolatedPos.z + sampleRadius * 1.5 < shadingPos.z)
            triangleVisible = false;
    }
}
```

`UNROLL/FLATTEN/LOOP/BRANCH` 是 UE 的宏；Unity 里对应 `UNITY_UNROLL`（UE 的 `UNROLL(n)` 带次数，Unity 侧没有等价的次数参数，**待核实**）、`UNITY_FLATTEN`、`UNITY_LOOP`、`UNITY_BRANCH`（`UNITY_BRANCH` 用多了在部分平台会编译失败）。

## 三、充分利用 GPU 的并行性

### 3.1 思维差异

CPU 版代码追求「尽早得出结果」：逐分量判断，一发现不可能就 `return false`。GPU 版代码追求「让所有 lane 做同样的工作」：三个分量一起算，最后用 `min/max` 归约，全程无分支。

同一个「射线与 AABB 求交」，两种写法：

```cpp
// CPU：逐轴早退
bool rayIntersect(const Ray3f &ray) const
{
    float nearT = -std::numeric_limits<float>::infinity();
    float farT  =  std::numeric_limits<float>::infinity();

    for (int i = 0; i < 3; ++i)
    {
        float origin = ray.o[i];
        float minVal = min[i], maxVal = max[i];

        if (ray.d[i] == 0.0f)
        {
            if (origin < minVal || origin > maxVal) return false;
            continue;
        }

        float t1 = (minVal - origin) * ray.dRcp[i];
        float t2 = (maxVal - origin) * ray.dRcp[i];
        if (t1 > t2) std::swap(t1, t2);

        nearT = std::max(t1, nearT);
        farT  = std::min(t2, farT);
        if (nearT > farT) return false;
    }
    return true;
}
```

```hlsl
// GPU：三分量统一计算，无分支
float2 rayAABBoxDist(float3 boundsMin, float3 boundsMax,
                     float3 rayOrigin, float3 rayDir)
{
    float3 invDir = 1.0 / rayDir;
    float3 t0 = (boundsMin - rayOrigin) * invDir;
    float3 t1 = (boundsMax - rayOrigin) * invDir;

    float3 tmin = min(t0, t1);
    float3 tmax = max(t0, t1);

    float nearT = max(max(tmin.x, tmin.y), tmin.z);
    float farT  = min(tmax.x, min(tmax.y, tmax.z));

    float dstToBox = max(0.0, nearT);              // 起点在盒内则为 0
    float dstInBox = max(0.0, farT - dstToBox);    // 盒内穿行距离
    return float2(dstToBox, dstInBox);
}

bool rayHitsAABBox(float3 boundsMin, float3 boundsMax,
                   float3 rayOrigin, float3 rayDir)
{
    float3 invDir = 1.0 / rayDir;
    float3 t0 = (boundsMin - rayOrigin) * invDir;
    float3 t1 = (boundsMax - rayOrigin) * invDir;
    float3 tmin = min(t0, t1);
    float3 tmax = max(t0, t1);

    float nearT = max(max(tmin.x, tmin.y), tmin.z);
    float farT  = min(tmax.x, min(tmax.y), tmax.z);
    return nearT < farT;
}
```

### 3.2 无分支写法的坑

1. **除零得到 ±Inf 是「有用」的**：`rayDir` 某分量为 0 时 `1/0 = +Inf`，`(min - o) * Inf` 得到 `±Inf`，`min/max` 仍然给出正确区间——这是这套写法的前提。但如果射线原点恰好落在包围盒边界上，会得到 `0 * Inf = NaN`，`min/max` 的传播结果依实现而定。稳妥做法是用 `step` 构造保号因子（不能直接用 `sign`：`sign(0) = 0`，零分量乘完仍是 0，`1/0` 仍然是 Inf）：

```hlsl
float3 signDir = step(0.0, rayDir) * 2.0 - 1.0;   // 0 归入正号，避免 sign(0) = 0
float3 safeDir = signDir * max(abs(rayDir), 1e-8);
float3 invDir = 1.0 / safeDir;
```

2. `min/max` 对 NaN 的行为在 HLSL/GLSL 中不保证（`min(NaN, x)` 可能返回任一），所以不要指望 NaN 被「吃掉」。
3. 归约顺序可以优化：先 `max(tmin.x, tmin.y)` 再与 `tmin.z` 取 `max`，比三项一起写少一次依赖。
4. 不要为了「无分支」把真正昂贵的计算也两边都算——分支的收益取决于两侧代价比。

### 3.3 更一般的并行性建议

- 同一 warp 内的像素尽量走相同路径：把「材质级」差异做成 keyword/uniform 而不是逐像素随机分支。
- 避免长依赖链：`a = f(b); b = g(c);` 这类串行会拉长延迟，GPU 靠切换 warp 隐藏延迟，但依赖链太长仍会掉占用率。
- 向量化：能一次算 3 个分量就不要写 3 次标量代码（编译器不一定帮你合并）。
- 纹理采样是主要瓶颈之一：减少采样次数、合并通道（把 roughness/metal/AO 打包进一张图）、用 `textureGather` 一次取 4 个 texel。

## 四、Separate textures and samplers

### 4.1 为什么要分离

许多图形 API 与 GPU 允许**采样器数量少于纹理数量**。D3D11 在单个 shader 中最多 128 张纹理，但采样器最多 16 个。如果每个纹理都和自己的采样器一一绑定（耦合语法），纹理一多就会超出采样器上限。

D3D11 风格 HLSL 的写法：

```hlsl
Texture2D    _MainTex;
SamplerState sampler_MainTex;          // "sampler" + 纹理名，Unity 的配对约定
// ...
half4 color = _MainTex.Sample(sampler_MainTex, uv);
```

由于纹理与采样器解耦，可以让多张纹理**共用一个采样器**：

```hlsl
Texture2D    _MainTex;
Texture2D    _SecondTex;
Texture2D    _ThirdTex;
SamplerState sampler_MainTex;

half4 color = _MainTex.Sample(sampler_MainTex, uv);
color += _SecondTex.Sample(sampler_MainTex, uv);
color += _ThirdTex.Sample(sampler_MainTex, uv);
```

Unity 的规则：名为 `sampler` + `纹理名` 的 `SamplerState` 会自动取该纹理的导入设置（过滤、wrap）。所以 `_MainTex` 配 `sampler_MainTex`。

### 4.2 兼容性

DX11 风格语法在部分老平台（GLES2）不可用。要么把 shader 限定到 `#pragma target 3.5` 以上跳过老设备，要么用 Unity 的宏做兼容层：

```hlsl
TEXTURE2D(_MainTex);
SAMPLER(sampler_MainTex);
// 采样
half4 c = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv);
half4 l = SAMPLE_TEXTURE2D_LOD(_MainTex, sampler_MainTex, uv, 2.0);
```

这些宏在 GLES3/Vulkan/Metal 上展开为 `Texture2D` + `SamplerState`，在 GLES2 上退化为 `sampler2D`。

### 4.3 OpenGL 的纹理单元

- 可用纹理单元数与显卡相关，可用 `glGetIntegerv(GL_MAX_TEXTURE_IMAGE_UNITS, ...)` 查询；常见是 32 个。
- 纹理单元是**状态机里的插槽**：先把纹理绑定到某个 unit，再把 unit 号告诉 shader 的 sampler uniform。
- 所有 shader program 共享同一组插槽，是**分时占用**，不是每个 shader 各有一份。
- Bindless（`GL_ARB_bindless_texture` / D3D12 descriptor heap）可以绕开这个上限，把纹理句柄直接存进缓冲。

```cpp
glActiveTexture(GL_TEXTURE0 + unit);
glBindTexture(GL_TEXTURE_2D, tex);
glUniform1i(glGetUniformLocation(prog, "uTex"), unit);
```

## 五、Instancing

### 5.1 GPU Instancing 的 shader 写法（Unity）

一次 draw call 画同一个 mesh 的多个实例，逐实例数据（矩阵、颜色、自定义属性）放在 instanced buffer 里。Unity 的写法：

```hlsl
#pragma multi_compile_instancing
#pragma instancing_options assumeuniformscaling

UNITY_INSTANCING_BUFFER_START(Props)
    UNITY_DEFINE_INSTANCED_PROP(float4, _Color)
UNITY_INSTANCING_BUFFER_END(Props)

struct appdata
{
    float4 vertex : POSITION;
    float3 normal : NORMAL;
    float2 uv     : TEXCOORD0;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

struct v2f
{
    float4 pos    : SV_POSITION;
    float2 uv     : TEXCOORD0;
    float4 color  : COLOR;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

v2f vert (appdata v)
{
    v2f o;
    UNITY_SETUP_INSTANCE_ID(v);             // 必须在最前面
    o.pos = UnityObjectToClipPos(v.vertex);
    o.uv  = v.uv;
    o.color = UNITY_ACCESS_INSTANCED_PROP(Props, _Color);
    return o;
}

half4 frag (v2f i) : SV_Target
{
    UNITY_SETUP_INSTANCE_ID(i);             // 片元里要用 instance id 时也要
    return tex2D(_MainTex, i.uv) * i.color;
}
```

要点：

- `UNITY_VERTEX_INPUT_INSTANCE_ID` 在 appdata 与 v2f 里都要有，才能把 id 传到片元。
- `UNITY_SETUP_INSTANCE_ID` 必须在读取任何 instanced 属性之前调用。
- instanced 属性不能放在 `UnityPerMaterial` 的普通 CBUFFER 里（那样所有实例共享一份），要用 `UNITY_INSTANCING_BUFFER`。
- 引擎侧还要开启材质的 `enableInstancing`，且实例之间**材质相同**才合批。
- 与 SRP Batcher 互斥：开了 SRP Batcher 时，普通（非 instanced）材质走另一条路径。

### 5.2 DOTS Instancing

DOTS/Hybrid Renderer 把每个实例的属性放进一块大的 GPU buffer，shader 通过 instance id 取。与「GPU Instancing」的区别是：数据布局由 ECS 决定，实例数量可达数十万，且不需要 per-instance uniform 数组。

```hlsl
#pragma target 4.5
#pragma multi_compile_instancing
#pragma multi_compile _ DOTS_INSTANCING_ON

struct appdata
{
    float3 positionOS : POSITION;
    float4 uv0        : TEXCOORD0;
    #if UNITY_ANY_INSTANCING_ENABLED
    uint instanceID : INSTANCEID_SEMANTIC;
    #endif
};

struct v2f
{
    float4 positionCS : SV_POSITION;
    float2 uv0        : TEXCOORD0;
    #if UNITY_ANY_INSTANCING_ENABLED
    uint instanceID : CUSTOM_INSTANCE_ID;
    #endif
};

CBUFFER_START(UnityPerMaterial)
    float4 _Color;
    float4 _MainTex_ST;
CBUFFER_END

#if defined(UNITY_DOTS_INSTANCING_ENABLED)
    UNITY_DOTS_INSTANCING_START(MaterialPropertyMetadata)
        UNITY_DOTS_INSTANCED_PROP(float4, _Color)
        UNITY_DOTS_INSTANCED_PROP(float4, _MainTex_ST)
    UNITY_DOTS_INSTANCING_END(MaterialPropertyMetadata)

    #define _Color      UNITY_ACCESS_DOTS_INSTANCED_PROP_WITH_DEFAULT(float4, _Color)
    #define _MainTex_ST UNITY_ACCESS_DOTS_INSTANCED_PROP_WITH_DEFAULT(float4, _MainTex_ST)
#endif

sampler2D _MainTex;

v2f vert (appdata v)
{
    v2f o = (v2f)0;
    UNITY_SETUP_INSTANCE_ID(v);
    UNITY_TRANSFER_INSTANCE_ID(v, o);

    float3 positionWS = TransformObjectToWorld(v.positionOS);
    o.positionCS = TransformWorldToHClip(positionWS);
    o.uv0 = v.uv0.xy * _MainTex_ST.xy + _MainTex_ST.zw;

    #if UNITY_ANY_INSTANCING_ENABLED
    o.instanceID = v.instanceID;
    #endif
    return o;
}

half4 frag (v2f i) : SV_Target
{
    UNITY_SETUP_INSTANCE_ID(i);
    return tex2D(_MainTex, i.uv0.xy) * _Color;
}
```

几个关键宏：

| 宏 | 作用 |
| --- | --- |
| `UNITY_DOTS_INSTANCING_START/END(Name)` | 声明一组逐实例属性块 |
| `UNITY_DOTS_INSTANCED_PROP(type, name)` | 声明某个属性是逐实例的 |
| `UNITY_ACCESS_DOTS_INSTANCED_PROP_WITH_DEFAULT(type, name)` | 读取；未启用 DOTS 时回落到常量 |
| `UNITY_SETUP_INSTANCE_ID` / `UNITY_TRANSFER_INSTANCE_ID` | 解析并传递 instance id |

**ShaderGraph**：项目里定义了 `ENABLE_HYBRID_RENDERER_V2` 宏后，ShaderGraph 生成的 shader 会自动带 DOTS 变体，材质面板不需要额外勾选；Hybrid Renderer 渲染时自动设置对应 keyword。

### 5.3 三种方式对比

| | 普通合批 | GPU Instancing | DOTS Instancing |
| --- | --- | --- | --- |
| 数据来源 | 每 draw 的 CBUFFER | instanced buffer / uniform 数组 | ECS 的大块 GPU buffer |
| 典型实例数 | 少 | 几十 ~ 几千 | 万 ~ 百万 |
| 逐实例属性 | 不支持 | `UNITY_DEFINE_INSTANCED_PROP` | `UNITY_DOTS_INSTANCED_PROP` |
| shader 关键宏 | — | `UNITY_VERTEX_INPUT_INSTANCE_ID` | 额外 `DOTS_INSTANCING_ON` 变体 |

## 六、Direct3D 11 资源绑定

D3D11 用 **Model-View** 模式描述资源：

- **Resource**（`ID3D11Resource`）= Model：负责数据本身、创建/销毁、内存布局。
- **View** = 对资源的一种「用法/视图」，绑定到管线的是 View，不是 Resource。

| View | 含义 | 典型用途 |
| --- | --- | --- |
| `ID3D11ShaderResourceView`（SRV） | 只读资源视图 | 纹理、结构化缓冲输入 |
| 常量缓冲（`ID3D11Buffer`，绑定到 `b#`） | 只读常量缓冲；D3D11 没有 `ID3D11ConstantBufferView` 这个接口，CBV 是 D3D12 的概念 | 矩阵、材质参数 |
| `ID3D11RenderTargetView`（RTV） | 只写渲染目标视图 | 颜色附件 |
| `ID3D11DepthStencilView`（DSV） | 只写深度/模板视图 | 深度附件 |
| `ID3D11UnorderedAccessView`（UAV） | 可随机读写 | compute、原子操作 |

绑定槽位按类型分寄存器空间：

| 寄存器 | 资源 | 绑定 API |
| --- | --- | --- |
| `b0..b13` | CBV | `VSSetConstantBuffers` 等 |
| `t0..t127` | SRV | `PSSetShaderResources` |
| `s0..s15` | Sampler | `PSSetSamplers` |
| `u0..u7` | UAV | `CSSetUnorderedAccessViews` |

D3D12 用 Root Signature + Descriptor Heap 取代了这套「槽位 + 即时绑定」，但 shader 侧的 `register(t0)` 声明方式保留了下来，所以理解这套编号对读 Unity/UE 的 HLSL 仍然有用。

## 七、绘制线

在片元着色器里画线，本质是「构造一个 0/1 的掩码，再与背景混合」。坐标先归一化：

```glsl
// 原点在屏幕中心，纵向范围约 [-1, 1]
vec2 st = (gl_FragCoord.xy * 2.0 - u_resolution) / u_resolution.y;
```

### 7.1 三种基本画法（以水平线为例）

**① 条件判断**

```glsl
vec3 lineColor = vec3(1.0, 1.0, 0.0);
vec3 color = vec3(0.6);
if (st.y > 0.0 && st.y < 0.02) color = lineColor;
gl_FragColor = vec4(color, 1.0);
```

<img src="../../技术笔记/assets/WEBRESOURCE05cc024c77d8e304d6759910c599db79.png">

**② `step` 相减**

```glsl
float pct = step(0.0, st.y) - step(0.02, st.y);   // [0, 0.02) 内为 1
color = color * (1.0 - pct) + lineColor * pct;    // 先扣掉背景再叠加
// 等价于
color = mix(color, lineColor, pct);
```

直接相加会得到「背景 + 线色」的错误亮度，因为原笔记里 `color = color + lineColor * pct` 没有先扣除背景：

<img src="../../技术笔记/assets/WEBRESOURCEd61088a10413ed7924d867fb59f6e24b.png">

改成 `mix` 后与条件判断结果一致：

<img src="../../技术笔记/assets/WEBRESOURCE04a9cc1b7068585a17668d0f08e05162.png">

**③ `smoothstep` 相减**

```glsl
float pct = smoothstep(0.0, 0.01, st.y) - smoothstep(0.01, 0.02, st.y);
color = mix(color, lineColor, pct);

// 想要边缘更锐利：对结果取幂，加快收敛
pct = pow(smoothstep(0.0, 0.01, st.y), 10.0)
    - pow(smoothstep(0.01, 0.02, st.y), 10.0);
```

<img src="../../技术笔记/assets/WEBRESOURCE7ddadc2aa5c9330abec209794274a46a.png">

<img src="../../技术笔记/assets/WEBRESOURCE966672c52643941ad7122535400c9443.png">

原笔记说「smoothstep 在区间内返回线性插值」是**错的**：它返回的是 Hermite 平滑插值 `3t² - 2t³`，所以边缘是软的而非线性的；这也正是它能抗锯齿的原因。

### 7.2 垂直线

把 `st.y` 换成 `st.x` 即可：

```glsl
pct = step(0.1, st.x) - step(0.12, st.x);
color = mix(color, lineColor, pct);

pct = smoothstep(0.2, 0.21, st.x) - smoothstep(0.21, 0.22, st.x);
color = mix(color, lineColor, pct);
```

<img src="../../技术笔记/assets/WEBRESOURCEec918cf383f59dd325a6d79f22702311.png">

### 7.3 斜线

用 `y = kx + b` 构造「阈值随 x 变化」的掩码：

```glsl
// step 版：硬边，斜线有明显锯齿
float lineKx(vec2 st, float k, float t, float lineWidth)
{
    float y = k * st.x + t;
    return step(y, st.y) - step(y + lineWidth, st.y);
}

// smoothstep 版：软边 + pow 收紧
float lineKxSmooth(vec2 st, float k, float t, float lineWidth)
{
    float y = k * st.x + t;
    return pow(smoothstep(y - lineWidth, y, st.y)
             - smoothstep(y, y + lineWidth, st.y), 25.0);
}
```

<img src="../../技术笔记/assets/WEBRESOURCEc398284f8277b44d4da3dc76f2cc04ab.png">

<img src="../../技术笔记/assets/WEBRESOURCEe7e6ce99394653e1e6c93d807c9bd9f4.png">

注意 `step(y, st.y)` 的参数顺序是 `step(edge, x)`：这里 `edge = y`、`x = st.y`，表示「st.y 在直线之上」。顺序写反是这类代码最常见的 bug。

### 7.4 线段：用两点确定

只靠斜率无法限定端点，需要**投影**：

- 令 `A` 为起点、`B` 为终点，`P` 为屏幕任意点。
- 若 `dot(B - A, P - A) > 0` 且 `dot(A - B, P - B) > 0`，则 `P` 落在 A、B 之间的带状区域内（即投影参数 `t ∈ [0,1]`）。
- 点到直线的距离用叉积更稳（不必 `acos`）：`dist = |cross(B - A, P - A)| / |B - A|`。

```glsl
float lineSegment(vec2 st, vec2 a, vec2 b, float lineWidth)
{
    vec2 ab = b - a;
    vec2 ap = st - a;

    // 投影参数，落在 [0,1] 之外就丢弃
    float t = dot(ap, ab) / max(dot(ab, ab), 1e-6);
    if (t < 0.0 || t > 1.0) return 0.0;

    // 点到直线的垂直距离
    float dist = abs(ab.x * ap.y - ab.y * ap.x) / max(length(ab), 1e-6);

    return 1.0 - smoothstep(lineWidth * 0.5, lineWidth, dist);
}
```

原笔记的版本用 `acos(dot(u,v)/(|u||v|))` 再 `sin(angle)*|v|` 求距离，数学上等价于叉积，但多了两次 `length`、一次 `acos`、一次 `sin`，而且 `acos` 的输入可能因浮点误差超出 `[-1,1]` 产生 NaN。叉积版本更快更稳。

<img src="../../技术笔记/assets/WEBRESOURCE6b0a0297c4049b3e272ba4eae9dbd7d5.png">

<img src="../../技术笔记/assets/WEBRESOURCE9fe879326445ecea13f85c59cdd61e56.png">

原笔记中「点乘结果大于 0 小于 1 表示方向相同」的说法也不严谨：点乘结果是长度乘积乘余弦，与「1」没有可比性；判断同向只需 `> 0`。

### 7.5 曲线

把直线方程换成曲线方程，其余逻辑不变：

```glsl
float sinLine(vec2 st, float lineWidth)
{
    float amplitude       = 0.08;
    float angularVelocity = 12.0;
    float offset          = 0.5;
    float phase           = 0.5;
    // 让波形动起来：phase = 12.0 * u_time;

    float y = amplitude * sin(angularVelocity * st.x + phase) + offset;
    return smoothstep(y - lineWidth * 0.5, y, st.y)
         - smoothstep(y, y + lineWidth * 0.5, st.y);
}

float powLine(vec2 st, float lineWidth)
{
    float y = pow(st.x, 5.0);
    return smoothstep(y - lineWidth * 0.5, y, st.y)
         - smoothstep(y, y + lineWidth * 0.5, st.y);
}
```

<img src="../../技术笔记/assets/WEBRESOURCEad028cd798513c88245b6cec4c43cf89.png">

<img src="../../技术笔记/assets/WEBRESOURCE5aed7a37790e08278b8582fdff5def54.png">

圆环同理，用 `distance(st, center)` 作为阈值：

```glsl
float circleLine(vec2 st, vec2 center, float radius, float lineWidth)
{
    float d = distance(st, center);
    return smoothstep(radius - lineWidth, radius, d)
         - smoothstep(radius, radius + lineWidth, d);
}
```

### 7.6 线宽与抗锯齿

- 固定线宽在不同分辨率下粗细不同。用屏幕导数让线宽随像素密度自适应：

```glsl
float w = fwidth(st.y) * 1.5;
float pct = smoothstep(-w, w, st.y) - smoothstep(0.02 - w, 0.02 + w, st.y);
```

- `smoothstep` 的过渡带宽度就是「抗锯齿的模糊半径」，太宽会糊、太窄会重新出现锯齿。
- 用 `pow` 收紧时不要用奇数次幂处理可能为负的差值，否则符号会翻转。

## 八、Arrow animation

一个「进度箭头」的片元逻辑：竖直方向按区间裁剪，水平方向按带宽分内外两层颜色。

```hlsl
float4 ratio;       // x: 起点(0~1), y: 终点(0~1), z: 内层宽度(0~1)
float4 outerColor;
float4 innerColor;
float4 result;

void ArrowAnimation()
{
    // 只保留 uv.y ∈ [ratio.x, ratio.y] 的部分
    // 乘起来等于 1 才保留；减去 0.0001 让边界处变成负数而被 clip 掉
    clip(step(ratio.x, uv.y) * step(uv.y, ratio.y) - 0.0001);

    float halfWidth = saturate(ratio.z) * 0.5;

    // (uv.x - (0.5 - halfWidth)) * ((0.5 + halfWidth) - uv.x) > 0
    // ⟺ uv.x 落在 [0.5 - halfWidth, 0.5 + halfWidth] 内
    float mask = step(0.0,
        (uv.x - 0.5 + halfWidth) * (0.5 + halfWidth - uv.x));

    result = lerp(outerColor, innerColor, mask);
}
```

要点：

- `clip(x)` 在 `x < 0` 时丢弃片元（等价 GLSL 的 `if (x < 0) discard;`）。用 `step*step - 0.0001` 而不是直接 `clip(step*step)`，是因为边界处乘积恰好为 0，而 `clip` 只在**严格小于 0** 时丢弃。
- `halfWidth` 用 `saturate` 夹到 `[0,1]`，否则 `ratio.z > 1` 会让区间反转、整条带消失。
- 用「乘积 > 0」判断区间，比两次比较更省指令，也不会引入分支。
- 原笔记里 `clip` 写了两遍、外圈颜色分支被注释掉，属于调试残留；上面的版本是它的等价整理。

<img src="../../技术笔记/assets/WEBRESOURCE7775c1485f1a10f09b6becea12113a6c.png">

## 九、Flipbook node

Flipbook（翻转书）把一张 `width × height` 网格图当作序列帧播放：`width` 是**列数**、`height` 是**行数**，`tile` 是当前帧索引（从 0 开始，按行优先）。

UV 换算：

```hlsl
float2 FlipbookUV(float2 uv, float2 size /*width,height*/, float tile)
{
    float2 tileSize = 1.0 / size;
    float2 offset = float2(fmod(tile, size.x), floor(tile / size.x)) * tileSize;
    return uv * tileSize + offset;
}
```

注意：

- 行优先时 `tile` 增大是「从左到右、从上到下」。Unity Shader Graph 的 Flipbook 节点默认 y 从上往下，直接采样会让动画上下颠倒，常见修法是 `uv.y = 1.0 - uv.y` 或在 offset 里用 `(size.y - 1 - row)`。
- 每帧只应显示一格，UV 要严格限制在 `[offset, offset + tileSize]` 内；相邻帧会因 mip / 双线性过滤「串色」，解决办法是把图做成图集时留 padding，或在采样时用 `tex2Dlod(..., 0)` 禁用 mip。
- 时间驱动：`tile = floor(_Time.y * fps) % (width * height)`，取模保证循环。
- Shader Graph 的 Flipbook 节点还有一个 `Time` 输入与 `Invert Y` 选项，本质与上式相同。

<img src="../../技术笔记/assets/WEBRESOURCE20c18abbb3253a8388845ba071608dc4.gif">

动图内容（文字已转录，图保留）：Shader Graph 中的 `Flipbook` 节点，输入 `UV = UV0`、`Width = 2`、`Height = 2`，`Tile` 从 `0.03` 递增到 `17.6` 播放这张 2×2 图集；`Invert X`、`Invert Y` 均**未**勾选（即按默认 y 从上往下采样，画面上下颠倒）。左侧 `Sample Texture 2D` 的 UV 取自 `Flipbook` 的 `Out(2)`，右侧 `Sample Texture 2D` 直接接 `UV0` 作为对照。

## 十、Skybox panoramic：全景图转 cubemap

### 10.1 等距柱状（equirectangular）采样

全景图是「经度 × 纬度」展开的 2:1 贴图。给定视线方向 `dir`：

```hlsl
float2 DirToEquirectUV(float3 dir)
{
    dir = normalize(dir);
    float latitude  = acos(dir.y);                 // [0, π]，从上到下
    float longitude = atan2(dir.z, dir.x);         // [-π, π]
    float u = 0.5 - longitude * (0.5 / UNITY_PI);  // [0, 1]
    float v = 1.0 - latitude  * (1.0 / UNITY_PI);  // [0, 1]
    return float2(u, v);
}
```

推导：

- `dir.y = cos(theta)`，`theta` 是与 +Y 轴的夹角，`acos` 得到 `[0, π]`；除以 `π` 归一化，再用 `1 -` 翻转，使贴图底部对应「下」。
- `atan2(z, x)` 得到 `[-π, π]`；乘 `0.5/π` 得到 `[-0.5, 0.5]`；`0.5 - x` 映射到 `[0, 1]`。

> 原笔记把 `latitude` 直接说成「范围 [0,1]」是不准确的：`acos` 的结果是 `[0, π]`，要除以 `π` 之后才是 `[0,1]`。
>
> 另外 `0.5 - longitude/(2π)` 这个符号约定决定了全景图的「正面」朝向以及左右是否镜像。Unity 内置的 Skybox/Panoramic 用的就是这套约定；如果你用别的工具生成的全景图，可能需要改成 `0.5 + longitude/(2π)` 并翻转一次 u，否则接缝位置和左右方向会不对（不同工具的约定不一致，导入前最好先目视核对）。

完整 shader（简化自原笔记）：

```hlsl
Shader "Lookdev/Skybox Panoramic"
{
    Properties
    {
        _Tint ("Tint Color", Color) = (.5, .5, .5, .5)
        [Gamma] _Exposure ("Exposure", Range(0, 8)) = 1.0
        _Rotation ("Rotation", Range(0, 360)) = 0
        [NoScaleOffset] _MainTex ("Spherical (HDR)", 2D) = "grey" {}
    }
    SubShader
    {
        Tags { "Queue"="Background" "RenderType"="Background" "PreviewType"="Skybox" }
        Cull Off ZWrite Off

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma target 2.0
            #include "UnityCG.cginc"

            sampler2D _MainTex;
            half4 _MainTex_HDR;
            half4 _Tint;
            half _Exposure;
            float _Rotation;

            float3 RotateAroundYInDegrees(float3 v, float degrees)
            {
                float alpha = degrees * UNITY_PI / 180.0;
                float sina, cosa;
                sincos(alpha, sina, cosa);
                float2x2 m = float2x2(cosa, -sina, sina, cosa);
                return float3(mul(m, v.xz), v.y).xzy;
            }

            struct appdata_t { float4 vertex : POSITION; UNITY_VERTEX_INPUT_INSTANCE_ID };
            struct v2f
            {
                float4 vertex : SV_POSITION;
                float3 texcoord : TEXCOORD0;
                UNITY_VERTEX_OUTPUT_STEREO
            };

            v2f vert (appdata_t v)
            {
                v2f o;
                UNITY_SETUP_INSTANCE_ID(v);
                UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
                float3 rotated = RotateAroundYInDegrees(v.vertex.xyz, _Rotation);
                o.vertex = UnityObjectToClipPos(rotated);
                o.texcoord = v.vertex.xyz;   // 方向向量，不是 UV
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                float2 tc = DirToEquirectUV(i.texcoord);
                half4 tex = tex2D(_MainTex, tc);
                half3 c = DecodeHDR(tex, _MainTex_HDR);
                c = c * _Tint.rgb * _Exposure;
                return half4(c, 1);
            }
            ENDCG
        }
    }
}
```

要点：

- 天空盒的顶点着色器把模型空间位置直接当方向用（`o.texcoord = v.vertex.xyz`），因为天空盒是一个以相机为中心的立方体，顶点方向即视线方向。
- `Cull Off ZWrite Off` + `Queue = Background`：不写深度、不剔除，保证永远在最底层。
- `DecodeHDR` 处理 RGBM 编码的 HDR 全景图；普通 LDR 贴图应去掉。
- 接缝在 `u = 0/1` 处，双线性过滤 + mip 会在接缝出现一条亮/暗线；解决方式是贴图导入时开 `Wrap Mode = Repeat`、或对 u 做一次 wrap 后手动采样相邻像素。

### 10.2 全景图转 cubemap

两种做法：

**① 导入时让引擎转**（Unity）：把 Texture Shape 设为 `Cube`，Mapping 选 `Latitude-Longitude (Legacy)` 或 `Panoramic`，Unity 会生成 6 面 cubemap。优点是采样时用 `texCUBE` 无接缝、mip 正确；缺点是 cubemap 分辨率固定，且会多占内存。

**② 自己渲染 6 面**：对每个面用 90° FOV 的相机渲染一次，或在一个 fullscreen Pass 里按面索引算方向：

```hlsl
float3 DirFromCubeFace(uint face, float2 uv)  // uv ∈ [-1, 1]
{
    float u = uv.x, v = uv.y;
    if (face == 0) return float3( 1.0,   -v,   -u);  // +X
    if (face == 1) return float3(-1.0,   -v,    u);  // -X
    if (face == 2) return float3(   u,  1.0,    v);  // +Y
    if (face == 3) return float3(   u, -1.0,   -v);  // -Y
    if (face == 4) return float3(   u,   -v,  1.0);  // +Z
    return              float3(  -u,   -v, -1.0);    // -Z
}

// 采样等距柱状全景图
float2 uvEq = DirToEquirectUV(normalize(DirFromCubeFace(face, uv)));
half3 c = tex2D(_MainTex, uvEq).rgb;
```

方向向量的 `±v` 符号决定每个面的朝向；写反会导致某个面上下颠倒或左右镜像。调试时先渲染 ±X 两面确认朝向，再补其余四面。

## 十一、smoothstep 的工程用法

### 11.1 定义与边界（再强调一次）

```hlsl
float smoothstep(float e0, float e1, float x)
{
    float t = saturate((x - e0) / (e1 - e0));   // HLSL 的 saturate = clamp(0,1)
    return t * t * (3.0 - 2.0 * t);
}
```

- `x <= e0` → 0；`x >= e1` → 1（端点归 0/1，不是开区间）。
- `e0 >= e1` 未定义；要反向渐变请写 `1.0 - smoothstep(lo, hi, x)`。
- `e0 == e1` 会除零。

<img src="../../技术笔记/assets/WEBRESOURCE41a35114960902f3bd90ed0da922e7e7.png" width="581" height="621">

### 11.2 常用模式

**软边圆点 / 光斑**

```hlsl
float d = length(uv - 0.5);
float mask = 1.0 - smoothstep(0.2, 0.5, d);   // 内 1 外 0
```

<img src="../../技术笔记/assets/WEBRESOURCEd00236b5e5236795bd50c3c303675f8d.png">

**圆环 / 描边**：两个 smoothstep 相减

```hlsl
float s1 = smoothstep(0.2, 0.3, d);
float s2 = smoothstep(0.3, 0.4, d);
float ring = s1 - s2;      // 只保留 [0.2, 0.4] 的环带
```

<img src="../../技术笔记/assets/WEBRESOURCE0e432f424c07306f62f6559c4b09e3af.png">

**程序化扫描线**（替代一张条纹贴图，省一次采样）

```hlsl
void ApplyScanline(inout half4 color, half3 positionWS)
{
    half posY = positionWS.y + _Time.y * _ScanlineSpeed;
    half uv = frac(posY / _ScanlineGap);   // 以 Gap 为周期重复

    half weight = smoothstep(0.10, 0.15, uv) - smoothstep(0.25, 0.30, uv)
                + smoothstep(0.40, 0.50, uv) - smoothstep(0.80, 0.90, uv);

    color.rgb += _ScanlineTint.rgb * weight * _ScanlineTint.a;
}
```

<img src="../../技术笔记/assets/WEBRESOURCE2c8209ce02243a22de77a35a9891990e.png">

<img src="../../技术笔记/assets/WEBRESOURCE12a56a10e620f350698e47334aae87d8.png">

**做法对比**：左边是程序生成，右边是纹理采样。程序化版本没有采样、没有 mip、分辨率无关，但参数调节不如美术画贴图直观。

**抗锯齿**：把过渡带宽度绑定到屏幕导数

```hlsl
float w = fwidth(x);
float mask = smoothstep(edge - w, edge + w, x);
```

**锐化**：`pow(smoothstep(...), n)`，`n` 越大边缘越硬（见第七节）。

### 11.3 常见误用

- 把 `smoothstep` 当 `lerp` 用：它的输出是 `[0,1]` 的权重，不是任意插值；要插值请 `lerp(a, b, smoothstep(...))`。
- 参数顺序写成 `smoothstep(x, e0, e1)`：三个参数都是 float，编译器不会报错，结果全错。
- 用反序 `smoothstep(hi, lo, x)` 求「内亮外暗」：多数平台能跑，但规范上是未定义行为，跨平台（尤其 Metal）可能不一致。
- 在 `smoothstep` 里做除法/开方，输入为 NaN 时结果静默变黑，排查困难。

## 十二、常见性能陷阱

**采样与带宽**

- **依赖采样（dependent texture read）**：用一次采样的结果去算下一次采样的 UV，会打断纹理流水线，移动端尤其贵。
- 采样次数过多：能打包进一张图的通道就打包；能用 `textureGather` 一次取 4 个 texel 就不要采 4 次。
- 未压缩纹理 / 过大的 mip 链：带宽是移动端第一瓶颈。
- 在循环里采样：先确认是否真有必要（SDF、体积雾等），否则改为预计算贴图。

**分支与占用率**

- 逐像素的 divergent branch：warp 内两条路都走，等于代价相加。
- 动态分支里用隐式 LOD 采样：编译报错或结果错误（见第二节）。
- `discard` / Alpha Test：破坏 early-Z，透明物体会导致全屏 overdraw。
- 过度 `unroll`：指令缓存Miss、寄存器溢出（spill），反而变慢。

**精度**

- `mediump`（fp16）范围只有 ±2^14 ≈ 16384。用 `_Time.y` 直接累加做 UV 偏移，约 4.5 小时后溢出，画面开始跳变；应使用 `frac()` 定期回绕，或用 `highp`。
- 大世界坐标用 `mediump` 会抖动，位置一律 `highp`（或相机相对坐标）。
- `pow(x, y)` 在 `x < 0` 时未定义；`normalize(0)`、`acos(1.0000001)` 产生 NaN。

**指令**

- `pow`、`sin`、`cos`、`atan` 都是多周期指令，能查表/近似就用近似。
- `normalize` 内部是 `rsqrt`，同一向量被多次 `normalize` 是浪费。
- 整数除法/取模在部分 GPU 上很慢，能用位运算就用位运算（`2 的幂` 场景）。
- 矩阵乘法注意顺序，`mul(M, v)` 与 `mul(v, M)` 的寄存器占用不同。

**工程层面**

- 变体爆炸：`multi_compile` 每多一个 keyword 就翻一倍。
- `#pragma target` 定得过高会把老设备直接排除；定得过低则用不了 texture/sampler 分离等特性。
- uniform 数组过大：常量缓冲有大小上限（D3D11 单 CB 64KB），超了会静默截断或绑定失败。
- 每帧 `SetPass` 次数、材质属性块（CBUFFER）未按 SRP Batcher 规范声明，会导致无法合批。

## 原始笔记索引

- `Shader/Shader Model.md`
- `Shader/branch flatten unroll loop.md`
- `Shader/充分利用GPU的并行性.md`
- `Shader/Separate textures and samplers.md`
- `Shader/Dots Instancing Shader.md`
- `Shader/Direct3D11资源绑定.md`
- `Shader/绘制线.md`
- `Shader/arrow animation.md`
- `Shader/Flipbook node.md`
- `Shader/skybox panoramic.md`
- `Shader/smoothstep.md`
