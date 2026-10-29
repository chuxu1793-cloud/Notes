# Unity 图形开发工作日志技术摘录

本文摘自一份 2021–2022 年的每日工作日志，记录的是一个基于 Unity 自研 SRP（在 URP 基础上定制的项目管线，着色器统一走内部模板生成）的图形/TA 开发过程。原文按天流水记录，混有大量任务清单与进度描述；这里只保留可复用的结论、规则、参数和踩坑解法，并按主题重新编排，**不按时间顺序**。凡原文未写清前提的条目，按上下文尽量标注引擎/平台/管线，无法确认的标「待核实」。原文中的图片引用全部是失效的本地绝对路径，已删除；承载结论的图以文字转述并标「原图缺失」。

## 自研软渲染器与光栅化

这一部分来自入职初期的练手项目：一个不依赖图形 API 的 CPU 软光栅渲染器（C++），用于把渲染管线各阶段手工实现一遍。

### 坐标系手性转换

从右手系迁移到左手系，需要成体系地改，而不是只改一个矩阵：

- 需要改的变换：视口变换、相机变换、透视变换；数据布局由「矩阵 × 列向量」改为「行向量 × 矩阵」，因此所有变换的乘积顺序也要反过来。
- 叉乘结果与手性绑定：同一组向量在左手系与右手系下叉乘结果方向相反，凡是依赖叉乘求法线、求切线、判断绕序的地方都要复核。
- 四元数同样受影响：左手系下四元数的叠加顺序要改，由四元数生成矩阵的步骤也要改（原文记为「似乎是转置」，**待核实**）。
- 3×3 的旋转矩阵可以直接转置（5-12 原话），但这条只在**行/列布局切换**时成立：同一线性映射在「矩阵 × 列向量」与「行向量 × 矩阵」两种写法之间互换时，矩阵内容互为转置。**左右手系切换不能只靠转置**：需要共轭变换 `S·R·S`（`S = diag(1,1,-1)`），即在两侧各乘一次轴翻转；把旋转矩阵直接转置当作手性转换是错的。「正交矩阵转置即逆、前提是不含缩放与剪切」为整理者补充的通用性质。

### 背面剔除

- 背面剔除在 NDC 空间进行；NDC 和相机坐标系一样采用左手系（5-11）。
- 背面剔除函数的实现约定（5-11）：顶点采用逆时针排列的多边形，基于左手系判断正反面（「绕序与手性需成对定义」为整理者归纳）。

### 线段光栅化

- 采用 Bresenham 算法（5-8）。
- 折线在连接点处容易断裂：原文的解释是「由浮点数推导出的末端端点」与「下一个要连接的线段起始端点」不重合。做法是先把线段两个端点的浮点坐标转换成屏幕像素坐标，再以量化后的像素为端点绘制（5-12）。原文作者自己标注了「上面的描述的不确定对不对」，此条**待核实**。
- 线框模式修改时「线段裁剪样式改为多边形」（5-19 原话）；原文未说明不改会出现什么现象。

### 裁剪与透视除法

一个典型的透视除法除零 bug，根因是裁剪分支不完整：

1. 三角形裁剪时，「两个顶点都在裁剪体内」的情况本应直接复制顶点，但代码漏判，仍然去和 w 平面求交；
2. 此时 `prev._w - curr._w == 0`：`0/0` 得到 `NaN`，`x/0` 按符号得到 `±inf`；
3. 用这个 `t` 做 `lerp`：`t = inf` 时 `w = w1 + t·(w2 − w1)` 展开为 `inf + inf·0`，即 `0·inf`，结果是 `NaN`；若两个 `w` 都有限、只是分母接近 0，则 `t` 极大但有限，`w` 会变成极小的正值或负值；
4. 不可信的 `w` 进入透视除法（`xyz / w`）：`NaN` 转整数时不同硬件行为不同（x86 上 `INVALID` 常落到 `INT_MIN`），会把包围盒撑爆；分母近零的极小 `w` 会把 NDC 放大到极大的数值。两者都可能表现为屏幕上离谱的巨大三角形。

规则（由该 bug 归纳）：任何平面求交前先判「两端点是否同侧」，同侧直接复制，不做除法。症状上，巨大三角形指向的是 w 的异常，但不应直接反推为某一种成因：可能来自 NaN 转整数（如 x86 上 INVALID→INT_MIN 撑爆包围盒），也可能来自分母近零造成的极小正 w，需结合实际 lerp 写法与取整方式判断。

### 排序与显示模式

- 不透明物体从前往后排序，透明物体从后往前排序（5-8 原话）。「不透明前向排序可利用 early-z 减少 overdraw」为整理者补充。
- 练手渲染器实现了三种显示模式：线框、Gouraud 着色、纹理映射；光照模型用「环境光 + 一盏平行光」；后续补充了 Phong 着色与「纹理 + 顶点着色」混合样式，以及线框与 Gouraud 叠加。
- 顶点法线需要在导入模型后重算（原文只记录了「顶点法线修改」这一项工作，未记录算法，**待核实**具体做法）。

### Skybox 采样错误

直接用立方体几何的法线去采样 cubemap 会产生「天空盒不随视角变化、像一张静态背景图」的现象：立方体旋转时法线跟着转，但 cubemap 本身不变，采样结果等价于固定背景。

解法：把 skybox 当成六个面各自带纹理坐标的普通贴图模型，用面自身的 UV 采样，而不是用法线当方向向量。

## Unity SRP / URP 工程

### 渲染顺序与排查方法

原文（8-4）记录的渲染顺序为：先按 Layer 排列，Layer 相同按 Order 排序；再按 RenderQueue 数字由小到大排列；Opaque 队列中同一 queue 的物体由近及远，Transparent 队列中同一 queue 的物体由远及近。

> 整理者推断，待核实：RenderQueue 属于材质/Pass 级别，若它实际先于 Order 生效，则同一 SortingLayer 下应先按 RenderQueue 划分，再在同 queue 内比较 Order 与距离；原文的排序记录未涉及这种情形。

不透明物体一般需要开启 ZTest + ZWrite，可以执行 early-z（8-4 原文备注）。

排查「某物体为什么在这个时刻被画」的五步（8-4）：确定它用的 Renderer → 确定所处的 RenderPass → 比较 Layer → 比较 Order → 比较 Queue。

### RT 与 clear 行为

- clear color / clear depth 的位置在 `ScriptableRenderer.cs` 的 `SetRenderPassAttachments` 附近（原文 11-2 自标为「猜测」，**待核实**，但可作为排查入口）。
- 「修改 clear 深度值」功能后来移动到 `ArkFeatureDepthInColorAlpha` 中（11-8 原文记录；从命名看应是一个 RenderFeature，原文未明说，**待核实**）。
- URP 的 skybox shader 是引擎内置的 ShaderLab/HLSL 资源，不在 C++ 源码中（11-05 记录的寻找结果）。
- URP 写入深度纹理的位置也在引擎侧；Editor 中修改其 alpha 值的做法与平台相关，见「深度与平台差异」（源 11-05、11-2）。

### 后处理接入方式

在自研管线下新增一个后处理效果的标准流程（以屏幕扭曲 Distort 为例）：

```text
1. 新建后处理 shader 文件（功能实现在 Distort.shader）
2. 新建 C# 接口，仿照 ScanLine.cs 的写法
3. 参数传递写在 PostProcessPass.Ark.cs
```

两个高频故障：

- 在 Editor 中给 Volume 添加该后处理 override 时提示 material 为空 —— 原因是美术工程里没有执行 `InitViaReflection`，重新 import 对应的 `Distort.cs` 即可恢复。
- 画面出现黑块 —— 优先检查是否片元被 `discard` 掉了（模板/alpha 测试的副作用），而不是先怀疑采样。

### Distort：用模板缓冲省一次 color buffer 拷贝

常规做法要把 camera color 拷一份再拿去做 UV 偏移采样；改为把「偏移量」和「哪些像素要偏移」塞进已有的两个通道：

- 特效 `Distort.shader`：开启模板写入，把该像素标记为「需扭曲」；同时计算 uv offset 并写入 `_CameraColorTexture` 的 **alpha 通道**（配合 `ColorMask` 只写 alpha，不破坏 RGB）。
- 后处理 `Distort.shader`：关闭模板写入、开启模板测试；从 `_CameraColorTexture` 的 alpha 读出 offset 应用到 uv 上；再采样同一张 `_CameraColorTexture` 输出。

净效果：省掉一次全屏 color buffer 拷贝，代价是占用 alpha 通道（因此该管线中 alpha 通道不能再承载透明度语义）。相关贴图约定：`t_wl_51`（alpha/遮罩）、`t_normalmap_02_n`（法线）。

### Editor 脚本注意事项

- 脚本重编译（域重载）后，非序列化字段、`static` 字段与运行时查找得到的缓存会丢；public 字段与 `[SerializeField]` 字段（含场景对象引用）会被序列化/反序列化保留，并非"所有字段都被重新初始化"。需要持久化的运行时查找结果要放在 `[UnityEditor.Callbacks.DidReloadScripts(0)]` 回调里重新做一遍。
- `hideFlags = HideFlags.DontSave` 只保证对象不被写入磁盘，**切换场景时对象依然存在**，必须自己监听场景切换并手动销毁，否则会出现重复对象。
- 用 `[SelectionBase]` 标记根节点，可让 Scene 视图点选时选中根而不是子 Mesh；工具生成的临时对象用 `[NotSave]` 命名约定 + 不参与保存，避免被美术误存进场景。

## Shader 框架与变体

### 模板生成与 IMPORT 宏

项目里所有角色/场景 shader 由统一模板生成，各 shader 的差异用一组 `#define` 汇总在文件头部的 `HLSLINCLUDE` 中：

```hlsl
HLSLINCLUDE
    #define ARK_SRP_BATCHER
    #define IMPORT_PBR_TEXTURE
    #define IMPORT_EMISSION
    #define IMPORT_LAYER2
    #define IMPORT_DITHER_OR_DISSOLVE
    #define IMPORT_DETAIL
    #define IMPORT_GLITCH
    #define IMPORT_HIGHLIGHT
ENDHLSL
```

Cartoon 系列在此基础上再加一组（示意）：

```hlsl
#define IMPORT_PBR_TEXTURE
#define IMPORT_DYEING
#define IMPORT_GRID
#define IMPORT_EMISSION
#define _MODEL_CARTOON
#define IMPORT_DITHER_OR_DISSOLVE
#define IMPORT_RIM_LIGHT
```

使用规则（均为踩坑后得出的结论）：

- IMPORT 宏**按功能分开写**，不要集中堆在一处（8-9 的结论，理由为整理者推断：关掉某个功能时便于定位受影响的 include）。
- 类似 `#define IMPORT_CARTOON_BROW`、各 shader 特有的宏，必须写在 `// <autogen: Master:Feature>` 标记**外面**（8-13 原文只说「要提到外面」，未说明机制）；DEBUG 面板里的选项（如 `IMPORT_RIM_LIGHT`）可以由模板勾选生成，其它 pass 只能手写（8-13 原话）。
- `IMPORT_PBR_TEXTURE` 会影响其它引用同一 cginc 的 shader（Billboard、Grass、TreeLeaves），改这个宏要连带回归这些 shader。
- 模板与生成文件不同步是高频问题，典型原因（源 9-24）：改了模板但没重新生成（`PBR_Planet_Graded`、`PBR_FX_Graded` 都是 IMPORT 改完忘记更新）；或有人手改了生成结果 —— `PBR_Cartoon_NPC_Graded` 多出 `[ArkToggle] _ZWrite`、并删除了 `[HideInInspector] _ZWrite`，`PBR_Cartoon_Graded` **多出** `CLUSTER_FORWARD_PLUS`，`PBR_Cartoon_Face_Graded`／`PBR_Cartoon_Brow_Graded` 只多一个换行。定位手段是直接 diff 模板与产物。
- `PBR` 的主体实现在 `ForwardPass.cginc` 中（原文 7-22 原话）；cginc 后缀后来统一改为 `.hlsl`（8-25 已完成）。

### SRP Batcher 兼容

三类报错与对应根因：

| 报错 | 根因 |
| --- | --- |
| `UnityPerMaterial var is not declared in shader property section (_DebugWeakShadowStrength)` | 变量在 `UnityPerMaterial` cbuffer 里，但 Properties 块缺少对应属性 |
| `invalid subscript 'vertColorMask'` | 结构体/变量字段未定义（宏开关导致某分支引用了不存在的成员） |
| `Material property is found in another cbuffer than "UnityPerMaterial" (_ShadowMode)` | 变量声明落在了别的 cbuffer，需移到 `UnityPerMaterial` 或改为全局 |

硬性规则：

1. 凡是在 `UnityPerMaterial` cbuffer 中出现的变量，**都必须在 Properties 段声明**，否则整个 shader 不兼容。
2. 同一个 shader 的所有变体必须拥有**完全相同的 property 接口**；某个变体多一个属性就会破坏兼容。
3. 属性数组是已知的兼容性死角：Unity 不支持在 Properties 段声明数组，因此 perMaterial 数组会破坏 SRP Batcher；perDraw 里的 `real4 unity_LightIndices[2]` 之所以没事，是因为它在 perDraw 而非 perMaterial。8-23 记录了「如何既用数组又保持兼容」的提问；8-20 对 `dyeingCollection` 需要属性数组一事记为「不支持」（实现方式未留下，**待核实**）。
4. `_ST` 这类由 Unity 隐式生成的缩放平移变量（如 `_ShapeFXMaskTex_ST`），是否需要显式声明取决于是否被当作普通属性读取，原文 8-20 记为「不确定」，**待核实**。

`SRP_BATCHER` 宏在 pass 中的三种来源（源 8-4，原文列在 to be confirmed 下，**待核实**）：

```text
forward pass  -> 模板上勾选，自动插入
自定义 pass   -> 手写 // <autogen:  Master:SRPBatcherFeature>
UsePass 生成的 pass -> 模板自动生成，无需干预
```

其它结论：

- Cartoon 无法做到「单向依赖 PBR」，因为 Cartoon shader 中直接使用了 `UsePass`，依赖是硬链接。
- 每个 Cartoon pass 都要带 `_MODEL_CARTOON` 宏，否则在非 SRP Batcher 路径下报错、且无法切换（原文列为 to be confirmed，**待核实**）。
- **不需要**兼容 SRP Batcher 的 shader：只用 GPU Instancing 的（`Grass`、`MeshGrass`）、以及整个场景只绘制一次的（天空、星球、云、大气轮廓一类）。判断标准是「是否真的会被频繁复用材质」，而不是「能不能改」。
- 当年兼容性摸底时，不兼容的 cginc 有：`ArkDayNightCommon`、`ArkDebug`、`ArkHeightFog`、`ArkWeather`、`ArkDyeing`；已兼容的有：`ArkDissolve`、`ArkDither`、`ArkEmission`、`ArkPBRTexture`、`ShadingModels`、`ArkCartoon`、`ArkRimLight`、`BaseStruct`、`ArkLayer2`、`ArkShadow`、`LightingModel`、`GICommon`、`Utils`、`ArkDetail`。
- 12-14 记录「完成了安卓平台下 srp batcher 兼容性修改」（原文未说明是否必须单独验证；整理者提示：桌面端通过不等于移动端通过）。

### 变体与关键字管理

- **变体裁剪发生在美术工程侧**：美术工程编译时做变体裁剪，然后把结果打进 AssetBundle 供客户端加载；因此客户端里**不存在调试用变体**，任何依赖调试关键字的验证都必须在美术工程里做。
- 验证流程：在 `ark_art` 的 Assets 下改 shader，想在 `ark_client` 看到效果，必须打 AssetBundle（Windows）并更新 `ark_resource`（8-3）。
- Dither 与 Dissolve 早期共用 `DITHER_OR_DISSOLVE` 一个宏，后来拆分，默认值按 shader 系区分：Cartoon 系默认开启，PBR 默认关闭，其它 shader 均不支持（11-11）。拆分动机原文未记（整理者推断：两者语义与代价不同、需要分别开关，**待核实**）。
- IBL 侧提供 `NO_PROBE_SH` / `PROBE_SH` 开关（10-20、10-22 各记录一次）；用途原文未说明。
- 溶解功能增加了材质有效性检查：如果材质所属 shader 不在白名单（`PBR_FX`、`EnergyFlow` 等）且没有 `SUPPORT_DISSOLVE` 关键字，直接 `LogError`；检查只在 development build 启用，正式包关闭。注意 `GFX`、`GFXDecal` 不走框架统一的溶解流程，早期因此频繁误报，需要加入白名单。

```csharp
// 溶解关键字检查（示意）
if (!material.IsKeywordEnabled("SUPPORT_DISSOLVE"))
{
    var sn = material.shader.name;
    if (sn != "Carava/MiniPBR/PBR_FX" && sn != "Carava/MiniPBR/EnergyFlow")
    {
        Debug.LogError($"[Dissolve] {sn} 未开启 SUPPORT_DISSOLVE", material);
    }
}
```

- 材质关键字有效性抽查的做法：逐个材质对比 `m_ShaderKeywords` 与 shader 实际支持的 keyword 列表，剔除历史残留。例如 `TextMeshPro/Distance Field` 上的 `GLOW_ON`/`UNDERLAY_ON` 有效，`Zeus/FloatingCreatures` 上的 `_RAINMODE_ON` 有效，而 `DIRLIGHTMAP_COMBINED LIGHTMAP_ON` 同时出现属正常组合。
- 存在 `NO_LIGHTMAP` 与 `LIGHTMAP_ON` 两个语义相近的关键字同时在 `PBR_Graded` 中使用、并在 `config.hlsl` 中互相牵连，原文未查明原因（**待核实**）。

## PBR 与贴图通道打包

### MSA / 通道组合规则

贴图槽位与通道语义是分离的：`_MainTex` 固定为 **rgbx**（basecolor 的 rgb，x 为备用通道）；第二张图（MSA / secondTex）**始终存在**，但通道含义随材质类型变化。

标准布局：`msoy` = metallic / smoothness / occlusion / 第四通道。

四种组合：

| 组合 | 通道布局 | 适用 |
| --- | --- | --- |
| 标准 | `rgbx + msoy` | 金属 workflow，无 emission、无 alpha |
| 标准 + E/A | `rgbx + msoy + ea` | 需要 emission 或 alpha |
| 绝缘体 + emission | `rgbx + esoy` | 非金属，且不需要单独 ea 贴图 |
| 绝缘体 + alpha | `rgbx + asoy` | 非金属，alpha 存进 msa 的 a 通道 |

绝缘体分支的进一步优化（「纯绝缘体」模式）：如果只需要 alpha 或只需要 emission，可以把它塞进 MSA 的 **m** 通道，从而完全省掉 ea 贴图 —— 绝缘体不需要 metallic，m 通道本来就空闲。

判据与派生宏：

```text
_ALPHATEST_ON || _ALPHABLEND_ON  -> 有 alpha
EMISSION                          -> 有 emission
NON_METALLIC                      -> 绝缘体
```

派生宏（原文 11-09 只列出宏名，未记录判据条件）：

```text
DIELECTRIC_ALPHA_FROM_METALLIC
DIELECTRIC_EMISSION_FROM_METALLIC
```

兜底：两者都不成立时，必须保证 `alpha = 1`、`emission = 0`（原文原话，未说明后果）。

注意「是否绝缘体」这个开关**只在模板内生效**（原文原话）；面板上增加了同名选项（原文 11-09 记为「面板上增加同标题选项」）。纯绝缘体材质还需要把 EA 贴图的 property 从面板上移除（原文 11-25 记录该项已完成）。

### 法线与切线

法线贴图分支（原文实测代码，来自 PBR forward pass）：

```hlsl
#ifdef NO_NORMAL_MAP
    data.normal = i.worldNormal;
#else
    float3x3 t2w = float3x3(i.worldTangent, i.worldBitangent, i.worldNormal);
    data.t2w = t2w;

    float3 normalTW = UnpackNormalRG(alphaChannels.xy);
    data.tangentNormal = normalTW;
    data.normal = lerp(Safe_Normalize(i.worldNormal),
                       Safe_Normalize(mul(normalTW, t2w)),
                       emissiontex.g);
#endif
```

要点：

- 切线空间到世界空间用 `float3x3(tangent, bitangent, normal)` 按行组装，再左乘切线空间法线。
- 这里 `emissiontex.g` 被用作「平面法线 ↔ 法线贴图」的混合权重（用途原文未说明，**待核实**）。
- 法线在 lerp 前后各做一次 `Safe_Normalize`（代码原样）；原文未解释原因。
- `NO_SEPARATE_NORMAL_MAP` 宏用于没有独立法线贴图的材质，此时不采样法线通道。

### 分层材质

`_Metallic` 与 `ArkLayer2.ApplyLayer2MSOColor` 在源 8-9 中作为并列条目出现（原文未说明二者关系，**待核实**）。

## 深度与平台差异

深度保存在一张高精度（float）纹理中（11-2），且**数值方向随图形 API 不同**（源日志此处有两张平台截图，**原图缺失**，下表据原文文字整理）：

| API | 深度方向（近 → 远） | 说明 |
| --- | --- | --- |
| D3D11 | 1 → 0 | `UNITY_REVERSED_Z` 处理方式 |
| Metal | 1 → 0 | `UNITY_REVERSED_Z` |
| Switch | 1 → 0 | `UNITY_REVERSED_Z` |
| Vulkan | 1 → 0 | `UNITY_REVERSED_Z` |
| OpenGL / GLES | 0 → 1 | 正常方式 |

由此派生的两条工程规则（整理者归纳）：

1. 对深度做线性化、比较、阈值判断的代码要按平台分支，不能写死 `0 = 近`。
2. 「把 clear 时的深度值写进 color 的 alpha 通道」：11-05 记录「找到 URP 中写入深度纹理的地方，Editor 中修改其 alpha 值为 0」，并注「`UNITY_REVERSED_Z` 情况下」；11-8 记录该功能移动到 `ArkFeatureDepthInColorAlpha` 中。GL 平台应取何值，原文未记（**待核实**）。

其它：

- shadowmap 的宽高必须一致（10-14 原话；原文未说明原因）。
- 深度纹理格式为 float（11-2 原话）；原文未给出精度估算建议。

## 阴影、贴花与全局光照

### ShadowDecal（贴花阴影）

- 7-21 记录「修正贴花阴影投射区域占比过小导致的性能浪费问题」（原文只此一句，具体修正方式未记录，**待核实**）。
- 阴影投射 pass 中增加了**法线数据源切换**功能（7-21 完成，7-22／7-27 用 shader 模板实现并在模板中加开关）。原文未说明不做切换会出现什么问题。
- 溶解时影子不消失：11-25 勾选完成「溶解后影子不消失」，12-14 记录「解决了之前遗留的溶解时阴影没有消失的 BUG」；12-15 另记 `GFX`、`GFXDecal` 没有走框架中的溶解流程，导致溶解开启检测功能频繁报错（两者是否同一问题，原文未明说，**待核实**）。
- 主光源阴影中的人物需要额外叠加贴花阴影（10-8 记录，12-8／12-14 列为待办；原因原文未说明，**待核实**）。

### 植被阴影的自遮挡

`TreeLeaves.shader` 引用了 `ZShadow`，并带有局部变量 `_ZShadow_ExtraDepthBias`；模板层为此增加了 `ENABLE_ZSHADOW_EXTRA_DEPTHBIAS` 选项。上述三项在源 11-22 都列在「待确定问题」下，用途**待核实**（整理者推测：树叶这类面片几何投射阴影时正反面深度接近，会用额外深度偏移避免 shadow acne；偏移量是否为 per-material、关闭时逻辑是否被消去，原文均未记录）。

### 阴影与 lightmap 踩坑

- **阴影随相机拉近而消失**：处置是禁用 `UpdateCameraDistance`、改用固定的 `cameraDistance`（9-27 起记录，属绕过）；成因原文未记（**待核实**）。
- **主光源阴影随相机操作时隐时现**：10-8 与上一条并列为「两个影子的 BUG」，同样通过禁用 `UpdateCameraDistance`、改用固定 `cameraDistance` 处理；成因**待核实**。
- **镜面反射跳变**：源 10-8 记为「场景缺失默认的 reflection probe，人物离开场景预置的 reflection probe 时导致镜面反射跳变」；`GlobalReflectionProbeManager.cs` 包含 ReflectionProbe 使用与管理功能。补默认 probe 是否即为当时修法，原文未明说（**待核实**）。排查手段（10-8）：在场景里实例化一个 Reflection Probe，用它的作用范围确认角色是否离开了范围。
- **改 shader 后 lightmap 失效**：需要重新载入一次（10-13）；原文未说明原因。
- **希望主光源参与烘焙**：需要把该 Light 的 `Light/Mode` 设置为 `Baked`（10-20）。
- `gi.bakedGI` 存在 0.5 的截断（10-22 记录「检查 gi.bakedGI 的 0.5 截断是否生效 # done」，用途原文未说明）。
- ShadowMask 贴图目前只应用在 Cartoon 系 shader 上（12-22）。两件事分开记：12-14 勾选完成了「shadowmask 与 IDtex 纹理合并」；1-6 记录「去掉 ShadowMask 贴图」，1-10 记录「去除 shadow mask 功能回退，等待美术那边消息」（最终结论**待核实**）。

### 球谐与间接光

- **间接镜面（indirect specular）的生效依赖 indirect diffuse from SH**（9-22 原话）。这是排查「IBL 没效果」时的第一检查项（整理者提示）。
- 间接漫反射的取数逻辑（9-23）：开启间接光照后 `SkySHFragment` 生效，函数内部根据 `SH_FULL_VERTEX`、`SH_FULL_FRAGMENT` 宏返回不同的值；`SH_FULL_FRAGMENT` 的开关在 Inspector 的「实验效果 / GI」下（10-13）。
- 实测（9-23）：把 `ARK_HAS_PROBE_SH` 下面的内容注释掉后，Indirect Diffuse From SH 的指令数增量为 11，和之前一样；原文据此认为该分支没有带来指令增量（是否完全消去原文未进一步说明）。
- 天空遮蔽（skyAO）的做法（10-19）：用**均匀白色的光**作为环境光烘焙，lightmap 中取白色成分为 skyAO 来调制 skySH，其他部分为剩余 GI：

```hlsl
gi.indirectDiffuse = (gi.skySH * gi.skyAO + gi.bakedGI) * surfaceData.occlusion;
```

这样一次烘焙同时拿到「方向性天光」与「遮蔽」（整理者理解，原文未说明收益）。

### IBL 分级方案与指令数

按质量从高到低（`env` 指最终环境镜面项）：

| 方案 | 表达式 | 说明 |
| --- | --- | --- |
| 开 | `env * gi.indirectSpecular` | 采样 cubemap |
| 中 | `env * gi.indirectSpecular'` | 不采样 cubemap（10-20 原话） |
| 简 | `nv = dot(n, v); env = specColor * (0.7 + nv*nv); smoothness = 0.7` | 原文未给说明 |
| 极简 | `env = specColor; smoothness = 0.7` | 原文未给说明 |
| Matcap | `env = smoothness * min(1.0, metallic*0.85 + 0.675) * baseColor * skySH` | 依赖 skySH（10-9 原话）。**方案后被删除，见源 10-27「删除 matcap 功能」** |
| 关 | `env = 0` | |

对应实测指令数（10-27；同一 shader、同一统计口径下；源日志此处的截图已失效，下表据原文文字整理，**原图缺失**）：

| 方案 | 仅 indirect specular | + indirect diffuse |
| --- | --- | --- |
| 开 | 42 | 54 |
| 简 | 29 | 41 |
| 极简 | 26 | 38 |
| Matcap | 17–19 | 30 |

「简」比「极简」多 3 条指令，即 `dp3 + mad + mul`（10-27）；Matcap 最便宜，且 matcap 依赖 skySH（10-9 原话）。表格中的 Matcap 一行对应已删除的方案，见源 10-27。

另一批更细的拆解（10-9／10-10，口径与上表不同，不可直接相加）：

- `env` 项约 16 条，`indirectSpecular` 项约 19 条；去掉系数后 `env'` 约 5 条，matcap 采样约 7 条。
- 只输出 `indirectSpecular` 为 40 条，只输出 `env` 为 37 条，两者同时输出为 56 条（40 + 37 = 77 与 56 不等；存在公共指令被复用为整理者推断）。
- 把 `env` 直接改成返回 1.0，总指令数**不变**（10-9）；原文未进一步解释，整理者理解为 `env` 的计算被编译器消去、开销不在这一项。

单条内置函数的指令代价（可直接用于估算）：

```hlsl
// SAMPLE_TEXTURECUBE_LOD  -> 1 条 sample_l
// reflect()               -> 3 条 (dp3 + add + mad)
// PerceptualRoughnessToMipmapLevel -> 4 条
```

## 性能、变体分级与资源流程

### Shader 性能分级

按「功能开关组合」把 PBR shader 分成 5 级，每级给出实测指令数（原文记作 `23/2`、`37/2`、`78/2`、`104/3`、`130/3`，斜杠两侧的含义原文未说明，推测为顶点/片元分别统计，**待核实**）：

| 级别 | 包含功能 | 指令数 |
| --- | --- | --- |
| 1 | 线性雾 + 顶点级 SkySH/ProbeSH + 直接漫反射 | 23 / 2 |
| 2 | + 法线贴图 | 37 / 2 |
| 3 | + 直接镜面反射 | 78 / 2 |
| 4 | + 间接镜面（IBL） | 104 / 3 |
| 5 | 高度雾 + SH 在 VS/FS 均衡计算 + 法线 + 直接光 + IBL | 130 / 3 |

分级的用途是给美术一个「按角色重要性选档」的量化依据，而不是让所有人无脑用最高档。配套还统计了各单项功能（雾、法线、直接光、IBL、Matcap）的指令增量。

### 指令数归因方法

给 shader 做性能预算时，靠「删代码看数字变化」逐项定位，比看反汇编猜更快：

1. 固定一个只输出目标项的测试变体，编译后用 D3D 反汇编统计 PS 指令数。
2. 一次只注释掉一个功能块，记录增量；增量即为该功能的边际代价（公共项会被编译器复用，所以单项之和通常大于总和）。
3. 用「让函数直接返回常量」的方式验证某项是否真的被消去 —— 例如把 `env` 改成 `return 1` 后指令数不变，说明 `env` 已被常量折叠，真正的开销不在这里。
4. 定位到具体行：`ArkShadow` 第 78 行的 `* unity_LightData.z` 对应 Direct Diffuse，`ForwardPass.hlsl` 第 200–221 行对应 Indirect Specular From IBL，注释掉即可让指令数回到基线版本。

注意统计结论只在同一编译配置（同变体、同平台、同宏组合）下可比；跨批次的数据（例如上面 IBL 的两组数字）口径不同，不能直接相减。

### 合批与实例化

- 只需要 GPU Instancing 的 shader **不需要** SRP Batcher 兼容（10-27 要求）。植被（`Grass`、`MeshGrass`）属于这一类。
- 贴花（Decal）的优化点：8-25 记录「优化 Decal 性能 # 矩阵运算由片段着色器转移到顶点着色器」。
- 碰撞体可视化工具用「新建 GameObject + MeshFilter/MeshRenderer」绘制 Collider mesh（7-7 的实现方式）；源文未给性能结论。

### 工程与资源流程

- 美术工程（`ark_art`）与客户端工程（`ark_client`）分离；shader 改动需在美术工程打包 AssetBundle 并更新资源包才能在客户端验证（具体步骤见「变体与关键字管理」）。
- 客户端使用**裁剪后的变体集**，没有调试变体（8-18、10-15），这决定了很多验证必须在美术工程完成。
- 天气资源约定（1-10）：rain 与 snow 的资产位于 `Assets/Resources/WeatherSystem/RainSnow`，二者使用的 shader 都是 `Zeus/FloatingCreatures`；lightning（闪电）使用 `Custom/LightningBeta`。
- 12-14 记录处理了一个 SampleMSATex 问题（原文未记录现象与解法）。
- 地形高度/站立面检测工具的性能教训很有代表性：方案二「逐像素采样高度图」实现简单但 shader 分支重；方案三改为「每个物体取一次 position」，最初在脚本里遍历所有材质逐个赋值，结果**操作卡顿**（材质数量大 + 每帧遍历 + 属性设置开销）；最终改为「脚本查询一次地形高度，通过全局变量传给 shader」，既保留了逐物体的精度，又消除了每帧遍历。
- 高度图存储技巧：16 位 `ushort` 高度值保存为 HDR 图片（像素用浮点保存、数值允许超过 1.0），在单通道里携带地形高度，shader 侧一次采样即可。能否无损取决于导入后的实际格式：Unity 的 HDR 导入格式视平台/压缩设置而定（RGB9e5、RGBAHalf、BC6H、ASTC HDR 等），均不足以无损承载全量 ushort；只有 RGBAFloat / R32F 可以无损。

### 工具实现要点

碰撞体/站立面可视化（地编与控制台双入口）：

```text
- 开关控制 GameObject 自身的 Renderer（用 renderer.enabled，可反复开关）
- Collider mesh 通过新建 GameObject + MeshFilter + MeshRenderer 绘制
- 需要处理以 SceneRoot 开头的所有对象
- 按 Layer 过滤：CullDistance60 / CullDistance240 / CullDistance800 /
  CullDistanceInfinity / FOLIAGE / ENTITY_GLOBAL_VISIBLE /
  ENTITY_DISTANCE_200 / TERRAIN / ENTITY
```

着色规则（`NormalVisualization` shader）：按表面法线与地面法线的夹角上色 —— `[0°, 55°]` 绿色（可站立），`[55°, 100°]` 红色（不可站立/过陡），`[100°, 180°]` 黄色（朝下/侧面，另一版本中侧面统一为黄色、不达标为红色）。

三个版本的演进值得记录，因为它体现了「精度 vs 性能」的取舍：v1 加载本地地形 mesh 按高度着色；v2 用 ushort→HDR 高度图，在片元里采样算高度差并把着色叠加进 PBR forward pass；v3 放弃改 PBR，改用碰撞体 mesh 绘制，且高度差改为逐物体取一次 position（再用全局变量传给 shader）。

### 优化路径的判定标准

摸底全量 shader 时，先分类再决定是否投入改造成本，比逐个硬改高效得多：

| 类别 | 判定 | 处置 |
| --- | --- | --- |
| 只走 GPU Instancing（`Grass`、`MeshGrass`） | 每帧只有少数几个 drawcall | 不做 SRP Batcher 兼容 |
| 全场景只绘制一次（天空 `StandardSky`、`Planet`、`Cloud`、`Atmosphere`、`SunOrMoonDisk`） | 单实例 | 不做 SRP Batcher 兼容，优先砍指令 |
| 高频复用材质（角色、地形、贴花、特效、植被单体） | 同材质多实例 | 必须兼容 SRP Batcher |
| 无材质引用 / 已被替代（如早期的 `RippleShader`） | 检索不到引用 | 标记废弃后删除，别留着当变体负担 |

判定的关键在于「同材质是否会被多个 Renderer 复用」，而不是 shader 本身的复杂度。天空类 shader 即使指令很多，改 SRP Batcher 也几乎没有收益。

### 容器同步

两个需要保持一致的对象集合，如果用关联容器，求异同需要两次遍历（A 查 B、B 查 A）。改用「维护一棵与场景同布局的树」，一次遍历即可完成比对与增删，遍历次数约减半，渐进复杂度仍是 O(n)，且能保留层级信息用于定位。

## 其它明确结论

- **HSR（Hidden Surface Removal）预剔除的粒度是图元**（8-25 提问后的结论），不是 drawcall，也不是片元。
- 框架的 Tick 组织（12-10）：`Node` 类持有 `ITickable` 的引用；Node 组成一个链表，依次执行 Tick；`RendererController` 继承自 `ITickable`。
- Unity 编辑器已知问题（10-10）：修改 shader 后不能生效，仿佛函数内容的修改无法触发 shader 编译；当时通过修改函数名可以生效，暂时这样使用（**待核实**是否为特定版本 bug）。
- 调试输出开关的用法（9-13）：在适当的位置添加 `DEBUG_OUTPUT` 语句，然后在 Project 窗口找到任意一个 shader 文件，右键 TA 工具 `Generate Debuger`。
- Unity Hub 启动参数 `-force-gles32` 用来选择图形 API（8-23）。
- 后处理黑块先查 `discard`；材质为空先查 `InitViaReflection`；shader 改动不生效先查变体裁剪与 AB 更新 —— 这三类问题在源日志中反复出现。

## 原始笔记索引

- `杂项/log.md`（仅技术摘录，已排除进度、人事与私人内容）
