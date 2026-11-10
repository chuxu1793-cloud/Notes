# Parent.prefab 渲染队列（RenderQueue）与绘制顺序分析

## 1. 结论摘要

| 问题 | 结论 |
|---|---|
| 各材质 RenderQueue 不同，实际渲染中有区别吗？ | **有，而且是实质区别。** 队列不仅决定顺序，还决定物体进入"不透明"还是"透明"draw list。分界线是 **2500**。 |
| 2500 以内的对象绘制时有先后顺序吗？ | **有，且确定。** 队列是主排序键（升序），同队列内不再按距离排序。 |
| 队列值大小是否只是"谁先谁后"？ | 不止。**只有 2500 这条线跨阶段**，同侧内部的数值差异对真不透明物体仅影响性能（overdraw / early-Z），不影响画面。 |
| 常说的"分桶"指什么？ | 指 **`QuantizedFrontToBack` 对距离的量化分档**，不是 SortingLayer。详见 4.2。 |
| 本项目最需要注意的地方 | 屋顶（3110）是**不透明 Shader 却走透明通道**；水/草（2230/2245/2295）是**透明 Shader 却走不透明通道**。 |

---

## 2. 关键问题：两处队列错配

> 下文出现的队列值，均为材质资产上的 `m_CustomRenderQueue` 覆盖值，优先级高于 Shader 的 Queue tag，且**跨场景生效**。

### 2.1 屋顶 3110（780 个子网格）

- Shader 本身是 Opaque（ZWrite On、Blend Off），但队列 3110 > 2500 → 进入**透明列表**。
- 透明列表恒用 `CommonTransparent`（BackToFront），所以屋顶会被按距离从远到近排序。
- 因为 `m_RequireDepthTexture = True` 且 CopyDepth 在 AfterOpaques，透明列表的物体**不会写入 `_CameraDepthTexture`** → 依赖场景深度的效果（SSAO、景深、水面深度交互等）看不到屋顶。
- 视觉上屋顶颜色基本正确（自身写深度），代价是多余排序/批次 + 深度纹理缺失。

### 2.2 水面 2245 / 水底 2295 / 草地 2230

- Shader 是 Transparent（Alpha Blend、ZWrite Off），但队列被覆盖到 < 2500 → 进入**不透明列表**。
- 不透明列表在本机跳过距离排序 → 叠加顺序被队列固定，与摄像机距离无关。
- 绘制发生在天空盒之前且不写深度，背景为天空时可能被天空盒覆盖。

---

## 3. URP 如何决定列表内顺序

### 3.1 不透明与透明列表使用不同的排序标志

`Runtime/UniversalRenderPipeline.cs:1152` 附近：

```csharp
var commonOpaqueFlags      = SortingCriteria.CommonOpaque;
var noFrontToBackOpaqueFlags = SortingCriteria.SortingLayer | SortingCriteria.RenderQueue
                             | SortingCriteria.OptimizeStateChanges | SortingCriteria.CanvasOrder;

bool hasHSRGPU = SystemInfo.hasHiddenSurfaceRemovalOnGPU;
bool canSkipFrontToBackSorting =
       (baseCamera.opaqueSortMode == OpaqueSortMode.Default && hasHSRGPU)
    || baseCamera.opaqueSortMode == OpaqueSortMode.NoDistanceSort;

cameraData.defaultOpaqueSortFlags = canSkipFrontToBackSorting
    ? noFrontToBackOpaqueFlags : commonOpaqueFlags;
```

`Runtime/Passes/DrawObjectsPass.cs:198`：

```csharp
var sortFlags = (data.m_IsOpaque)
    ? renderingData.cameraData.defaultOpaqueSortFlags
    : SortingCriteria.CommonTransparent;   // 透明列表恒用，无 HSR 跳过分支
```

> 注意不对称：**只有不透明列表有"跳过距离排序"的分支，透明列表永远 BackToFront。**
> 本机 `HSR = True`、`opaqueSortMode = Default`，因此走跳过分支，不透明列表里距离键被整条移除（原因见附录 A）。

---

## 4. 排序键如何协同工作

这些键以 `SortingCriteria` 位标志组合后交给 `DrawRenderers`：**开了哪个 flag，排序键里就多一个比较字段**，引擎按固定字段顺序做字典序比较；位值大小与优先级无关。

### 4.1 各键职责

| 键 | 字段来源 | 作用 | 对 MeshRenderer 的现实意义 |
|---|---|---|---|
| SortingLayer | `Renderer.sortingLayerID` | 最粗的**分层**（离散值，无量化），与 2D/UI 共用层级 | 本 prefab 全在 Default，退化为常量 |
| RenderQueue | `Material.renderQueue`（覆盖优先，否则 Shader Queue tag） | 材质意图顺序，主排序键 | 决定 2210→2230→2245→2250→2295 |
| QuantizedFrontToBack / BackToFront / FrontToBack | 摄像机空间距离 | 距离键；`Quantized` 表示先把**距离量化成粗糙档位** | 量化让同档物体还能被下一步聚合 |
| OptimizeStateChanges | pass / 材质 / mesh 等状态 | 把状态相同的物体聚在一起，减少 SetPass | 所以同队列内不是"按提交顺序" |
| CanvasOrder | CanvasRenderer 绘制序号 | UI 顺序；普通 MeshRenderer 恒 0 | 3D 场景基本空转 |
| RendererPriority | MeshRenderer 的 Priority | 渲染器优先级 | 本项目 flags 未包含，不生效 |

### 4.2 "分桶"指的是按距离划出粗略档位

- "分桶"只发生在**距离**这一维，且只在开启 `QuantizedFrontToBack` 时存在；
- 档位的具体划分方式（步长、是否非线性）**引擎未公开**，文档只用 "rough" 描述，不能按固定步长推算。

**为什么要对距离量化**——这是它与 `OptimizeStateChanges` 的配对逻辑：

- 若按**精确距离**排序，场景里几乎每个物体的距离值都不同 → 距离键几乎永不相等 → 排在后面的 `OptimizeStateChanges` **永远轮不到生效**，材质被距离打散，**SetPass** 次数上升。
- 把距离量化成粗糙档位后，大量物体落进同一档、距离键相等 → 才轮到下一级按状态聚合 → 在"大致近的先生成"和"减少状态切换"之间取得平衡。
- 对照 `CommonTransparent` 用的是 `BackToFront`（精确距离）：透明混合正确性要求**严格从远到近**，`OptimizeStateChanges` 在这里只能做"距离相同"时的补位，不能破坏远近关系。

### 4.3 推导出的比较链

```
SortingLayer → RenderQueue → 距离（方向 + 是否量化） → 状态聚合/平局 → CanvasOrder
```

三条设计上的自证（由 URP 选择 flag 组合的方式反推）：

- **队列压过距离**：否则 2295 的水底就不可能稳定盖在 2250 的地面之后。
- **`QuantizedFrontToBack` 必须和 `OptimizeStateChanges` 配对**：精确距离排序会把材质打散，量化后"同档内按状态聚合"，在 early-Z 与批次之间取平衡。
- **距离键必须排在状态键之前**：否则 `CommonTransparent` 的 BackToFront 会被状态聚合破坏，透明混合出错。

---

## 5. 对 Parent.prefab 的直接结论

1. **2210 → 2230 → 2245 → 2250 → 2295 是固定顺序**，与摄像机位置无关。这些物体在不透明列表内，而本机（HSR）该列表不比较距离。
2. 草地(2230)、水面(2245)、水底(2295) 是 `ZWrite Off + Blend`，这个固定顺序**直接决定最终像素**：两层水永远按 2245 → 2295 叠加；因为不比较距离，绕到另一侧也不会翻转。**想调整叠放关系，唯一手段是改队列值。**
3. 屋顶(3110) 走透明列表，恒用 `CommonTransparent`（含 BackToFront），所以反而按距离从远到近排序；与不透明列表行为不对称，且不进入 `_CameraDepthTexture`。
4. 真不透明材质之间（2210 立面 / 2250 地面）的队列差异只影响 overdraw 与批次，不影响画面。

---

## 6. 若需修正（待确认，尚未执行）

| 目标 | 建议 | 风险 |
|---|---|---|
| 屋顶回到正常不透明流程 | 把 23 个屋顶材质的 `m_CustomRenderQueue` 改为 Shader 自身的 2450 | 属于材质资产级修改，影响所有引用场景；需确认没有别处依赖 3110 的"最后绘制"效果 |
| 水/草回到透明流程 | 改回 ~3000（Shader 默认队列） | 会改变现有叠放顺序，画面可能与当前美术预期不同 |
| 仅调整水面层叠 | 修改 2245 / 2295 两个队列值的相对大小 | 影响面较小，但仍需全场景验证 |

> 上述修改均未执行。改动前应先确认这些材质在其它 prefab / 场景中的使用情况。

---

## 附录 A：为什么 HSR GPU + Default 会跳过距离排序

```csharp
bool canSkipFrontToBackSorting =
       (baseCamera.opaqueSortMode == OpaqueSortMode.Default && hasHSRGPU)
    || baseCamera.opaqueSortMode == OpaqueSortMode.NoDistanceSort;
```

- `opaqueSortMode == Default` 的语义是"交给管线决定"，不是"一定排序"；`FrontToBack` 会强制排序，`NoDistanceSort` 无条件跳过。
- 前向排序的唯一目的是**填充率优化**：在传统 IMR 显卡上，近的先画能让远处像素靠 early-Z 提前失败，减少片元着色。代价是 CPU 排序。
- HSR（PowerVR 式 TBDR、Apple Silicon、Mali/Adreno 等）在每个 tile 内先解析深度关系，再只对最终可见片元着色——可见片元数由最终深度胜者决定，与提交顺序无关。既然换不来片元着色次数下降，排序就是纯 CPU 浪费。
- 本机 `SystemInfo.hasHiddenSurfaceRemovalOnGPU = True`（Metal / Apple Silicon），故走跳过分支；换到无 HSR 的桌面独显会自动回到 `CommonOpaque` 并恢复距离分档。
- 边界：HSR 的"顺序无关"只对不透明、深度测试 + 深度写入、无混合的几何成立。alpha test / discard 会破坏 HSR；而 URP 这个判断是**全局粗粒度**的，不区分具体材质。

---

## 附录 B：移动平台的 HSR 要点

### B.1 为什么移动端特别在意 HSR

移动 GPU 普遍是 **Tile-Based**（分块渲染）：屏幕切成小块（Mali 典型 16×16），每块在片上内存里完成几何 → 光栅 → 着色，最后才写回主存。

这带来两个约束：

- **带宽和功耗是首要瓶颈**，而不是算力。每个被着色的片元都要付出片上带宽与功耗代价。
- 因此"少着色被遮挡的片元"收益很大——这正是 HSR 的价值。

HSR（Hidden Surface Removal）在硬件层面先解析出**最终可见的片元**，再只对这些片元执行片元着色器。与 IMR 显卡的 early-Z 不同，它在整块 tile 的几何提交完成后就能确定可见性，因此**与绘制顺序基本无关**。

### B.2 各家的实现形态（代表性，细节不公开）

| 厂商 / 架构 | 渲染模型 | HSR 形式 | 备注 |
|---|---|---|---|
| PowerVR（Imagination） | TBDR | 真正的 per-tile 延迟着色 + HSR | HSR 概念的发源地 |
| Apple 自研 GPU（A/M 系列） | TBDR（同源思路） | HSR 默认参与 | Metal 下有 Memoryless 附件配合 |
| Arm Mali | Tile-based | **FPK**（Forward Pixel Kill） | Mali 计数器有 `Fragment FPK Active Cycles` |
| Qualcomm Adreno | Binning / Tile | **LRZ**（Low-Resolution Z） | 低分辨率深度预剔除 |

> 这些是厂商各自的实现，行为细节不对外公开。Unity 只用**一个布尔量** `SystemInfo.hasHiddenSurfaceRemovalOnGPU` 来概括，精度很粗。

### B.3 什么时候 HSR 会失效

HSR 只对"**不透明 + 深度测试 + 深度写入 + 不 discard**"的几何成立。以下状态会让它退化或完全失效：

- **开启混合（Blend）**——最常见，也是本项目水/草的情况；
- **ZWrite Off**（不写深度，硬件无法确定可见性）；
- **discard / alpha test / `clip()`**——在 PowerVR / Apple 上会直接打破 HSR；
- 在片元着色器里**写深度**（`SV_Depth`）；
- 部分实现中：**采样深度纹理**、模板操作、多渲染目标等。

注意：HSR 失效 ≠ early-Z 失效。深度测试本身仍然工作，只是"少着色"的收益消失。此时强行 front-to-back 仍可能减少被遮挡片元的着色，但收益取决于具体状态，需要实测。

### B.4 Unity / URP 如何使用这个信息

1. **跳过不透明列表的距离排序**（见附录 A）。URP Changelog 的原话：

   > Sorting of opaque objects disables front-to-back sorting flag, when camera settings allow that and the GPU has hidden surface removal.

2. **判断是全局的**：`hasHiddenSurfaceRemovalOnGPU` 只是设备级能力，URP **不会**逐个材质检查 Blend / discard。于是就存在最坏组合：

   > 因为设备有 HSR → 跳过距离排序（省了 CPU）；  
   > 但这些材质其实开着 Blend/discard → HSR 也用不上（没省 GPU）。  
   > **两头都没占到。**

3. **移动端带宽相关开关**：

   - `CopyDepthMode`：`AfterOpaques` / `AfterTransparents` 的选择会影响一次全屏深度拷贝，URP Changelog 明确提到这能带来移动端带宽节省；
   - Store Actions / MSAA resolve：移动端片上内存访问比主存便宜，尽量让附件留在 tile 内存里。

4. `Camera.opaqueSortMode` 是**手动覆盖**的入口：`FrontToBack` 会强制排序（无视 HSR），`NoDistanceSort` 强制跳过，`Default` 交给管线判断。

### B.5 对本项目的实际影响

| 对象 | 队列/状态 | 移动端会发生什么 |
|---|---|---|
| 地面 2250 / 立面 2210 | Opaque，ZWrite On | 正常享受 HSR；距离排序被跳过，但 HSR 使顺序无关 |
| 水面 2245 / 水底 2295 / 草地 2230 | **Blend + ZWrite Off**，却在 opaque 区间 | **HSR 失效 + 距离排序被跳过** → 双重损失，且叠加顺序被队列固定 |
| 屋顶 3110 | Opaque，但在透明列表 | 完全不参与不透明阶段的 HSR；且不进入 `_CameraDepthTexture` |

结论：移动端优化重点**不是**"要不要 front-to-back"，而是：

1. 把 **Blend + ZWrite Off** 的材质移出 opaque 区间（回到 2501+），让它们走该走的透明流程；
2. 减少这类全屏/大面积混合材质的**覆盖面积**（水体、草地往往是移动端 overdraw 大户）；
3. 屋顶若确认不需要"最后绘制"，回到 2450 可重新进入不透明阶段，拿到 HSR。

