# Unity Shader 与材质

> 本文整合 ShaderLab 写法、CBuffer/ShaderLibrary 约定、PBR 金属流、法线贴图、深度获取、浮点编码、变体与编译优化、GPU Instancing、Compute Shader 与 Cubemap 采样等内容。

## 1. ShaderLab 结构

一个 Unity Shader 由若干层嵌套的代码块组成，外层是 ShaderLab 声明，内层才是 HLSL。

```hlsl
Shader "Custom/Demo"
{
    Properties
    {
        _MainTex ("Albedo", 2D) = "white" {}
        _Color   ("Color", Color) = (1,1,1,1)
        _Gloss   ("Smoothness", Range(0,1)) = 0.5
        [Toggle] _UseDetail ("Use Detail", Float) = 0
    }

    SubShader
    {
        Tags { "RenderType"="Opaque" "Queue"="Geometry" }

        Pass
        {
            Name "Forward"
            Tags { "LightMode"="UniversalForward" }

            Cull Back
            ZWrite On
            ZTest LEqual
            Blend Off

            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"

            struct Attributes
            {
                float4 positionOS : POSITION;
                float2 uv         : TEXCOORD0;
            };

            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float2 uv         : TEXCOORD0;
            };

            Varyings vert(Attributes input)
            {
                Varyings output;
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv = input.uv;
                return output;
            }

            half4 frag(Varyings input) : SV_Target
            {
                return half4(1, 1, 1, 1);
            }
            ENDHLSL
        }
    }
    Fallback "Hidden/Shader Graph/FallbackError"
}
```

要点：

- `Properties` 只声明材质面板上的可调参数，**不代表变量一定在材质级别生效**（见第 2 节 `SetGlobalFloat`）。
- `SubShader` 可以有多份，Unity 按当前平台能力自上而下选择第一份可用的。
- `Pass` 内部的 `Tags { "LightMode"=... }` 决定该 Pass 被 SRP 在哪个阶段调用。
- 渲染状态（`Cull`/`ZWrite`/`ZTest`/`Blend`/`Stencil`）属于 Pass 级设置，会被该 Pass 覆盖。

## 2. Properties 的 Decorator Attribute

ShaderLab 的 `Properties` 支持在变量前加 `[...]` 形式的 Attribute。它分为两类：

| 类型 | 说明 |
|---|---|
| Drawer | 必须与**一条** property 一一对应，负责绘制该 property 的控件 |
| Decorator | 可以实现**多条对应一条** property（甚至不对应 property），只负责额外界面元素 |

关键区别在于：Decorator 不改变 property 的存储与取值，只在 Inspector 上追加显示内容，并且可以带条件显示。

```csharp
[Warning(ShowIf(DIELECTRIC and USE_EMISSION and _ALPHABLEND_ON), When Non Metallic texture Mode is on you have to choose JUST ONE of Emission or Cutout or Blend. Please close Alpha Blend or Emission. If both texture channels are required then you must turn off Non Metallic Mode)]
[RenderingMode] _Mode("Blend Mode", Int) = 0
```

自定义 Decorator 需要继承 `MaterialPropertyDrawer`：

```csharp
public class WarningDecorator : MaterialPropertyDrawer
{
    private readonly string text;
    private readonly ShowIf condition;

    public WarningDecorator(params string[] paramList)
    {
        try
        {
            var terms = ShaderPropertyDrawerParser.Parse(paramList);

            foreach (var item in terms)
            {
                if (item is ShowIf)
                    condition = item as ShowIf;
                else if (item is string)
                    text = item as string;
                else
                    throw new NotImplementedException($"{GetType().Name}不支持 {item.GetType().Name} 类型的参数");
            }

            if (text.Length == 0)
                throw new Exception("Warning需要提供详细的说明");
        }
        catch (Exception e)
        {
            Debug.LogError(e);
            throw;
        }
    }

    public override void OnGUI(Rect position, MaterialProperty prop, GUIContent label, MaterialEditor editor)
    {
        if (condition.IsValid(editor))
        {
            var r = EditorGUILayout.GetControlRect(true, 46);
            EditorGUI.HelpBox(r, text, MessageType.Warning);
        }
    }

    public override float GetPropertyHeight(MaterialProperty prop, string label, MaterialEditor editor)
    {
        return -2; // 负数表示跟随上一个控件，具体含义见 MaterialPropertyDrawer 文档
    }
}
```

**踩坑结论：自定义 Decorator 的类名必须以 `Decorator` 结尾，不能以 `Drawer` 结尾**，否则 Unity 会按 Drawer 的规则去匹配 property，导致绑定失败或控件不显示。

## 3. 全局变量与 Shader.SetGlobal*

`Shader.SetGlobalFloat/Vector/Texture` 写了却"不生效"，绝大多数原因是 property 仍然声明在 `Properties` 里。

- `Properties` 中的参数是**材质局部**的，每个材质各有一份，Shader 里读的是材质 CBUFFER 中的值。
- 需要全局设置的变量，必须**从 `Properties` 中删掉**，然后在脚本里用 `Shader.SetGlobalFloat/Vector/Texture` 赋值；此时 Shader 里直接声明同名 uniform 即可（放在 `$Global` CBUFFER 中）。

```csharp
Shader.SetGlobalFloat("_GlobalIntensity", 0.5f);
```

```hlsl
// 不再出现在 Properties 中
float _GlobalIntensity;
```

补充（待核实）：若 property 仍保留在 `Properties` 中，材质的序列化值会覆盖全局值，表现为 `SetGlobal` 完全无效；不同渲染管线对覆盖时机的处理略有差异。

## 4. Blend Mode 的三种形式

Unity 的 `Blend` 命令有三种书写形式：

```hlsl
Blend Off

Blend SrcFactor DstFactor

Blend SrcFactorRGB DstFactorRGB, SrcFactorA DstFactorA
```

1. `Blend Off`：关闭混合，源颜色直接覆盖目标。
2. `Blend SrcFactor DstFactor`：RGB 与 Alpha 使用同一组因子。
3. `Blend SrcFactorRGB DstFactorRGB, SrcFactorA DstFactorA`：RGB 与 Alpha 分别指定因子（逗号分隔）。

混合方程（默认 `BlendOp Add`）：

```text
Final = SrcColor * SrcFactor  <BlendOp>  DstColor * DstFactor
```

默认 `BlendOp Add` 时即为 `Src * SrcFactor + Dst * DstFactor`。**注意因子是乘在颜色上的，不是加在颜色上的**，常见的错误写法是把 `Blend One One` 当成"叠加亮度"，实际结果是源与目标直接相加且没有 Alpha 参与。

常用组合：

| 目的 | 写法 |
|---|---|
| 不透明 | `Blend Off` |
| 传统 Alpha 混合 | `Blend SrcAlpha OneMinusSrcAlpha` |
| 预乘 Alpha | `Blend One OneMinusSrcAlpha` |
| 叠加发光 | `Blend One One` |
| 正片叠底 | `Blend DstColor Zero` |

```csharp
public enum BlendMode
{
    Opaque,
    Cutout,
    Alpha
}
```

```csharp
switch (blendMode)
{
    case BlendMode.Opaque:
        material.SetOverrideTag("RenderType", "");
        material.SetInt("_SrcBlend", (int)UnityEngine.Rendering.BlendMode.One);
        material.SetInt("_DstBlend", (int)UnityEngine.Rendering.BlendMode.Zero);
        material.SetInt("_ZWrite", 1);
        material.DisableKeyword("_ALPHATEST_ON");
        material.DisableKeyword("_ALPHABLEND_ON");
        material.renderQueue = -1;
        break;

    case BlendMode.Cutout:
        material.SetOverrideTag("RenderType", "TransparentCutout");
        material.SetInt("_SrcBlend", (int)UnityEngine.Rendering.BlendMode.One);
        material.SetInt("_DstBlend", (int)UnityEngine.Rendering.BlendMode.Zero);
        material.SetInt("_ZWrite", 1);
        material.EnableKeyword("_ALPHATEST_ON");
        material.DisableKeyword("_ALPHABLEND_ON");
        material.renderQueue = (int)UnityEngine.Rendering.RenderQueue.AlphaTest;
        break;

    case BlendMode.Alpha:
        material.SetOverrideTag("RenderType", "Transparent");
        material.SetInt("_SrcBlend", (int)UnityEngine.Rendering.BlendMode.SrcAlpha);
        material.SetInt("_DstBlend", (int)UnityEngine.Rendering.BlendMode.OneMinusSrcAlpha);
        material.SetInt("_ZWrite", 0);
        material.DisableKeyword("_ALPHATEST_ON");
        material.EnableKeyword("_ALPHABLEND_ON");
        material.renderQueue = (int)UnityEngine.Rendering.RenderQueue.Transparent;
        break;
}
```

## 5. CBuffer

### 5.1 为什么 Unity 不提供 MVP，而是拆成 M 和 VP

两者的**更新频率不同**：

- `M`（ObjectToWorld）存入 `UnityPerDraw`，同一次绘制期间不变，可随每个物体更新。
- `VP`（ViewProjection）存入 `UnityPerFrame`，整帧不变，每帧只更新一次。

如果合成一个 MVP，每个物体都要重新上传整块矩阵，浪费带宽，也破坏了 SRP Batcher 所依赖的 CBUFFER 布局稳定性。

> 补充说明：Constant Buffer 并非所有图形 API/版本都支持，OpenGL ES 2.0 级别不支持 UBO，Unity 在旧平台上会退化为逐个 uniform 上传（待核实具体平台矩阵）。

### 5.2 常见的 CBuffer

- `UnityPerMaterial`：材质级参数
- `UnityPerCamera`：相机矩阵、时间等
- `UnityPerDraw`：每个物体的 M 矩阵、UnityObjectToWorld 相关
- `UnityPerFrame`：VP 等帧级数据
- `UnityLighting`：光源数据
- `UnityShadows`：阴影相关
- `$Global`：`Shader.SetGlobal*` 写入的全局数据

## 6. Common.hlsl 中的约定

`Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl` 开头有一段关于编码规范的注释，值得遵守。

### 6.1 空间与坐标系

- Unity 世界空间 **Y 轴向上、左手坐标系**。
- 世界空间到视图空间会变成**右手系**，因此变换矩阵的行列式为负。
- 反射探针采集 Cubemap 时，视图空间仍按 Cubemap 惯例保持左手系，行列式为正。
- 光照代码假设 **1 Unity unit = 1 米**，这对基于物理的光照单位与平方反比衰减非常关键，模型比例必须正确。

### 6.2 变量名后缀表示空间

| 后缀 | 含义 |
|---|---|
| `WS` | World Space |
| `RWS` | Camera-Relative World Space，已减去相机平移以提高精度 |
| `VS` | View Space |
| `OS` | Object Space |
| `CS` | Homogeneous Clip Space |
| `TS` | Tangent Space |
| `TXS` | Texture Space |

示例：`NormalWS`。

### 6.3 向量命名

- 常规向量用**大写字母**：`V`（视线）、`L`（光照）、`N`（法线）、`H`（半角）。方向一律从当前像素指向外部，可直接用于光照方程。
- 大写默认**已归一化**；未归一化的向量加 `un` 前缀，如 `unL`。
- 向量默认归一化是"几乎处处成立"的约定，不是编译器保证，自己写的函数仍需显式归一化。

### 6.4 结构体与入口命名

```hlsl
struct AttributesDefault   // 顶点着色器输入
struct VaryingsDefault     // 顶点着色器输出
```

- 结构体使用 PascalCase，按入口类型加前缀。
- 使用时变量名统一叫 `input` / `output`。
- 入口函数名：`VertDefault`、`FragDefault` / `FragForward` / `FragDeferred`。

### 6.5 其他硬性约定

- 浮点常量写作 `1.0`，不要写 `1`、`1.0f`、`1.0h`。
- uniform 变量使用下划线前缀 + 首字母大写：`_LowercaseThenCamelCase`。
- 禁用 `in` 修饰符，只用 `out` / `inout`；也不要写 `inline`（无意义）。
- 函数的 `out` 参数必须放在参数列表最后。
- ShaderLibrary 的头文件**不要** include `Common.hlsl`，应由使用它的 `.shader` 或 `Material.hlsl` 负责包含。
- **所有 uniform 必须放进 constant buffer**，不能放在全局命名空间。原因：Compute Shader 要求各 kernel 的 CB 布局一致，而全局 namespace 下的 uniform 未被使用时会被优化掉，导致每个 kernel 的布局不同。
- C# 与 HLSL 共享的结构体必须按 `float4` 对齐并手动 padding。
- CB 中的数组一律用 `float4` 承载，避免打包（packing）问题，尤其要保证 Compute Shader 与 Pixel Shader 一致；即**不要用 `SetGlobalFloatArray` / `SetComputeFloatParams`**。
- 数组可以在 HLSL 中做别名（alias）拆分：

```hlsl
uniform float4 packedArray[3];
static float unpackedArray[12] = (float[12])packedArray;
```

- ShaderLibrary 中的函数必须是**无状态**的，内部不声明 uniform。
- 需要显式精度时用 `float` 或 `half`；两者都支持时用 `real`；同时需要两种精度版本时必须各写一份。

## 7. 纹理采样宏：TEXTURECUBE_PARAM

ShaderLab/HLSL 中，纹理与采样器是分开声明的。下面这组宏把"纹理 + 采样器"打包成一个参数，便于在函数间传递：

```hlsl
#define TEXTURE2D_PARAM(textureName, samplerName)        TEXTURE2D(textureName),       SAMPLER(samplerName)
#define TEXTURE2D_ARRAY_PARAM(textureName, samplerName)  TEXTURE2D_ARRAY(textureName), SAMPLER(samplerName)
#define TEXTURECUBE_PARAM(textureName, samplerName)      TEXTURECUBE(textureName),     SAMPLER(samplerName)
```

使用方式：

```hlsl
TEXTURECUBE(_CubeMap);
SAMPLER(sampler_CubeMap);

half4 SampleCube(TEXTURECUBE_PARAM(tex, sampler_tex), float3 dir)
{
    return SAMPLE_TEXTURECUBE(tex, sampler_tex, dir);
}

// 调用
half4 c = SampleCube(TEXTURECUBE_PARAM(_CubeMap, sampler_CubeMap), normalWS);
```

宏展开后就是两个逗号分隔的实参，正好对应函数形参中的"纹理, 采样器"。在 URP/HDRP 中应优先使用 `SAMPLE_TEXTURECUBE` / `SAMPLE_TEXTURECUBE_LOD` 等宏，以便适配不同平台的采样器绑定方式。

## 8. Unity PBR 金属流

### 8.1 Metallic Workflow 的参数含义

Unity 标准材质采用**金属流（Metallic Workflow）**，核心参数：

| 参数 | 通道 | 含义 |
|---|---|---|
| Albedo | RGB | 基础色。金属材质此处存的是**反射色（F0）**，非金属存的是漫反射色 |
| Metallic | 单通道 | 0 = 电介质，1 = 金属，中间值仅用于过渡/脏污 |
| Smoothness | 单通道 | 平滑度，与 Roughness 互为补数：`roughness = 1 - smoothness` |
| Normal | RGB | 切线空间法线 |
| Height | 单通道 | 视差/位移用 |
| Occlusion | 单通道 | AO |
| Emission | RGB | 自发光 |

对金属流，最终反照率拆分为：

```text
diffuseColor  = albedo * (1 - metallic)
specularColor = lerp(0.04, albedo, metallic)
```

即：金属没有漫反射，电介质的 F0 固定约 0.04（4%）。这是"接口简单直观、不是奇怪的经验参数"的来源。

PBR 的优点：

1. 物理真实；
2. 材质与光照解耦；
3. 接口简单直观，不是奇怪的经验参数；
4. 容易解决问题与扩展功能。

<img src="../../技术笔记/assets/FE670818219F4901B018014BD98B682C.jpg" width="620" height="159">

### 8.2 间接光照的两条路径

- **间接漫反射**：球谐系数（SH）。
- **间接镜面反射**：UE4 的 split sum approximation，Unity 中使用 `EnvBRDFApprox`：

```hlsl
inline half3 EnvBRDFApprox(half3 SpecularColor, half Roughness, half NoV);
```

### 8.3 Unity 中的 SH：Sky 与 Probe

Unity 的球谐光照分为两类：

- **Sky**：`Environment Lighting` 提供的实时全局 SH，额外叠加全局 SH Light 补光（如 `ArkFeatureSHAdditionalLight`）。
- **Probe**：标准 Unity SH，可接收本地 LightProbe 烘焙结果，通过 `ark_hasLightProbe` 之类的开关判断 probe 是否存在。

`Lighting` 窗口的 `Environment Lighting` 中，`Source` 有三项：

- `Gradient`：三色天光；
- `Skybox`：以天空盒作为光源；
- `Color`：纯色光源。

### 8.4 烘焙产出的资源

点击 `Lighting / Environment -> Generate Lighting` 会对场景中 static 的物体做光照烘焙，产出：

- 每个 `ReflectionProbe` 一张 HDR Cubemap（`ReflectionProbe-0`、`ReflectionProbe-1` …）；
- Skybox 对应的 ReflectionProbe 通常是编号最后一个，如 `ReflectionProbe-10`；
- Lightmap 贴图。

动态模型进入 ReflectionProbe 范围后会用该 Cubemap 作为光源；处于多个探针之间时做插值；都不在范围内则使用默认 skybox。

### 8.5 法线贴图的压缩格式 DXT5nm

非移动平台默认对法线贴图使用 **DXT5nm**：用 A 通道存 x 分量、G 通道存 y 分量，因为这两个通道位数最多（RGBA 分别为 5/6/5/8 位）。z 分量由单位向量关系重建。

```hlsl
real3 UnpackNormalAG(real4 packedNormal, real scale = 1.0)
{
    real3 normal;
    normal.xy = packedNormal.ag * 2.0 - 1.0;
    normal.z = max(1.0e-16, sqrt(1.0 - saturate(dot(normal.xy, normal.xy))));

    // z 重建之后再缩放，保证与 UnpackNormalRGB 行为一致。
    // 此时 normal 不是单位长度，但经过 TBN 变换后还会归一化，因此不需要在这里处理。
    normal.xy *= scale;
    return normal;
}
```

移动平台通常使用 ASTC/ETC2，Unity 会按平台选择对应的 `UnpackNormal` 分支。

## 9. 法线贴图：模型空间 vs 切线空间

法线贴图可以存**模型空间**或**切线空间**的法线。

### 9.1 模型空间法线

优点：

- 实现简单，不需要构造切线空间变换矩阵，只需用模型矩阵把法线变到世界空间，或把光线变到模型空间。
- 边界平滑：同一坐标系下存储，二次插值能得到平滑法线；切线空间法线在边缘/尖角处对多张不同切线空间贴图插值会出现缝隙。

缺点：不能复用、不支持 UV 动画、不能只存两个分量、骨骼蒙皮后法线不准确。

### 9.2 切线空间法线

- **适合骨骼动画**：存的是相对切线空间的方向，蒙皮后仍能得到正确法线；模型空间法线则会随网格变形而失效。
- **复用度高**：z 恒为正，复用到别的模型上不会出现离谱的翻转；一块砖的六个面可共用同一张切线空间法线图。
- **支持 UV 动画**：弯曲水流、岩浆可用 UV 偏移做扰动。模型空间法线做偏移一定错——例如对胶囊体偏移，背面的法线会跑到正面。
- **可压缩**：z 恒正，只存 xy，z 用 `sqrt(1 - x² - y²)` 重建。

> 原文此处写作 `√(x2+y2)`，正确关系应为 `z = sqrt(1 - x² - y²)`（单位向量约束），否则 z 会随 xy 增大而增大。

### 9.3 OpenGL 与 DirectX 的 UV 差异

- OpenGL：U 向右，V **向上**增长（+y），纹理原点在左下角。
- DirectX：U 向右，V **向下**增长（-y），纹理原点在左上角。

Unity 的纹理坐标遵循 OpenGL 标准。因此在 Windows/DirectX 下叉乘得到的副切线 `B'` 方向会反过来。Unity 因此把手性信息存进 `tangent.w`，在正交化得到 `T'` 时按平台写入手性值：

```cpp
// ImportMeshUtil.cpp -> OrthogonalizeTangent
// normalf, tangentf, binormalf 分别是 N、T'、B'
float dp = Dot(Cross(normalf, tangentf), binormalf);
// 比较 N×T' 与 B' 是否同向；不同向说明是 DirectX 平台
if (dp > 0.0F)
    outputTangent.w = 1.0F;
else
    outputTangent.w = -1.0F;
```

$$
\mathbf{B}' = T'.w\,(\mathbf{N}\times\mathbf{T}')
$$

这也是为什么手写切线空间变换时，副切线要写成 `cross(normalWS, tangentWS.xyz) * tangentWS.w`，漏掉 `w` 会在部分平台出现法线翻转。

### 9.4 法线从模型空间到世界空间

```hlsl
float3 TransformObjectToWorldNormal(float3 normalOS, bool doNormalize = true)
{
#ifdef UNITY_ASSUME_UNIFORM_SCALING
    return TransformObjectToWorldDir(normalOS, doNormalize); // 等比缩放，直接用 M
#else
    // 非等比缩放必须用逆转置矩阵；这里交换相乘顺序实现转置效果
    float3 normalWS = mul(normalOS, (float3x3)GetWorldToObjectMatrix());
    if (doNormalize)
        return SafeNormalize(normalWS);
    return normalWS;
#endif
}
```

> 注意：严格来说法线应使用 `(M^-1)^T`。上面的写法用 `mul(normalOS, M^-1)` 得到的是 `M^-1 * normalOS` 的转置形式，在 HLSL 行向量约定下等价，但仅在矩阵为仿射变换且忽略平移时成立。

## 10. 片段着色器中获取深度值

"片段的深度值"有两种来源：当前片段的深度、采样 pre-z 深度图得到的深度。当前片段深度有三种获取方式。

### 10.1 用 `SV_POSITION`

`SV_POSITION` 传入片段着色器前已被插值并做透视除法，OpenGL 平台还会执行 `glDepthRange(0,1)`，其 z 就是该片段的非线性深度（深度缓冲值）。

```hlsl
float depth = input.positionCS.z;
depth = Linear01Depth(depth, _ZBufferParams);
```

### 10.2 用 varying 存裁剪空间的 zw，在 fs 手动透视除法

```hlsl
// vertex
output.screenPos = ComputeScreenPos(output.positionCS); // 返回值范围 [0, w]

// fragment
float depth = input.screenPos.z / input.screenPos.w; // 透视除法 -> NDC
depth = Linear01Depth(depth, _ZBufferParams);
```

注意这里得到的是 **NDC 深度**，而 `Linear01Depth` 期望的输入是**深度缓冲值**。D3D-like 平台（缓冲值 `[0, 1]`，Reversed-Z 为 `[1, 0]`）两者数值恰好相同，可以直接传；GL/GLES 的 NDC 是 `[-1, 1]`，直接喂进去会整体错位，需要先按平台换算成缓冲值（`ndc * 0.5 + 0.5`），或干脆直接采样 `_CameraDepthTexture` 的 `.r`。

### 10.3 用 `COMPUTE_EYEDEPTH` / `COMPUTE_DEPTH_01`

直接在顶点着色器里变换到相机空间，把线性深度存进 varying：

```hlsl
#define COMPUTE_EYEDEPTH(o)  o = -mul(UNITY_MATRIX_MV, v.vertex).z
#define COMPUTE_DEPTH_01     -(mul(UNITY_MATRIX_MV, v.vertex).z * _ProjectionParams.w)
```

`_ProjectionParams.w` 是 `1 / FarPlane`。取反是因为 Unity 的观察空间 z 轴翻转为相机前方为 -z（范围 `[-n, -f]`），取反后得到正值。

```hlsl
float depth = 0;
COMPUTE_EYEDEPTH(depth);        // [0, far]
// fragment
depth *= _ProjectionParams.w;   // -> [0, 1]
```

**三种方式都能得到线性深度**：前两种插值得到的是非线性深度（深度缓冲值），需要在片元里再线性化；第三种在顶点阶段直接插值线性值。

### 10.4 深度缓冲里存的到底是什么

以顶点 P 为例：

1. 观察空间坐标 `P_view = (x, y, z, 1)`；
2. 裁剪空间坐标 `P_clip = P_view · M_projection`，范围 `[-w, w]`；
3. NDC：`P_ndc = P_clip / w`，范围 `[-1, 1]`；
4. 屏幕坐标：`screenX = (ndcX * 0.5 + 0.5) * width`，Y 同理；
5. 深度缓冲：`d = ndcZ * 0.5 + 0.5`，范围 `[0, 1]`。

这个 `d` 就是深度图中的值。它是**非线性**的：近处精度高、远处精度低。

<img src="../../技术笔记/assets/DB61C51D1C784170A0D9BC671D43B404.png" width="620" height="286">

<div align="center">投影矩阵</div>

<img src="../../技术笔记/assets/37F416AA65F040048103CDC5B245D08A.png" width="620" height="286">

<div align="center">顶点 P 的投影变换</div>

### 10.5 Unity 中深度图的使用

```csharp
camera.depthTextureMode = DepthTextureMode.Depth; // 告诉相机需要深度纹理
```

```hlsl
sampler2D _CameraDepthTexture;

// 屏幕后处理：直接用 uv
float depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, uv);
// 等价于 float depth = tex2D(_CameraDepthTexture, uv).r;
float linearDepth = Linear01Depth(depth, _ZBufferParams);

// 投影纹理采样
float d = SAMPLE_DEPTH_TEXTURE_PROJ(_CameraDepthTexture, UNITY_PROJ_COORD(screenPos));

// 指定 LOD
float d2 = SAMPLE_DEPTH_TEXTURE_LOD(_CameraDepthTexture, UNITY_PROJ_COORD(float4(uv, 0, 0)));
```

`SAMPLE_DEPTH_TEXTURE` 的源码就是 `tex2D(sampler, uv).r`。`UNITY_SAMPLE_DEPTH` 会按平台分派：PS3 用三通道点积解码，旧 Flash 平台用 `DecodeFloatRGBA`，其余平台直接取 `.r`。因此在需要兼容多通道编码深度的平台上，应使用 `UNITY_SAMPLE_DEPTH` 而不是裸 `tex2D`。

`ComputeScreenPos` 返回的是**齐次坐标下的屏幕坐标，范围 `[0, w]`**，设计上就是给 `tex2Dproj` 用的：

```hlsl
inline float4 ComputeNonStereoScreenPos(float4 pos)
{
    float4 o = pos * 0.5f;
    o.xy = float2(o.x, o.y * _ProjectionParams.x) + o.w; // _ProjectionParams.x = -1 表示投影 y 轴被翻转（渲染到 RT）
    o.zw = pos.zw;
    return o;
}
```

`tex2Dproj` 与 `tex2D` 的区别也在这里：`tex2Dproj` 采样前会把 uv 的 xy 除以 w，完成透视除法；

```hlsl
float a = tex2Dproj(_CameraDepthTexture, UNITY_PROJ_COORD(screenPos)).r;
float b = tex2D(_CameraDepthTexture, screenPos.xy / screenPos.w).r; // 等价
```

### 10.6 线性化函数

```hlsl
// Z buffer -> 线性 0..1 深度（0 在眼睛，1 在远平面）
inline float Linear01Depth(float z, float4 zBufferParam)
{
    return 1.0 / (zBufferParam.x * z + zBufferParam.y);
}

// Z buffer -> 线性相机深度（世界单位）
inline float LinearEyeDepth(float z, float4 zBufferParam)
{
    return 1.0 / (zBufferParam.z * z + zBufferParam.w);
}
```

`_ZBufferParams` 在启用 reversed-Z 的平台上为：

```text
x = -1 + far/near
y = 1
z = x/far
w = 1/far
```

## 11. 浮点数编码到 RGBA8

现代平台基本都支持渲染到 float16/float32 纹理，编码到 RGBA8 主要是为了兼容旧平台（典型场景：pre-depth 把 32 位深度编码进 RGBA8）。

```hlsl
inline float4 EncodeFloatRGBA(float v)
{
    float4 enc = float4(1.0, 255.0, 65025.0, 16581375.0) * v;
    enc = frac(enc);
    enc -= enc.yzww * float4(1.0 / 255.0, 1.0 / 255.0, 1.0 / 255.0, 0.0);
    return enc;
}

inline float DecodeFloatRGBA(float4 rgba)
{
    return dot(rgba, float4(1.0, 1 / 255.0, 1 / 65025.0, 1 / 16581375.0));
}
```

原理：把 `v` 依次乘以 `255^0..255^3`，取小数部分得到各位"数字"，再减去进位，使每个通道落在 `[0,1]`；解码时按 `255` 的幂次加权求和。

**精度说明（易错点）**：理论上 4 个 8 位通道可表示 `255^4 ≈ 4.2e9` 个台阶，但编解码全程使用 float32，尾数只有 23 位（含隐含位 24 位），因此**实际有效精度约为 24 位左右**，而不是 32 位。当 `v` 接近 1 时，高位通道的小数部分已接近 float32 的分辨率极限，低位通道提供的信息会被舍入误差吞掉。需要更高精度时应直接使用 `R32F` 或 `R32_UINT` 深度纹理。

<img src="../../技术笔记/assets/WEBRESOURCEf4411f83185cb16c4b0389ba4158cfa3.png" width="548" height="1398">

## 12. 顶点输入 position 用 float3 还是 float4

```hlsl
struct appdata
{
    float3 vertex : POSITION;
};

struct appdata
{
    float4 vertex : POSITION;
};
```

两者都能编译通过，区别在于从顶点缓冲读取的通道数：

- `float3` 只读取 xyz，w 分量在寄存器中由驱动/编译器补成 **1.0**（作为常量），后续 `UnityObjectToClipPos` 使用 `float4(v.xyz, 1.0)` 语义；
- `float4` 读取完整 4 分量，w 直接来自顶点数据（Unity 的网格顶点 w 通常也是 1）。

汇编上的表现是输入声明与寄存器占用不同：`float3` 版本只声明 3 个分量，`float4` 版本占满一个 4 分量输入寄存器。对绝大多数模型，顶点 w 均为 1，因此两者计算结果一致，`float3` 更省寄存器；只有在自定义网格、需要顶点 w 参与变换（例如某些投影/扭曲效果）时才必须用 `float4`。

<img src="../../技术笔记/assets/7E5058C0B08741388BDEDC900ED7C60A.png" width="620" height="313">

（待核实：不同图形 API 与编译器对缺失 w 的处理是否严格为常量 1.0，D3D11 与 Metal 在部分版本上的汇编表现略有差异。）

## 13. Shader 变体

### 13.1 为什么需要变体

以"是否提供法线贴图"为例，有两种实现思路：

1. 在底层 Shader 里定义外部传入的 int，用 `if` 分支处理。缺点：引入运行时分支，影响 GPU 执行效率。
2. 在上层（ShaderLab）用宏封装，编译期按宏组合编译出多套底层 Shader。缺点：Shader 体积变大。

Unity 内置 Shader 采用后者。`multi_compile` 与 `shader_feature` 定义关键字（keyword），编译时按组合生成多份代码，每一份就是一个 **Variant**。内置宏可以用 `#pragma skip_variants XX` 屏蔽，有助于减小 ShaderLab 体积与包体。

### 13.2 multi_compile

```hlsl
#pragma multi_compile _ _NORMALMAP
#pragma multi_compile _QUALITY_LOW _QUALITY_MEDIUM _QUALITY_HIGH
```

- 生成所有组合，**全部打包**，即使没有任何材质使用。
- 运行时可通过脚本自由切换。
- 每个关键字占用 keyword 名额。Unity 的 local keyword 上限为 256/Shader，内置关键字已占用一部分，变体数量需要严格控制。
- 默认选择第一个变体。
- 行首的 `_` 表示"该关键字未定义"的那个变体，可以少声明一个关键字而仍然具备开关能力。

### 13.3 shader_feature

```hlsl
#pragma shader_feature _ _NORMALMAP
```

- 是 `multi_compile` 的子集，**作用域相同**：两者的关键字都可以逐材质开关，也可以用 `Shader.EnableKeyword` 做全局开关；真正的差别只在未被任何材质引用的变体是否在打包时剔除。
- 未被任何材质使用的变体在**打包时被剔除**。
- 关键坑：正因为会被剔除，**打包后无法通过代码开启一个从未被任何材质使用过的 keyword**——运行时 `EnableKeyword` 找不到对应变体，表现为功能无效。需要在编辑器里保证该变体被至少一个材质引用，或改用 `multi_compile`。

### 13.4 Material 与 Variant 的关系

一个 Material 同一时刻只对应它所使用 Shader 的一个 variant。切换通过：

```csharp
material.EnableKeyword("_NORMALMAP");
material.DisableKeyword("_NORMALMAP");

Shader.EnableKeyword("_GLOBAL_FOG");   // 影响所有使用该 Shader 的材质
Shader.DisableKeyword("_GLOBAL_FOG");
```

在编辑器非运行模式下修改，keyword 会被写入 `.mat` 文件（需将 `Asset Serialization` 设为 `Force Text` 才能用文本编辑器查看）。

另外，`[Toggle]` 装饰器会自动生成"属性名大写 + `_ON`"的宏：

```hlsl
[Toggle] _LambertLit("Lambert Lit", Float) = 1.0
// 自动生成 _LAMBERTLIT_ON
#pragma shader_feature _ _LAMBERTLIT_ON
```

## 14. Shader 编译优化

编译器的核心思路是**从输出倒推做表达式引用分析**，把整个计算过程视为一张表达式依赖图。

1. **未引用丢弃**：所有未被输出引用的 struct、局部变量、函数都会被坍塌丢弃。
2. **重复赋值优化**：同一变量的多次写入只保留最后一次。
3. **跨行合并**：相邻语句中可合并的表达式会被合并。
4. **远距离相同表达式合并**：公共子表达式消除（CSE）。
5. **常量表达式折叠**：
   - 乘以 1 直接消除，不产生指令；
   - 乘以 0 会向前追溯，关闭整条计算链路。

实践结论：

- 不需要为了"省性能"手动删掉未使用的中间变量，编译器会做；
- 但**不要依赖编译器消除有副作用的计算**（如 `tex2D` 采样、`ddx/ddy`），这些不会被当作纯表达式删除；
- 把 `0` 因子的分支提前，可以让整条链路被折叠掉，这是做 feature 开关时"零成本"的来源。

<img src="../../技术笔记/assets/WEBRESOURCEc07504b3e6969ea267e5c6b58d865401.png" width="672" height="321">

<img src="../../技术笔记/assets/WEBRESOURCE4a1b5a6eb2d6a7bfc8102df6da280bd0.png" width="1038" height="182">

<img src="../../技术笔记/assets/WEBRESOURCE107984806327d25152ff87f17f993f22.png" width="817" height="303">

<img src="../../技术笔记/assets/WEBRESOURCE68130cf14b56f643be1b975d72a28cee.png" width="1035" height="571">

<img src="../../技术笔记/assets/WEBRESOURCE46e8f8b0a20e0c0667561493d0f56b73.png" width="994" height="324">

## 15. GPU Instancing

### 15.1 与静态/动态合批的对比

| 方式 | 时机 | 缺点 |
|---|---|---|
| 静态合批 | 运行前把静态网格合并成更大的网格 | 材质必须相同；需要额外存储合并后的网格数据 |
| 动态合批 | 运行时合并视野内的动态对象 | 合并本身有 CPU 开销；只适合小型网格 |
| GPU Instancing | 运行时对可见对象，一次 draw call 渲染同一网格多次 | 受实例属性、排序等条件限制 |

GPU Instancing 的作用：

1. 批量渲染 mesh 相同的物体，降低 draw call；
2. 这些物体可以有各自的参数（通过 per-instance 属性）。

### 15.2 Unity 侧写法

Shader 中需要：

```hlsl
#pragma multi_compile_instancing

struct Attributes
{
    float4 positionOS : POSITION;
    float2 uv         : TEXCOORD0;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

struct Varyings
{
    float4 positionCS : SV_POSITION;
    float2 uv         : TEXCOORD0;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

UNITY_INSTANCING_BUFFER_START(Props)
    UNITY_DEFINE_INSTANCED_PROP(float4, _Color)
UNITY_INSTANCING_BUFFER_END(Props)

Varyings vert(Attributes input)
{
    Varyings output;
    UNITY_SETUP_INSTANCE_ID(input);
    UNITY_TRANSFER_INSTANCE_ID(input, output);
    output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
    output.uv = input.uv;
    return output;
}

half4 frag(Varyings input) : SV_Target
{
    UNITY_SETUP_INSTANCE_ID(input);
    half4 c = UNITY_ACCESS_INSTANCED_PROP(Props, _Color);
    return c;
}
```

各宏的含义：

| 宏 | 展开/作用 |
|---|---|
| `UNITY_VERTEX_INPUT_INSTANCE_ID` | 等价于增加字段 `uint instanceID : SV_InstanceID;` |
| `UNITY_INSTANCING_BUFFER_START(Props)` | `CBUFFER_START(UnityInstancing_Props) struct {` |
| `UNITY_INSTANCING_BUFFER_END(Props)` | `} PropsArray[UNITY_INSTANCED_ARRAY_SIZE]; CBUFFER_END` |
| `UNITY_SETUP_INSTANCE_ID(v)` | `unity_InstanceID = v.instanceID + unity_BaseInstanceID;` |
| `UNITY_TRANSFER_INSTANCE_ID(v, o)` | `o.instanceID = v.instanceID;` |
| `UNITY_ACCESS_INSTANCED_PROP(Props, var)` | `PropsArray[unity_InstanceID].var` |

C# 侧：

```csharp
// 1. 材质勾选 Enable GPU Instancing
material.enableInstancing = true;

// 2. 逐实例参数
var block = new MaterialPropertyBlock();
block.SetVector("_Color", Color.red);
renderer.SetPropertyBlock(block);

// 3. 或手动批量绘制
Graphics.DrawMeshInstanced(mesh, 0, material, matrices, count, block);
```

### 15.3 踩坑结论

1. 使用多个 per-instance 属性时，`MaterialPropertyBlock` 中**不必**填满所有属性。
2. 某个 instance 缺少属性时使用材质默认值；没有默认值则用 0。
3. **不要把非 instanced 的属性写进 `MaterialPropertyBlock`**，否则 instancing 会被关闭，并生成不同的材质实例。
4. **静态合批的优先级高于 GPU Instancing**：勾了 Batching Static 的对象不会被 instancing。
5. 以下情况会导致 instancing 失败：
   - 材质被修改（产生材质变体）；
   - 需要按深度排序（透明物体）。

<img src="../../技术笔记/assets/A03AD3545DD64256800FF857DE87B62C.png" width="620" height="566">

## 16. Compute Shader

> 原文仅有一张截图，以下为该主题的基础补充，仅保留与 Unity 侧使用直接相关的内容。

一个 `.compute` 文件的基本结构：

```hlsl
#pragma kernel CSMain

RWTexture2D<float4> _Result;
uint _RTSizeX;
uint _RTSizeY;

[numthreads(8, 8, 1)]
void CSMain(uint3 id : SV_DispatchThreadID)
{
    if (id.x >= _RTSizeX || id.y >= _RTSizeY) return;
    float2 uv = (id.xy + 0.5) / float2(_RTSizeX, _RTSizeY);
    _Result[id.xy] = float4(uv, 0, 1);
}
```

C# 侧调度：

```csharp
var cs = Resources.Load<ComputeShader>("PostProcess/MyCS");
int kernel = cs.FindKernel("CSMain");
cs.SetTexture(kernel, "_Result", rt);
cs.SetInt("_RTSizeX", rt.width);
cs.SetInt("_RTSizeY", rt.height);
cs.Dispatch(kernel, Mathf.CeilToInt(rt.width / 8f), Mathf.CeilToInt(rt.height / 8f), 1);
```

关键点：

- `numthreads(x, y, z)` 是**一个线程组**内的线程数；`Dispatch(gx, gy, gz)` 是**线程组的数量**，总线程数为两者相乘。
- `SV_DispatchThreadID` 是全局线程 ID，`SV_GroupThreadID` 是组内 ID，`SV_GroupID` 是组 ID。
- 组内共享内存用 `groupshared`，D3D11 下单组上限 32KB。
- `RWTexture2D` / `RWStructuredBuffer` 属于 UAV，只能通过 `kernel` 索引绑定。
- **Metal 不支持对 Texture 的原子操作**，只支持 `RWBuffer` 的原子操作。需要原子累加时要把二维坐标压成一维：

```hlsl
uint ScreenID2BufferIdx(uint2 id)
{
    return id.y * _RTSize.x + id.x;
}
```

- Dispatch 的线程组 / 线程 ID 是**抽象索引，没有朝向语义**：`id.x`、`id.y` 只是索引编号，往纹理或 RT 写数据时坐标原点由平台与 API 约定决定，需要时自行翻转 y。

<img src="../../技术笔记/assets/WEBRESOURCEd18f3368135e1b1139a66eebd87da199.png">

## 17. Unity 中的 Cubemap 实现

### 17.1 Panorama 图的方向约定

1. 经度：以 **-z 轴为起点**，绕 y 轴的欧拉角，范围 `[0, 360]`。
2. 纬度：绕 x 轴的角度，范围 `[-90, +90]`。

```csharp
// 欧拉角 -> 方向向量
Vector3 direction = Matrix4x4.Rotate(Quaternion.Euler(latitude, longitude, 0)) * Vector3.back;
// Vector3.back = (0, 0, -1)
```

### 17.2 立方体六面的约定

1. 每个面在生成时都做了**垂直方向（v 方向）的镜像翻转**，所以从上往下对应 uv 的 v 从 0 到 1。
2. Unity 世界坐标系是左手系；由于经度从 -z 轴开始，**+z 轴指向 panorama 的中间位置**。
3. 方向向量采样 Cubemap 的三步：
   1. 取 xyz 中**绝对值最大**的分量作为主轴，决定采样的二维面；
   2. 用另外两个分量除以主分量的绝对值，归一化到 `[-1, +1]`，再映射到 `[0, 1]`；
   3. 对该面做二维纹理采样（线性滤波）。

### 17.3 HLSL 采样实现

```hlsl
float2 Scale01(float2 direction)
{
    return direction * 0.5 + 0.5;
}

float4 SampleEnvironmentMap(float3 direction)
{
    direction = normalize(direction);
    float3 absDir = abs(direction);
    float4 color = 0;
    float2 uv = 0;

    if (absDir.x > absDir.y && absDir.x > absDir.z)
    {
        if (direction.x > 0.0)
        {
            uv.x = -direction.z / absDir.x;
            uv.y = -direction.y / absDir.x;
            uv = Scale01(uv);
            color = tex2D(_PositiveX, uv);
        }
        else
        {
            uv.x = direction.z / absDir.x;
            uv.y = -direction.y / absDir.x;
            uv = Scale01(uv);
            color = tex2D(_NegativeX, uv);
        }
    }
    else if (absDir.y > absDir.z)
    {
        if (direction.y > 0.0)
        {
            uv.x = direction.x / absDir.y;
            uv.y = direction.z / absDir.y;
            uv = Scale01(uv);
            color = tex2D(_PositiveY, uv);
        }
        else
        {
            uv.x = direction.x / absDir.y;
            uv.y = -direction.z / absDir.y;
            uv = Scale01(uv);
            color = tex2D(_NegativeY, uv);
        }
    }
    else
    {
        if (direction.z > 0.0)
        {
            uv.x = direction.x / absDir.z;
            uv.y = -direction.y / absDir.z;
            uv = Scale01(uv);
            color = tex2D(_PositiveZ, uv);
        }
        else
        {
            uv.x = -direction.x / absDir.z;
            uv.y = -direction.y / absDir.z;
            uv = Scale01(uv);
            color = tex2D(_NegativeZ, uv);
        }
    }
    return color;
}
```

### 17.4 为什么符号是这么取的

以 +X 面为例：

```text
u = -direction.z / absX;
v = -direction.y / absX;
```

`direction.z`、`direction.y` 的取值范围都是 `[-1, 1]`：

- `direction.z` 从 -1 到 +1，在 +X 面上是从**右向左**走；而 uv.x 需要从左往右，所以取负号。
- `direction.y` 从 -1 到 +1，在 +X 面上是从**下向上**走，与常规 uv 的 v 方向一致；但 Unity 的 Cubemap 每个面在 v 方向上已经翻转，需要再取负号抵消。

### 17.5 C# 侧等价采样

用 `Cubemap.GetPixels(face, 0)` 拿到 `Color[]` 后没有硬件采样可用，需要自己做双线性插值：

1. 用同样的分支选面，得到 `uv`；
2. `uv *= envDim - 1` 转成数组下标（`envDim = sqrt(colors.Length)`）；
3. 取 floor/ceil 四个邻域像素，按 `blendU`、`blendV` 做两次 `Color.Lerp`。

注意 `uv[1]` 不需要再翻转——Cubemap 六个面的纹理数据本身已经上下翻转过了。

## 18. 其他工程结论

- 矩阵的**行主序/列主序对矩阵乘法结果没有影响**，但对矩阵的**构造**有影响。按行填充 `half3x3(tangent, bitangent, normal)` 后，`mul(normalTS, tangentToWorld)` 才能得到正确结果。
- HLSL 的 `groupshared` 只用于 Compute Shader 的组内共享内存；`static` 全局变量对应用程序不可见；`nointerpolation` 阻止顶点输出被插值。
- 后处理绘制的是**全屏三角形**，`cmd.Blit` 实际只画一个三角形：

```hlsl
float4 GetFullScreenTriangleVertexPosition(uint vertexID, float z = UNITY_NEAR_CLIP_VALUE)
{
    float2 uv = float2((vertexID << 1) & 2, vertexID & 2); // (0,0) (2,0) (0,2)
    return float4(uv * 2.0 - 1.0, z, 1.0);                 // (-1,-1) (3,-1) (-1,3)
}
```

- URP 的 Single Pass Forward 会把光源一次性传入 Forward Pass，但可传数据有限：默认 Forward 路径为 **1 盏主光 + 每物体 4 盏附加光**（附加光数量可在 URP Asset 配为 1–8），Forward+ 路径支持的光源数高于此。

## 原始笔记索引

- `Unity/pbr in unity.md`
- `Unity/Unity PBR金属流.md`
- `Unity/CBuffer.md`
- `Unity/Common.h中的一些约定.md`
- `Unity/TEXTURECUBE_PARAM(tex, texSampler).md`
- `Unity/Blend Mode的三种形式.md`
- `Unity/Properties的Decorator Attribute.md`
- `Unity/法线贴图使用中的问题.md`
- `Unity/浮点数编码到RGBA8.md`
- `Unity/片段着色器中获取深度值.md`
- `Unity/vs中position槽位float3与float4的区别.md`
- `Unity/Shader.SetGlobalFloat不生效.md`
- `Unity/shader变体.md`
- `Unity/shader编译优化.md`
- `Unity/Compute Shader.md`
- `Unity/GPU Instancing.md`
- `Unity/Unity中的Cubemap实现.md`