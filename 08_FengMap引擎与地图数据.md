# FengMap 引擎与地图数据

蜂鸟（FengMap）室内地图内核的架构、渲染、坐标体系与主题配置笔记。内容以 `FMKernel` C++/OpenGL ES 实现为背景，涉及 `FMData`/`FMNode`/`FMRender` 三层对象模型、楼层与图层组织、VBO 与三角化、墨卡托坐标转换，以及 `.theme` 主题配置格式。

## 引擎架构与场景图

### 三层对象模型：Data → Node → Render

| 层次 | 基类 | 职责 |
|---|---|---|
| 数据 | `FMData` / `FMDataGroup` | 只描述几何与属性；`getType()` 返回数据类型，`getBoundingBox()` 返回包围盒 |
| 场景节点 | `FMNode` / `FMNodeGroup` | 维护父节点指针、本节点变换矩阵 `_matrix`、子节点数组 |
| 渲染对象 | `FMRender` / `FMRender20` | 维护 VBO、shader、纹理；基类持有一个 `FMNode*` 回指节点 |

要点：

- `FMDataGroup` 相比 `FMData` 增加 `std::vector<FMData*> _datas` 及管理函数；`FMNodeGroup` 与之一一对应，持有一个 `FMNode` 数组。
- `FMNode` 含指向父节点的指针。**计算当前节点的模型变换矩阵时，必须把所有父节点的变换矩阵叠加进来**，并继续向下传递。
- `Data` 传入 `Node`，`Node` 传入 `Render`；渲染时 `render()` 调用子节点的 `draw()`，`draw()` 再调用子节点的 `render()`，整棵树递归遍历。
- `FMPlane` 不是独立数据类型，只是一个过渡类，用于承载平面顶点数组。

节点类举例：

- `FMLineNode`：线描述与绘制，包含线类型（点线/中心线）、线模式（像素宽度/空间宽度）、平滑模式（各种样条曲线）、深度模式（是否做深度剔除）。
- `FMPlaneNode`：按“面”数据绘制场景中的面。
- `FMMaskNode`：高亮/遮罩类绘制（原稿标注为猜测）。
- `FMTextureNode`、`FMCompassNode`（指南针）、`FMWaterMarkNode`（水印）（原稿推测）。

### shader 的类型划分

节点类型与数据类型一一对应，但 shader 类型是重新划分的：

```text
BASIC, MODEL, TEXTURE, GEOMETRY, LINE, EXTENT,
TEXT, POLYGON, POLYGON_MARK, POLYGON_WATER,
LINE_SPECIAL, GLTF
```

（原稿写作 `BASEIC`、`LINE_SPECAIL`，已更正为 `BASIC`、`LINE_SPECIAL`。）

一个 context 对应一组 shader。context 本身只是一个 `void*`，没有专门维护它的类（待核实）；创建 shader 时需要它来确定“在哪个 context 中”创建（待核实）。

### 标识符规则

- `int _id` 是 `FMData` 的属性；`int _gid` 是楼层的属性。
- 楼层（`FMLayerGroup`）用 `_gid` + `_id` 标识。
- 图层（`FMLayer`）只用 `_id` 标识。
- 模型 / 外部模型用 `_id` + `_eid` 作为唯一标识。
- 多楼层显示状态 `MultiDisplayStatus`：楼层 ID + 该楼层透明度。

### 数据载入流程

```text
scene 数据 → 按楼层加载 FMLayerGroup
  每个楼层 → 按图层加载 FMLayer
    每个图层 → 按图层类型调用对应的 load 函数
```

图层类型见后文“图层类型与 load 函数”。分层加载（按需加载）时：

1. 先只加载根节点数据与默认显示楼层，创建 Node 和 Render；
2. `view::setMultiDisplay` 触发目标楼层的数据加载，并同步创建 Node、Render，Data 与 Node 都要挂进场景图；
3. 每增加一层都会扩大包围盒，因此 `fullScale`（比例尺/全屏缩放）必须跟着更新；
4. 主题效果也要改成可动态添加。

### initScene 流程

```text
a. 载入 fmap 数据，生成 FMData
b. 由 FMData 生成场景图（组节点 + 叶节点）
c. 由场景图为每个节点创建 render 对象
d. view 设置根节点
e. 用比例尺等级设置缩放范围
```

其中场景图与渲染对象的创建：

- `FMNodeCreator::createNode` 负责查找支持该数据类型的**适配器**，再调用适配器的 `constructFMNode`。适配器对象是全局的。对 layer 类型的数据，要遍历其子数据递归 create。
- `FMNode::initRender`：组节点先 `FMCreateRender`，再对每个子节点逐个 `initRender`；非组节点直接 `FMCreateRender`。
- `FMRenderCreator::createRender` 同样是“查适配器 → `constructFMRender`”。

`view` 设置根节点的步骤：

```text
a. 场景图中各节点设置 View 对象
b. 多楼层显示设置（initMuitiDisplayStatus / setLayerGroupDisplay / setLayerGroupVisible）
c. 初始化矩阵参数：按楼层扩充包围盒，用包围盒中心初始化平移向量
d. 建立全局模型变换矩阵（setupModelMatrix / updateSceneNodeMatrix）
e. 把全局模型变换矩阵更新到场景图
```

### 一帧的渲染流程（FMView20::render）

1. 建立全局顶点变换矩阵：初始场景处理（平移到原点、缩放、旋转）+ 手势操作产生的变换。（注：源两处记述不一致，以实现步骤（旋转在缩放前）为准，见后文“顶点变换顺序”。）
2. 全局矩阵叠加投影变换得到 MVP，分别传给主场景图与 HUD 场景图。
   - 组节点先叠加自身的矩阵，再向下传给所有子节点。
   - 应用视锥体裁剪。
3. 执行 `draw()`，按场景树递归绘制。

### 延迟初始化（lazyInit）

- `initRender` 在 `initScene` 阶段执行，只负责创建 render 对象。
- shader 与 VBO 的创建推迟到**第一次执行 render 时**，由 `lazzyInit()` 完成，且只执行一次。
- `FMRender20::lazzyInit()`：`initShader()`（各派生类重写以创建不同类型的 shader）→ `initBuffers()`（创建 VBO 并上传顶点数据）。
- `FMGLTFModelNodeRender20::lazzyInit()`：`initShader()` → `initShaderExt()` → `initBuffers()` → `initTexture()`。

这样切楼层、切主题后可以只重建必要资源，避免初始化阶段一次性加载全部 GPU 资源（推断）。

## 图层类型与节点

### 楼层即 FMLayerGroup

一个楼层（`FMLayerGroupNode`）包含以下图层：

| 图层 | 容器形态 |
|---|---|
| `FMPointImageLayer` | 数组 |
| `FMPointTextLayer` | 数组 |
| `FMPolygonLayer` | 数组 |
| `FMPolygonMarkLayer` | 数组 |
| `FMMaskPolygonLayer` | 数组 |
| `FMLabelLayer` | 单变量 |
| `FMFacilityLayer` | 单变量 |

**每个楼层只有一个 `labelLayer` 和一个 `facilityLayer`。**

### 图层类型与 load 函数

| 图层类型 | 加载函数 | 生成的数据类 |
|---|---|---|
| `polygon_store` | `loadFMModelGroup` | `FMModel : FMPlane : FMData`（主要属性是 `FMPlane` 的顶点数组） |
| `point_storelabel` | `loadFMLabelGroup` | `FMLabel`（主要属性是位置坐标） |
| `point_poi` | `loadFMFacilityGroup` | `FMFacility`（主要属性是位置坐标） |
| `polygon_extent` | `loadFMExtentGroup` | `FMExtent : FMPlane : FMData` |
| `polygon` | `loadFMPolygonGroup` | `FMPolygon : FMPlane : FMData` |
| `polygon_label` | `loadFMPolygonLabelGroup` | `FMPolygonLabel : FMLabel`（主要属性是位置坐标） |
| `point_model` | `loadFMExternalModelGroup` | `FMExternalModel : FMData`（主要属性是位置坐标） |

（原稿将 `FMLabel` 记作 `public FMDataGroup`，但此处描述“主要属性是位置坐标”，与单个数据对象更吻合，待核实。）

`FMLayer` 自身是 `public FMDataGroup`，所以加载函数实际是把该图层下的所有元素一次性建成数据对象。

### 场景图与渲染顺序

`FMSceneNodeRender20` 持有 `_lineLayers`、`_layerGroups`、`_locationLayers` 三个数组，初始化时由 `setupLayers()` 查找填充。绘制顺序：

```text
_lineLayers → _layerGroups → _locationLayers
```

每个 `_layerGroups`（楼层）内部的绘制顺序：

```text
_extentLayers        底板
_modelLayers         模型
_polygonLayers       多边形
_externalModelLayers 外部模型
_facilityLayers      设施
_labelLayers         标签
_pointImageLayers    点图
_pointTextLayers     文字
_polygonLabelLayers  多边形标签
_polygonMarkLayers   多边形标记
```

`_modelLayers` 本身没有 shader，需要对其上的 `modelNode` 分通道绘制：

```cpp
drawNodes(0);  // 线框
drawNodes(1);  // 平面
drawNodes(2);  // image
```

### 外部模型（glTF）链路

```text
FMExternalModelLayer
  └─ FMExternalModelNode (NodeGroup)   // 维护全部矩阵变换
       └─ FMGLTFModelNode (Node)       // 仅包含 glTF 数据，无其他操作
```

`FMExternalModel` 本身是 `public FMData`。

glTF 侧有两个实现假设，直接限制了多材质模型的渲染效果：

- **`materials` 数组只取一个元素**；
- **假定所有 primitive 共用同一个 material，且各 primitive 的属性相同**。

主题相关：

- `FMExternalModelTheme`：`_id`（主题 ID）、`_color`（颜色）、`_model`（对应 glTF 模型名）。
- `FMThemeAdaptor` 维护各主题字典、apply 字典、批量设置字典，外部模型字典为 `std::map<int, FMExternalModelTheme*> _externalModelThemes`。

初始化流程（只在初始化阶段执行一次，正式渲染后不再执行）：

```text
1. FMThemeAdaptor::adapt(FMView* view)
2. adaptExternalModelTheme(view->getSceneNode())
   a. adaptExternalModelLayer()               // 用 FMNodeFinderByType 遍历场景图，收集所有外部模型图层
   b. FMExternalModelNode::setTheme(FMTheme*) // 按 apply 索引查字典，按 glTF 文件名加载/取模型
        1) applyTheme(theme)                  // 载入 glTF 数据、生成节点、加入外部模型节点、初始化渲染器
3. FMExternalModelNode::setThemeMatrix(translate, rotate, scale)
   1) updateMatrix()        // 更新节点矩阵
   2) updateRenderMatrix()  // 更新渲染矩阵
```

注意：`FMExternalModelNode` 作为组节点维护的矩阵会向下传给子节点，但并非所有组节点的矩阵都一定生效（原稿对此存疑）。

### 节点删除与延迟释放

从场景图删除一个节点包含三部分：

1. 节点本身；
2. 节点持有的数据；
3. 节点持有的 render。

渲染器**不能在绘制过程中立刻销毁**，由单例 `FMRenderReleaser` 延迟释放：

```text
1) addViewRender() 把待删 render 加入 std::map<FMView*, std::vector<FMRender*>> _viewRenders
2) 每帧 beforeDraw 调用 FMReleaseViewRender()，统一删除上一帧登记的所有渲染器
```

即：删除只标记，真正的 GL 资源释放推迟到下一帧绘制之前。

### 标签与设施的遮挡

- 设施（facility）与标签（label）的遮挡处理逻辑相似，但 Facility 更简单：**没有按 `minLevel` 排序**，因此会出现少量错误，但影响不大。
- 判断顺序必须是：**先做等级判断 `isShow`，再做遮挡判断 `cover`**。反过来（先 cover 后 isShow）是错的。
- 遮挡包含两个判断：
  1. 是否超出视口：按屏幕坐标计算，图标中心点超出屏幕区域即标记为遮挡；
  2. 与“标签池”中的标签逐个比较。标签池 = 当前未被遮挡、将要显示的标签集合。

## 渲染

### 模型 VBO 布局

`FMModelNodeRender20::initBuffers()`：

1. 组装顶点数据，每个顶点固定 9 个 float，紧密排列：

```text
[VX VY VZ CR CG CB CA TX TY] [VX VY VZ CR CG CB CA TX TY] ...
```

2. 创建 VBO，把顶点数据传入 GPU。
3. 创建索引 VBO，拿到“每一面墙的顶点索引数组”和“每一条顶面边线的顶点索引数组”的内存起始地址。
4. 预留总索引缓冲区空间，按 **底面 → 墙 → 顶面边线** 的顺序依次上传。

`_vbos[MODELR20_VBO_SIDEINDEX]` 一个值承担两种含义：既是墙的顶点索引起始地址，又是底面顶点索引的数量——因为索引按“底面 → 墙”连续存放，墙的起始偏移正好等于底面索引数量（推断）。

### 带洞多边形的三角化（FMModel::triangulateProcess）

核心文件：`geometry_p2t.h` / `geometry_p2t.cc`，关键函数 `makeVerticesCCW`、`TriangulateProcess`、`TriangulateProcessWithHoles`。

流程：

```text
1. 对 fmap 多边形做三角化（分无洞、有洞两种情况）
2. 处理洞：把洞的顶点数据追加到已处理顶点数组之后（此时仍是平面上的点）
3. 计算模型高度
4. 复制顶点数据，复制顺序为 顶面、底面、顶面、底面
5. 设置顶面边线索引数组，以及所有洞的顶面边线索引数组
   note: _lineIndexs 的数量 = 要绘制的边线对象数量
6. 设置墙面索引数组，以及所有洞的墙面索引数组
   note: n 个顶点 n 条边，每条边一面墙，每面墙 6 个顶点，_sideIndexs 的数量 = 外墙 + 洞墙的绘制对象数量
```

结论：

- 顶面三角化**不会产生新顶点**（原稿确认）。
- 模型顶面顶点排列必须是**逆时针**。
- 每个顶点都必须配 UV 坐标。

### polygon offset 与 z-fighting

- 线与面结合处的闪烁，通过打开 **Polygon Offset** 解决。
- 同类问题：`setfitview` 后出现的 z-fighting（易通行项目）需要专门处理。
- 绘制模型时把面剔除关掉，可以得到“只显示线框 / 全透明”的效果（作为线框模式的实现手段）。
- 线框模式有三种显示样式：**楼层模式**（含底板与模型）、**全模型模式**、**单模型模式**（原稿 2018/06/01 的计划中记录，实现情况未见后续结论）。

### FMLine 扩边

`FMLineNode::lazzyInit()` → `init()` → `constructVertices()` → `constructGeometryPlaneVerticesForOuterColor()`。

`constructGeometryPlaneVerticesForOuterColor` 的作用：根据折线顶点，生成新的顶点、纹理坐标和顶点索引，**新生成的两个顶点位于原顶点的两侧**，从而把一条数学意义上的线扩成有宽度的面片（外扩部分用于描边/外发光颜色）。

约束：**线的类型和线的模式必须一一对应**（点线/中心线 × 像素宽度/空间宽度）。

线数据可以分成两段顶点数组，调用两次 `draw()`，但共用同一个 program。

### 特殊线与普通线的 program 切换

切换发生在 `FMLineLayer` 中：

```cpp
glUseProgram(_shader20->getProgram());
glUseProgram(_lineSpecialShader->getProgram());
```

### 导航线 fragment shader 与 UV 循环

```glsl
uniform vec4 interColor;
uniform vec4 outerColor;
uniform float stepLen;      // getLineTexSperateRatio
uniform sampler2D uSampler;
varying vec2 vUv;

void main()
{
    vec4 color;
    if (vUv.x < 0.0 || vUv.x > 1.0)
    {
        color = outerColor;   // 线宽之外：外扩颜色
    }
    else
    {
        color = interColor;   // 线宽之内：内部颜色
    }

    vec4 texColor = texture2D(uSampler, vec2(vUv.x, 1.0 - mod(vUv.y, stepLen)));
    gl_FragColor = mix(color, texColor, texColor.w);
    // 等价于 color * (1 - texColor.w) + texColor * texColor.w
}
```

（上述代码由原稿分段片段按原逻辑整理成完整 shader，逻辑未改。）

注意点：

1. `mod` 求余实现纹理沿线路方向的循环，余数范围 `[0, stepLen)`（原稿有误，已更正：闭区间 → 左闭右开），`stepLen` 即循环步长。
2. 写成 `1.0 - mod(vUv.y, stepLen)` 是因为**顶点 UV 的生长方向与纹理图片的纹理坐标方向相反**。
3. 纹理参数必须设为 `GL_CLAMP_TO_EDGE`，不能想当然地用 `GL_REPEAT`。循环已经由 shader 的 `mod` 完成，采样坐标始终落在 `[0,1]` 内；用 `REPEAT` 会在 1.0 边界处采到另一端的像素，产生接缝（推断）。
4. 纹理过滤参数需注意平台差异：`GL_NEAREST_MIPMAP_LINEAR` 在某些 OpenGL 版本/环境下会导致贴图失败，改成 `GL_LINEAR` 即可正常（原稿判断为 OpenGL 版本问题）。

### 文字渲染

三层次分工：

| 类 | 职责 |
|---|---|
| `FMPointTextLayer` | 提供 `RenderMatrix` 与 `BillboardMatrix`（布告板矩阵）、全局 MVP |
| `FMPointTextNode` | 保存文字位置信息，提供 `Matrix`（平移矩阵，表示在 layer 中的位置） |
| `FMTextNode` | 图形顶点坐标、纹理坐标、颜色（填充/边线/透明）、`PerspectiveMatrix`、`Matrix`/`TextMatrix` |

- `FMPointTextLayer`、`FMPointTextNode`、`FMTextNode` **共用一个 shader**。
- `FMTextNode` 的 `PerspectiveMatrix`：从 `renderMatrix` 中提取缩放矩阵，据此计算图片的缩放因子；`TextMatrix` 承载手势操作导致的绕 Z 轴旋转。
- 文字颜色可变的原因：文字纹理的像素值只作为**权值**，用来调制外部设置的颜色。
- label 文字的字体、颜色、大小由主题决定。
- `FMTextNode` 通过 `addNode` 挂到 `FMPointTextNode` 上；`FMPointTextNode::changeText` 需要判断传入的是否为字符串数组，以支持多行显示。

### OpenGL 图元类型速查

| 图元 | 含义 |
|---|---|
| `GL_TRIANGLES` | 独立三角形，每 3 个顶点一个 |
| `GL_TRIANGLE_STRIP` | 三角形条带，012/123/234… 各成一个三角形 |
| `GL_TRIANGLE_FAN` | 三角形扇面，012/023/034… 各成一个三角形，可表达任意凸多边形 |
| `GL_LINES` | 每两个顶点一条线段（01、23、45…） |
| `GL_LINE_STRIP` | 所有点依次连接成一条折线 |

## 坐标与变换

### layerNodeCoord 与 mapCoord

| 名称 | 含义 |
|---|---|
| `layerNodeCoord`（楼层坐标） | 相对于**包围盒中心**的相对坐标 |
| `mapCoord`（地图坐标） | 单位为**米**的墨卡托坐标 |

相关转换函数：`FMConvertMapCoordsToLayerNodeCoords`、`FMConvertMapCoordToLayerGroupNodeCoord`、`calcIntersectMapCoord`（由屏幕触点求地图坐标）。

### 数据中心 vs 包围盒中心

场景数据有一个中心 `scene->_origin`，**所有顶点数据都是相对于该中心的坐标**，采用相对坐标是为了提高精度：坐标值乘以 1000，即由米换算到毫米（1 m = 1000 mm）。需要注意，×1000 本身只是换单位，float 的相对精度并不因此改变；真正的定标收益来自配合整数/定点存储，以及以大数中心为原点带来的数值范围收窄。

注意这两个中心不是同一个东西：

- 顶点数据相对的：数据中心 `scene->_origin`；
- 全局矩阵变换首先平移到的：**包围盒中心**。

模型拾取的交点究竟用哪个中心，原稿存疑（待核实）。

### 顶点变换顺序

```text
平移到原点（附加手势操作的 XY 增量；此 XY 是相机坐标系的 XY，不是模型的 XY）
  → 旋转（手势操作）
  → 缩放到宽度 [-1, 1] 以适配全屏（附加手势增量）
  → 远离相机的平移
  → 相机后移，保证全屏显示（无手势操作可保证全屏，有手势增量则不保证）
  → 透视投影
```

要点：

- **每次矩阵变换都先平移到原点。**
- 顶点与 MVP 相乘得到裁剪坐标系下的位置；**方向向量与 MVP 相乘只受 model-view 矩阵中的旋转和缩放部分影响**（平移不影响方向）。完整 MVP 会被投影影响（透视除法通过 w 改变结果），所以法线要用逆转置矩阵变换。
- 全局变换采用的是“基于初始姿态先平移、再旋转、最后缩放”的写法。

### Y/Z 轴互换

初始化时全局变换矩阵就附加了一个绕 X 轴（`_inclineAngle`）旋转 **-90°** 的变换，因此一开始 XY 平面就被“放倒”了，即完成 Z-up 到 Y-up 的转换。

- `_rotateAngle`：绕 Z 轴（水平旋转）的角度。
- `_inclineAngle`：绕 X 轴（俯仰）的角度。

### FMView::translate 手势平移

`void FMView::translate(Vec2f start, Vec2f end)`：

```text
1) 由起点、终点的屏幕坐标（像素）算出偏移向量
2) 偏移向量从屏幕坐标转成 OGL 坐标
3) 偏移向量除以缩放系数，得到场景中等比例的物体偏移量
4) 偏移向量乘以当前旋转矩阵的逆矩阵，保证 x 增量指向右、y 增量指向上/屏幕里
5) 把偏移向量累加到全局变换使用的 _translate 上
```

第 4 步必须乘旋转矩阵的逆矩阵：拖动的效果是在**当前姿态**上展现的，但偏移量的叠加是在**初始姿态**基础上进行的，类似 billboard 的思路。另外，手势平移只改变场景中楼层 X、Y 方向的增量。

```cpp
_scnenOrigin = _currOBB._position;
_translate = -_scnenOrigin;
```

`_scnenOrigin.z()` 表示 Z 轴的值，单位为米。

### 全屏适配与透视投影

- 顶点数据初始位于 Z 轴正半轴（原稿示例为 50，实际随数据不同；原稿此处有删除线）。
- 先把最长边缩放到长度为 2。
- 透视投影的视锥体很小：近裁剪面**高 0.06495191、宽 0.11547006**，near = 0.1，far = 100。
- 因此不能靠放大模型铺满屏幕，而要用**相机后移**的办法让物体布满屏幕。

`_fullScale` 是 OGL 视口与模型的比值：OGL 视口宽 2、高 `ar * 2`（ar 为宽高比）；模型宽度取最大宽度（对角线方向），高度取实际高度，单位均为米。

透视投影前为什么会额外加一个沿 Z 轴负方向的增量，原稿未给出结论（待核实）。

### 缩放等级与比例尺

- 缩放等级在 `FMView` 构造函数中已经设置，`FMScene` 中也有 `_scaleLevels`。原稿暂注释掉 `FMScene` 的 `_scaleLevels`，实际取值待核实。
- `initMatrixParameters` 的初始缩放倍数依据**整个场景图的包围盒**，与当前显示哪些楼层无关。
- 比例尺的作用主要是判断 facility / label 是否显示；`FMModelNode`、`FMFacilityNode`、`FMLabelNode` 都只在各自指定的比例尺等级区间内显示，超出范围就不绘制。

### 包围盒的坑

- 包围盒在载入数据时就会创建。
- **场景旋转会导致 AABB 失效**，因此需要 OBB（有向包围盒）。
- 用 `setMultiDisplay` 设置显示楼层时，如果显示的楼层数大于模型本身的楼层数，会补出默认参数（0）的空楼层，导致整体包围盒中心计算错误，模型偏离正确位置。

### 拾取

模型拾取按“视点射线与模型包围盒的交点”判断，因此**不需要面也能选中**。副作用是：隐藏的模型如果只改了可见性、没有从拾取集合里剔除，仍然可以被选中（项目 BUG）。另外，2D 模式下可以通过减少绘制面元来提升性能。

### 墨卡托投影

Web Mercator（Spherical Mercator）把地球当作**球体**而非椭球体，与常规墨卡托投影的区别仅此一点；理论精度差别在 0.33% 以内，比例尺越大越可忽略。

特性：

- **等角**：形状不变，正方形投影后仍是正方形；方向和相互位置正确，因此导航计算方向不会出错。
- **圆柱**：经线、纬线都是平行直线且互相垂直，经线间隔相同，纬线间隔从标准纬线向两极逐渐增大。
- 代价是面积变形，越靠近两极越夸张。

坐标范围（以赤道为标准纬线、本初子午线为中央经线，交点即原点，向东向北为正）：

$$R = 6378137\ \text{m},\quad X_{\max} = \pi R = 20037508.3427892\ \text{m}$$

$$x \in [-20037508.3427892,\ 20037508.3427892]$$

Y 轴在纬度趋近 90° 时趋于无穷，工程上同样截断到 $\pm 20037508.3427892$，使投影平面成为正方形（便于预切静态瓦片）。反算得到对应纬度：

$$\varphi_{\max} = 85.05112877980659^\circ$$

因此地理坐标范围为经度 $[-180, 180]$、纬度 $[-85.05112877980659, 85.05112877980659]$。

经纬度与墨卡托互转：

$$x = \text{lon} \times \frac{20037508.34}{180}$$

$$y = \ln\left(\tan\left(\frac{(90 + \text{lat})\pi}{360}\right)\right) \times \frac{20037508.34}{\pi}$$

（原稿有误，已更正：原式乘的是 $20037508.34/180$，漏掉了 $180/\pi$ 的弧度换算。下面的 C# 实现本身是对的——`Math.Log(...)/(π/180)` 已经含 $180/\pi$，再乘 $20037508.34/180$，与乘 $20037508.34/\pi$ 等价。）

反算时先做与正算对称的归一化，把米换算回“角度制”的中间量：

$$y' = \frac{Y}{20037508.34} \times 180$$

再把 $y'$ 代入反正切（$y'$ 是角度值，故指数项写作 $\exp(y'\pi/180)$）：

$$\text{lon} = \frac{x}{20037508.34} \times 180,\qquad
\text{lat} = \frac{180}{\pi}\left(2\arctan\left(e^{\,y'\pi/180}\right) - \frac{\pi}{2}\right)$$

```csharp
// 经纬度转墨卡托
public Vector2D lonLat2Mercator(Vector2D lonLat)
{
    Vector2D mercator = new Vector2D();
    double x = lonLat.X * 20037508.34 / 180;
    double y = Math.Log(Math.Tan((90 + lonLat.Y) * Math.PI / 360)) / (Math.PI / 180);
    y = y * 20037508.34 / 180;
    mercator.X = x;
    mercator.Y = y;
    return mercator;
}

// 墨卡托转经纬度
public Vector2D Mercator2lonLat(Vector2D mercator)
{
    Vector2D lonLat = new Vector2D();
    double x = mercator.X / 20037508.34 * 180;
    double y = mercator.Y / 20037508.34 * 180;
    y = 180 / Math.PI * (2 * Math.Atan(Math.Exp(y * Math.PI / 180)) - Math.PI / 2);
    lonLat.X = x;
    lonLat.Y = y;
    return lonLat;
}
```

### dpi 与比例尺

- dpi（dots per inch）= 每英寸的像素点数；1 英寸 = 2.54 厘米。
- 所以 `dpi / 2.54` 表示 **1 厘米内的像素点数**（像素/厘米）。
- 比例尺 = 图上距离 / 实际距离。

由此，若屏幕上量得 $p$ 像素、对应实际距离 $D$（厘米），则：

$$\text{图上距离} = \frac{p}{\text{dpi}} \times 2.54\ \text{cm},\qquad
\text{比例尺} = \frac{p \times 2.54}{\text{dpi} \times D}$$

## 配置文件与主题

### 基本事实

- `.theme` 文件是 JSON。fmap 数据内部只包含顶点位置信息；**颜色和纹理信息全部来自主题文件**。
- 主题按作用对象分区：`viewtheme`（背景颜色）、`extenttheme`（楼层地面）、`storetheme`（店铺模型）、`storelabeltheme`（店铺文字标识）。
- 主题在 `applyTheme` 阶段把纹理图片名称等赋给数据对象（如 `FMImage` 只维护一个 string，保存来自主题配置的图片名）。
- 模型相关主题项：`storetheme`（模型主题字典）、`floorapply`（按楼层设置）、`storeapply`（按单个商户设置），另有 label 侧的 `storelabeltheme` / `floorlabelapply` / `storelabelapply`。
- 表格中的字段名带 `<块名>_` 前缀，只是为了说明归属；**实际 JSON 是嵌套结构**（`storetheme` 是数组，元素内是 `id`/`color`/…；`floorapply` 元素内是 `floorname` + `typeapply` 数组）（待核实）。

### 店铺模型：storetheme 字段

| 字段 | 含义 | 取值 |
|---|---|---|
| `id` | 配色方案唯一标识 | 大于 0 的整数 |
| `color` | 模型显示颜色 | RGB(0~255)，逗号分隔 |
| `alpha` | 模型不透明度 | 0.0~1.0 |
| `strokecolor` | 轮廓线颜色 | RGB(0~255)，逗号分隔 |
| `strokewidth` | 轮廓线宽度 | 大于 0、小于 10 |

（原稿字段写作 `storethem_alpha`，应为 `storetheme_alpha` 的笔误，已统一。）

### 应用规则：floorapply / typeapply / storeapply

| 字段 | 含义 | 取值 |
|---|---|---|
| `floorapply.floorname` | 楼层名称/id | `"0"` 为缺省设置 |
| `typeapply.type` | 该配色方案适用的店铺类型 | 6 位十进制类型码，逗号分隔，`0` 为缺省 |
| `typeapply.normalid` | 正常状态配色方案标识 | 取 `storetheme.id` |
| `typeapply.selectedid` | 选中状态配色方案标识 | 取 `storetheme.id` |
| `storeapply.floorname` | 店铺所在楼层 | 无 `0` 缺省 |
| `storeapply.nameapply.storename` | 店铺名称/id | FengMap sid，可逗号分隔多个 |
| `storeapply.nameapply.normalid` | 正常状态配色方案标识 | 取 `storetheme.id` |
| `storeapply.nameapply.selectedid` | 选中状态配色方案标识 | 取 `storetheme.id` |

优先级语义（推断，待核实）：`floorapply` 按楼层批量套用（`floorname = "0"` 是全局缺省），`storeapply` 针对具体商户覆盖 `floorapply` 的结果；同一层内 `typeapply` 按店铺类型匹配，命不中则回退到缺省。

### 店铺模型配置示例

> 以下 JSON 据原 `.tif` 截图转写（截图裁掉了外层花括号，这里补齐为完整对象）；字段值与缩进可能和实际文件有出入。

```json
{
  "storetheme": [
    {
      "id": "1",
      "color": "255,0,0",
      "alpha": "0.8",
      "strokecolor": "0,0,0",
      "strokewidth": "2.0"
    },
    { "id": "2" },
    { "id": "3" }
  ],
  "floorapply": [
    {
      "floorname": "0",
      "typeapply": [
        {
          "type": "170000,170100,170200,170300,170400",
          "normalid": "1",
          "selectedid": "2"
        },
        {}
      ]
    },
    { "floorname": "1" }
  ],
  "storeapply": [
    {
      "floorname": "1",
      "nameapply": [
        {
          "storename": "103329948,103329949,103329950,103329951,103329952",
          "normalid": "1",
          "selectedid": "2"
        },
        {}
      ]
    },
    {}
  ]
}
```

只写 `id` 而不写其他字段的方案，表示其余字段沿用缺省值（待核实）。

### 店铺文字标识：storelabeltheme 字段

| 字段 | 含义 | 取值 |
|---|---|---|
| `id` | 配色方案唯一标识 | 大于 0 的整数 |
| `fillcolor` | 文字填充色 | RGB(0~255)，逗号分隔 |
| `strokecolor` | 文字边线色 | RGB(0~255)，逗号分隔 |
| `alpha` | 文字不透明度 | 0.0~1.0 |
| `fontsize` | 字号 | 屏幕点坐标 |

（原稿写作 `storelabelthem_alpha`，同样为笔误。）

应用端字段与模型侧同构：`floorlabelapply_floorname` / `typeapply_type` / `normalid` / `selectedid`，以及 `storelabelapply_floorname` / `nameapply_storename` / `normalid` / `selectedid`。

> 以下 JSON 据原 `.tif` 截图转写（截图裁掉了外层花括号，这里补齐为完整对象）；字段值与缩进可能和实际文件有出入。

```json
{
  "storelabeltheme": [
    {
      "id": "1",
      "fillcolor": "0,0,0",
      "strokecolor": "255,255,255",
      "alpha": "1.0",
      "fontsize": "14.0"
    },
    { "id": "2" },
    { "id": "3" }
  ],
  "floorlabelapply": [
    {
      "floorname": "0",
      "typeapply": [
        {
          "type": "170000,170100,170200,170300,170400",
          "normalid": "1",
          "selectedid": "2"
        },
        {}
      ]
    },
    { "floorname": "1" }
  ],
  "storelabelapply": [
    {
      "floorname": "1",
      "nameapply": [
        {
          "storename": "103329948,103329949,103329950,103329951,103329952",
          "normalid": "1",
          "selectedid": "2"
        },
        {}
      ]
    },
    {}
  ]
}
```

同一份说明文档另附了等价的 XML 写法（字段一一对应，`typeapply` 可写成多个并列节点），以下据截图转写。它是否为“早期版本”格式、当前引擎是否仍支持，原稿未说明（待核实）：

```xml
<storelabeltheme id="1" fillcolor="0,0,0" strokecolor="255,255,255" alpha="1.0" fontsize="14.0"/>
<storelabeltheme id="2"/>
<storelabeltheme id="3"/>

<floorlabelapply floorname="0">
  <typeapply type="170000,170100,170200,170300,170400" normalid="1" selectedid="2"/>
  <typeapply type="180000,180100" normalid="2" selectedid="3"/>
</floorlabelapply>
<floorlabelapply floorname="1"/>

<storelabelapply floorname="1">
  <nameapply storename="103329948,103329949,103329950,103329951,103329952" normalid="1" selectedid="2"/>
</storelabelapply>
```

### 模型顶面贴图与 UV

顶面贴图不是“把纹理贴到模型上”，而是**在顶面单独画一个矩形**贴图，所以 ModelShader 里比较乱（它同时包含线、顶面、侧面、热力图、顶部图、选中贴图几套逻辑）。

UV 生成思路：

```text
1. 先把顶面按相邻边“展开”到平面：以相邻顶点连线为旋转轴，对相邻两顶点做 -90° 旋转，
   得到新的顶点坐标；收集所有展开后的顶点，求包围矩形（对应纹理坐标空间）
2. 对每个原始顶点计算它在包围矩形中的相对位置，即得 UV
```

要点：

- 每个顶点都必须有 UV。
- 顶面三角化不产生新顶点，不需要为 UV 额外加点。
- 贴图发生在初始化阶段，数据本身不因旋转而改变，因此“先旋转还是先应用主题”不影响贴图结果。

## 工程经验

### 平台与数据格式

- **iOS 像素颜色按 BGR 存储**，直接按 RGB 取会偏色；纹理贴图在 iOS 上的色差问题即由此解决。
- 纹理对象创建前应把图片宽高扩展到 2 的整数次幂：`FMTexture::optimize()` 负责这一步。
- `glTexImage2D` 的格式/对齐参数是常见出错点。

### OpenGL 矩阵：列主序

- GL 的矩阵在内存中**按列主序**存放：数组里的数字优先填充矩阵的列向量。
- GLSL 的矩阵与 OpenGL 一致，都是**按列存储**：`m[i]` 取第 i 列，`mat4(...)` 的构造参数按列填充。构造函数写成 16 个分量时，书写顺序容易被误读为「按行填充」，实际是先填满第 0 列再填第 1 列；与按行主序存矩阵的库对接时需要转置。
- `Matrix` 类中的平移矩阵和透视投影矩阵做了转置；默认的正视矩阵和透视矩阵都会创建。
- 视图矩阵可参照 `gluLookAt` 实现；自写矩阵库与 OSG 的实现存在行/列主序差异，替换时容易出错（模型变换矩阵替换曾失败，改用 OSG 矩阵库后成功）。

### 绘制与资源

- 批量绘制（批次合并）与视锥剔除冲突，最终放弃。
- 打开面剔除会让半透明/线框效果失效；绘制 model 时关掉面剔除可实现全透明。
- 参考性能：某 10347_max_level 地图，线框模式约 16.5 fps，非线框约 10.3 fps（线框反而更快，因为省掉了面填充）。首次数据载入约 4.7 s，第一帧约 0.5 s，稳定后单帧约 0.095 s。
- `attribute` 变量的声明顺序交换后会导致绘制出错，原因不明（原稿未定位，待核实）。

### 代码组织

- 头文件里只对用到的类型做前置声明，不包含对应头文件，到 `.cpp` 实现时再按需 include。
- Xcode 工程添加资源时注意是否存在同名文件，否则会被覆盖。
- 主题/资源加载器要区分平台与版本（glTF 1.0 与 2.0 的场景图节点层数、矩阵数量不同，需要版本自适应）。
- 调试 glTF 时可先用 OSG 导出 `.obj + .mtl + .png`，再转成 glTF 做最小用例。

### 常见坑清单

| 现象 | 原因 / 处理 |
|---|---|
| 线与面结合处闪烁 | 打开 Polygon Offset |
| `setfitview` 后 z-fighting | 提升计算精度 / 调整 polygon offset |
| 导航线被模型遮挡 | 绘制顺序 + polygon offset（`_lineLayers` 在 `_layerGroups` 之前绘制，因此更易被模型遮挡（推测）） |
| POI 遮挡与比例尺判断错乱 | 先判等级 `isShow`，再判遮挡 `cover`；Facility 未按 `minLevel` 排序 |
| 多楼层显示后模型整体偏移 | `setMultiDisplay` 楼层数超出实际楼层，补出的空楼层污染包围盒中心 |
| 场景旋转后裁剪/拾取异常 | AABB 失效，改用 OBB |
| 隐藏的模型仍能被选中 | 拾取只按包围盒射线判断，需同时剔除不可见节点 |
| 贴图全黑/失败 | 纹理过滤参数（mipmap 模式）、图片解析、`glTexImage2D` 参数逐项排查 |
| 纹理颜色在 iOS 偏色 | iOS 像素为 BGR 序 |

## 原始笔记索引

- `Log/FengMap_Kernel/FMView.md`
- `Log/FengMap_Kernel/FMLine.md`
- `Log/FengMap_Kernel/FMLayerGroupNode.md`
- `Log/FengMap_Kernel/Label.md`
- `Log/杂记/FengMap_kenel日志.md`
- `Log/杂记/蜂鸟地图显示配置文件格式说明.md`
- `Log/杂记/墨卡托坐标.md`
- `Log/杂记/工作日志1.md`（仅 FengMap 文字/模型/UV 渲染部分）
- `Log/杂记/工作日志2.md`（仅线绘制/深度/剔除/包围盒部分）
- `Log/杂记/dpi.md`