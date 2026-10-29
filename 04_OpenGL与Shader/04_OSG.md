# OSG 与 osgEarth

> 本文由 `OpenGL/OSG/` 下 4 篇笔记重组而成：`osg笔记.md`、`OSG笔记：.md`、`osgEarth的数据加载流程.md`、`osgEarth数据加载流程PNG.md`（后两篇合并）。内容按“OSG 场景图基础 → LOD 与分页调度 → osgEarth 数据加载 → 空间信息与瓦片组织 → 工程经验”组织。
>
> 文中 API 与流程以 osgEarth 2.4 时代的实现为背景；新版 osgEarth（3.x）类名与目录结构有变化，阅读时注意版本差异。

---

## 1. OSG 场景图与对象模型

### 1.1 智能指针：ref_ptr 与 observer_ptr

OSG 的对象几乎都继承自 `osg::Referenced`，用引用计数管理生命周期。

- **`osg::ref_ptr<T>`**：强引用，负责对象的创建与销毁。`ref_ptr` 增加引用计数，离开作用域或重新赋值时减少计数；计数归零对象被删除。用于“拥有”对象。
- **`osg::observer_ptr<T>`**：观察者，只观察一个对象，**不影响**其创建与销毁。被观察对象销毁时，`observer_ptr` 会得到通知并自动置空（`valid()` 返回 false）。用于“不拥有、但需要知道对方是否还活着”的场景，典型如父节点指针、DatabasePager 中记录的 PagedLOD。

```cpp
osg::ref_ptr<osg::Node> node = osgDB::readNodeFile("cow.osg");   // 拥有
osg::observer_ptr<osg::Group> parent = group;                    // 观察，不增计数
if (parent.valid()) { /* 父节点仍存在 */ }
```

选择原则：会成为环、或生命周期由别人管理的对象，用 `observer_ptr`；否则用 `ref_ptr`。DatabasePager 的 `_parentGroup` 就使用 `observer_ptr`，因为加载线程工作期间父节点可能已被摘除。

### 1.2 回调与遍历：NodeCallback

继承 `osg::NodeCallback` 并重写：

```cpp
virtual void operator()(osg::Node *node, osg::NodeVisitor *nv);
```

**必须在函数末尾调用 `traverse(node, nv);`**，否则该节点后续的其它回调、以及子节点的遍历都不会执行。这是 OSG 回调最常见的坑：表现是“某个节点/子树突然不更新或不渲染”。

典型写法：

```cpp
class MyCallback : public osg::NodeCallback
{
public:
    virtual void operator()(osg::Node *node, osg::NodeVisitor *nv)
    {
        // 更新逻辑
        traverse(node, nv);   // 必须保留
    }
};
```

### 1.3 Viewer、Scene 与 DatabasePager

- `osgViewer::Scene` 在 `osgViewer::View` 对象的构造函数里创建，一个 View 对应一个 Scene。
- `Scene` 的构造函数里调用：

```cpp
setDatabasePager(osgDB::DatabasePager::create());
```

因此 View 创建时就已经持有一个可用的分页数据库管理器。

- 基本渲染浏览器是 `osgViewer::Viewer`；osgEarth 在此基础上提供 `osgEarth::Util::EarthManipulator` 等，并组织地形数据，但核心渲染仍由 osgViewer 完成。

### 1.4 OSG 的矩阵与向量约定

OSG 的 `osg::Matrix` 元素在内存中**按行排列**：

```cpp
value_type _mat[4][4];
```

`SET_ROW` 字面上设置一行，但 OSG 采用**行向量**约定，顶点变换写作：

```
v_world * M_model * M_view * M_projection
```

而 GLSL 采用列向量约定，写作：

```glsl
gl_Position = P * V * M * vec4(position, 1.0);
```

两者描述的变换相同，矩阵互为转置、乘法顺序相反。把 OSG 矩阵传给 `glUniformMatrix4fv` 时：

- 直接按原内存上传、`transpose = GL_FALSE` 即可（`osg::Uniform` 的 `FLOAT_MAT4` 分支也是这么做的）：GPU 把第 0 行解释为第 0 列，得到的正是 `M` 的转置，而 `Mᵀ` 恰好就是 GLSL 列向量写法 `P * V * M * v` 所需要的矩阵；
- 若上传时 `transpose = GL_TRUE`，矩阵会再被转置一次，与 shader 的乘法顺序对不上，渲染结果才会完全错误；在 CPU 侧先转置再配 `GL_FALSE` 同样多转了一次，结论相同。

`osg::Matrix::makeTranslate(x, y, z)` 的平移在**最后一行**：

```cpp
void Matrix_implementation::makeTranslate(value_type x, value_type y, value_type z)
{
    SET_ROW(0, 1, 0, 0, 0)
    SET_ROW(1, 0, 1, 0, 0)
    SET_ROW(2, 0, 0, 1, 0)
    SET_ROW(3, x, y, z, 1)
}
```

这正是行向量约定的体现：`[x y z 1] * M` 得到平移后的点。

---

## 2. LOD 与分页调度

### 2.1 osg::LOD（细节层次节点）

`osg::LOD` 根据观察点到节点的距离（或节点在屏幕上占的像素大小）选择显示哪个子节点：近处显示精细模型，远处显示简化模型，从而降低渲染压力、提高帧率。

内部维护一个范围列表：

```cpp
typedef std::pair<float, float> MinMaxPair;
typedef std::vector<MinMaxPair> RangeList;
RangeList _rangeList;
```

`_rangeList` 保存每个子节点的可显示距离范围 `[min, max]`，遍历时与当前距离比较，决定该子节点是否显示。

核心函数：

```cpp
void LOD::traverse(NodeVisitor &nv);
```

添加子节点时自动扩展范围列表：

```cpp
bool LOD::addChild(Node *child)
{
    if (Group::addChild(child)) {
        if (_children.size() > _rangeList.size()) {
            // 取当前最后一个范围的 max 作为新范围的起点，因此 _rangeList 递增
            float maxRange = !_rangeList.empty() ? _rangeList.back().second : 0.0f;
            _rangeList.resize(_children.size(), MinMaxPair(maxRange, maxRange));
        }
        return true;
    }
    return false;
}
```

> `resize` 扩容不会破坏已有元素，只在尾部追加。默认追加的范围是 `[maxRange, maxRange]`，即一个空区间；通常随后会用 `setRange` 明确指定。

#### 2.1.1 距离中心模式

```cpp
enum CenterMode
{
    USE_BOUNDING_SPHERE_CENTER,                    // 用包围球中心
    USER_DEFINED_CENTER,                           // 用户自定义中心
    UNION_OF_BOUNDING_SPHERE_AND_USER_DEFINED      // 两者结合
};

void setCenterMode(CenterMode mode);
```

默认用包围球中心计算到视点的距离；也可 `setCenter` 指定自定义点。

#### 2.1.2 范围模式

```cpp
enum RangeMode
{
    DISTANCE_FROM_EYE_POINT,   // 按到视点的距离
    PIXEL_SIZE_ON_SCREEN       // 按屏幕像素大小
};

void setRangeMode(RangeMode mode);
```

#### 2.1.3 示例

```cpp
osg::ref_ptr<osg::Node> CreateNode()
{
    osg::ref_ptr<osg::Group> root = new osg::Group;

    osg::ref_ptr<osg::Node> cow    = osgDB::readNodeFile("cow.osg");
    osg::ref_ptr<osg::Node> glider = osgDB::readNodeFile("glider.osg");

    // 按距离：0~30 显示牛，30~100 显示滑翔机
    osg::ref_ptr<osg::LOD> lod = new osg::LOD;
    lod->addChild(cow.get(),     0.0f,  30.0f);
    lod->addChild(glider.get(), 30.0f, 100.0f);

    // 按屏幕像素大小：0~1000 显示牛，1000~2000 显示滑翔机
    osg::ref_ptr<osg::LOD> lodByPixel = new osg::LOD;
    lodByPixel->setRangeMode(osg::LOD::PIXEL_SIZE_ON_SCREEN);
    lodByPixel->addChild(cow.get(),     0.0f, 1000.0f);
    lodByPixel->addChild(glider.get(), 1000.0f, 2000.0f);

    osg::ref_ptr<osg::PositionAttitudeTransform> pat =
        new osg::PositionAttitudeTransform;
    pat->setPosition(osg::Vec3(-20.0f, 0.0f, 0.0f));
    pat->addChild(lodByPixel.get());

    root->addChild(lod.get());
    root->addChild(pat.get());
    return root.get();
}
```

**注意**：`osg::LOD` 会一次性把所有子节点的模型载入内存，只是在绘制时按范围选择。它节省的是绘制/三角形处理，不节省内存。

### 2.2 osg::PagedLOD（分页细节层次节点）

`PagedLOD` 继承自 `LOD`，实现动态分页加载：只在实际需要时加载对应层级的模型文件，卸载不再需要的节点。加载由独立线程调度，不会阻塞渲染线程。

```cpp
osg::ref_ptr<osg::Node> CreateNode()
{
    osg::ref_ptr<osg::Group> root = new osg::Group;

    osg::ref_ptr<osg::PagedLOD> page = new osg::PagedLOD;
    page->setCenter(osg::Vec3(0.0f, 0.0f, 0.0f));

    page->setFileName(0, "cow.osg");
    page->setRange(0, 0.0f, 50.0f);

    page->setFileName(1, "glider.osg");
    page->setRange(1, 50.0f, 100.0f);

    page->setFileName(2, "cessna.osg");
    page->setRange(2, 100.0f, 200.0f);

    root->addChild(page.get());
    return root.get();
}
```

要点：

- `setFileName(index, filename)` 与 `setRange(index, min, max)` 的 `index` 必须对应同一层级。
- `PagedLOD` 内部维护 `PerRangeData` 类型的数组，每个元素对应一个层级的文件名、范围等数据。
- 除增加子节点外，每次 `addChild` 还会扩充 `_perRangeDataList` 的容量。
- 节点被删除/移出场景后，DatabasePager 通过过期机制回收；需要正确注册（见 2.3）。

### 2.3 DatabasePager 内部机制

#### 2.3.1 注册 PagedLOD

```cpp
virtual void registerPagedLODs(osg::Node *subgraph, unsigned int frameNumber = 0);
```

注释说明：查找子图中的所有 `PagedLOD` 节点并注册到 DatabasePager，以便跟踪过期节点；**只应在更新线程（update thread）调用**。

所谓“注册”，就是把 `PagedLOD` 节点插入 DatabasePager 维护的 `_activePagedLODList`：

```cpp
osg::ref_ptr<PagedLODList> _activePagedLODList;
```

`_activePagedLODList` 实际是 `SetBasedPagedLODList` 对象，内部维护一个 set：

```cpp
typedef std::set< osg::observer_ptr<osg::PagedLOD> > PagedLODs;
PagedLODs _pagedLODs;
```

使用 `observer_ptr` 的原因：PagedLOD 可能已被场景图释放，set 不能阻止其销毁，但需要知道它是否还存在。

调用链：

```
addLoadedDataToSceneGraph()
  → registerPagedLODs(subgraph)
      → 构造 FindPagedLODsVisitor(_activePagedLODList)
      → 遍历子图，把找到的 PagedLOD 插入 set
```

`registerPagedLODs` 在 `addLoadedDataToSceneGraph` 中被调用；后者在分页线程把已加载数据合并进场景图时执行。

#### 2.3.2 加载请求队列

```cpp
osg::ref_ptr<ReadQueue> _fileRequestQueue;   // 文件加载请求队列

struct OSGDB_EXPORT ReadQueue : public RequestQueue
{
    ReadQueue(DatabasePager *pager, const std::string &name);
    void block()   { _block->block(); }
    void release() { _block->release(); }
    virtual void updateBlock();

    osg::ref_ptr<osg::RefBlock> _block;
    std::string _name;
    OpenThreads::Mutex _childrenToDeleteListMutex;
    ObjectList _childrenToDeleteList;
};
```

`_childrenToDeleteList` 是 `_fileRequestQueue` 的成员，保存待删除的子节点，删除操作在分页线程中批量执行，避免在渲染线程做耗时释放。

> 笔记中“PagedLOD 每帧只发出（如果需要的话）一个瓦片载入请求”带有问号，属于当时的观察/猜测，现代实现会按 `DatabasePager` 的 `_doUpdate` 与最大请求数配置批量发起（待核实具体版本行为）。

#### 2.3.3 文件缓存相关类

```cpp
// 文件列表：内部是 set<string>
class OSGDB_EXPORT FileList : public osg::Object
{
    typedef std::set<std::string> FileNames;
    FileNames _files;
};

// 一次数据库修订：增/删/改文件列表
class OSGDB_EXPORT DatabaseRevision : public osg::Object
{
    std::string _databasePath;
    osg::ref_ptr<FileList> _filesAdded;
    osg::ref_ptr<FileList> _filesRemoved;
    osg::ref_ptr<FileList> _filesModified;
};

// 一个数据库路径下的多次修订
class OSGDB_EXPORT DatabaseRevisions : public osg::Object
{
    std::string _databasePath;
    typedef std::vector< osg::ref_ptr<DatabaseRevision> > DatabaseRevisionList;
    DatabaseRevisionList _revisionList;
};

// 文件缓存
class OSGDB_EXPORT FileCache : public osg::Referenced
{
    std::string _fileCachePath;
    typedef std::list< osg::ref_ptr<DatabaseRevisions> > DatabaseRevisionsList;
    DatabaseRevisionsList _databaseRevisionsList;
};
```

`osgDB::Options` 汇总了读取选项，常用成员：

| 成员 | 作用 |
|---|---|
| `_str` | 选项字符串（如 `"noTexture=1"`） |
| `_databasePaths` | 数据搜索路径列表 |
| `_objectCacheHint` | 缓存哪些对象类型：`CACHE_NONE`、`CACHE_NODES`、`CACHE_IMAGES`、`CACHE_HEIGHTFIELDS`、`CACHE_ARCHIVES`、`CACHE_OBJECTS`、`CACHE_SHADERS`、`CACHE_ALL` |
| `_objectCache` | 对象缓存 |
| `_precisionHint` | 双精度导入选项：`DOUBLE_PRECISION_VERTEX/NORMAL/COLOR/...`，通常只为预处理保留双精度，渲染前转 float |
| `_buildKdTreesHint` | 是否自动构建 kd-tree：`NO_PREFERENCE` / `DO_NOT_BUILD_KDTREES` / `BUILD_KDTREES` |
| `_authenticationMap` | 认证信息 |
| `_pluginData` / `_pluginStringData` | 插件私有数据 |
| `_findFileCallback` / `_readFileCallback` / `_writeFileCallback` | 文件查找/读写回调 |
| `_fileLocationCallback` | 文件位置回调（本地/远程） |
| `_fileCache` | 文件缓存对象 |
| `_terrain` | 地形节点（observer） |
| `_parentGroup` | DatabasePager 设置的“请求文件将插入的节点”，使用 `observer_ptr<osg::Group>`，因为分页线程加载期间父节点可能被摘除 |

---

## 3. osgEarth 数据加载流程

### 3.1 总览

用户只需编写 earth 文件（XML）指定数据源，不必关心底层如何渲染，osgEarth 就能在三维球上显示地形、影像、模型等。整体加载流程分为四步：

1. 读取 earth 文件；
2. 构建 map；
3. 构建 mapNode；
4. 把节点加入场景树。

![osgEarth 数据加载整体流程](../../技术笔记/assets/7D655E4557E643A88FD45705063BFC01.png)

### 3.2 读取 earth 文件

osgEarth 继承了 OSG 的插件机制，提供专门读取 earth 文件的 `osgdb_earth` 插件。查找插件的流程：

1. 根据文件后缀名查找读写插件；
2. 找到后根据文件名构造 `ReadNodeFunctor`；
3. 调用插件的 `doRead()` 完成读取。

读取影像、高程或普通模型都走同一套插件机制，区别只是插件与 `doRead` 的实现。

![查找 earth 插件流程](../../技术笔记/assets/B42BC7C604A74D78818BFEE258BB1799.png)

图中各环节的调用链（`ExampleResources.cpp` → `ReadFile.h/.cpp` → `Registry.cpp`）：

```
MapNodeHelper::load()                                          // ExampleResources.cpp
  → readNodeFile(filename, Registry::instance()->getOptions())  // ReadFile.h
      → readNodeFile()                                          // ReadFile.cpp
          → ReaderWriter::ReadResult readNode()
              → options->getReadFileCallback()                  // 有回调则先走回调
              → _readFileCallback.valid()
              → readNodeImplementation()
                  → readImplementation()
                      → read(readFunctor)                       // 按后缀名查找读写插件
                          → doRead(ReaderWriter& rw)            // 插件内完成真正的读取
```

- `ReadNodeFunctor(filename, options)` 定义在 `Registry.h`；`read()` 根据文件名构造该仿函数，再交给插件的 `doRead()` 执行，具体读取函数由 `ReadNodeFunctor::doRead()` 提供。
- 插件查找策略与 OSG 通用插件机制一致：先按文件后缀名定位读写插件，再调用其 `doRead`。

`osgdb_earth` 的主要工作是把 earth 文件中的 XML 标签转换成后续构建 map 所需的 **conf 对象**：标签的嵌套关系变成父子关系，每个对象携带自己的属性。下面是一个包含 image、elevation、model（feature_geom + ogr）、overlay、options 等标签、比较完整的 earth 文件：

```xml
<map name="Globe" type="geocentric" version = "2">
    <image name="17-0" driver="gdal">
        <url>../data/image/17L-34-35-1085-1095/0.tif</url>
    </image>
    <elevation name="elevation" driver="gdal">
    <model name="buildings" driver="feature_geom">

        <features name="buildings" driver="ogr">
            <url>../data/boston_buildings_utm19.shp</url>
            <build_spatial_index>true</build_spatial_index>
        </features>

        <layout>
            <tile_size_factor>45</tile_size_factor>
            <level name="default" max_range="20000">
                <selector class="buildings"/>
            </level>
        </layout>
    </model>
    <overlay name="china boundaries" driver="feature_geom">
    <options>
</map>
```

几点说明：

- 原截图是编辑器的**折叠大纲视图**：`<elevation>`、`<overlay>` 行首仍是 `+` 号，说明其子标签（如 `<url>`）未展开，结束标签 `</elevation>`、`</overlay>` 也未显示；`<options>` 同样是折叠状态，截图里没有它的 `</options>`。这几个元素在真实文件中都必须闭合，且 `overlay` 通常还需要 `<url>`。
- `version = "2"` 中的空格是原文件写法（XML 允许），属性名为 `version`。
- `max_range`、`tile_size_factor`、`build_spatial_index`、`selector class` 都是 osgEarth 2.x 的 feature_geom/feature 插件属性。

对应的 conf 结构如下：`map` 节点下按标签类型分出 `image`、`elevation`、`model`、`overlay`、`options` 等子节点，`model` 再向下分 `features`、`layout`，每个节点各自带 `name`/`driver`/`url` 等属性。

![earth 文件转换后的 conf 结构](../../技术笔记/assets/AE5BF5E064B84D9B8C0B2C9C425AB094.png)

### 3.3 构建 map

根据 conf 对象构造一个逻辑上的 map。**这一步并不读取实际数据**，只是按属性分类，确定：

- map 包含哪些图层；
- 每个图层的名字、数据源（TileSource）；
- 每个图层所需的 driver 插件；
- 空间参考系 profile 等。

![通过属性构造 map](../../技术笔记/assets/65AE119C0E9E442198EF7E1F5F75764D.png)

图中各环节的关键点（`EarthFileSerializer2.cpp` → `map.cpp`）：

```
ConfigSet images       = conf.children("image");       // XML 中所有 image 元素
ConfigSet heightfields = conf.children("elevation");
ConfigSet models       = conf.children("model");
ConfigSet overlays     = conf.children("overlay");
ConfigSet masks        = conf.children("mask");
```

- 每个 `ConfigSet` 遍历后得到一个 `layerDriverConf`，即 XML 里一条元素上的**全部属性**（name、driver、url 等）。
- 由 `layerDriverConf` 分别产出：`ImageLayer` / `ImageLayerOptions`（`Default_tile_size` 为 256）、`ElevationLayer` / `ElevationLayerOptions`（`Default_tile_size` 为 16）、`ModelSourceLayer` / `ModelSourceOptions`、`MaskLayer` / `MaskLayerOptions`。
- `map.cpp` 侧由 `Map::addImageLayer()`、`Map::addElevationLayer()`、`Map::addModelLayer()`、`Map::addTerrainMaskLayer()` 把图层挂到 map 上，入口是 `deserialize()`。

> 原图把 XML 写成 “XLM”，此处按正确拼写 **XML** 记录（原图有误，已更正：`XLM` → `XML`）。

### 3.4 构建 mapNode（核心）

根据 map 与 options，调用具体 driver 插件，构建真正的地形节点。分为两步。

#### 3.4.1 第一步：渲染前预处理

在 osgViewer 开始渲染前，由地形引擎的 `preInitialize()` 完成。osgEarth 2.4 的默认地形引擎是 `MpTerrainEngineNode`，因此通常在 `MpTerrainEngineNode::preInitialize()` 中：

- 搭建地形节点的初步框架；
- 加载底图影像（base layer）；
- 生成包含地形节点、模型节点、overlay 模型节点的子树。

在根据 map 的空间参考系设置 TerrainEngineNode 的坐标系和椭球模型时，会读取 map 的 `profile`。`Profile` 是确定数据空间信息的关键属性；若 options 中没有设置 profile，默认 SRS 为 **WGS84**。

![构建 mapNode 第一步逻辑](../../技术笔记/assets/0A9DC2976D514497821A97D342EF8A0C.png)

![第一步详细流程](../../技术笔记/assets/B3C1C7A12B914DD6BB0AD4EDEC701C8A.png)

图中展开的代码要点（`MapNode.cpp` → `TerrainEngineNode.cpp`）：

```cpp
// MapNode.cpp：地形引擎需要一个 Container 来设置光照状态，不能直接在引擎上设置
_terrainEngineContainer = new osg::Group();
_terrainEngineContainer->setDataVariance(osg::Object::DYNAMIC);
this->addChild(_terrainEngineContainer.get());

// 开启一个地形功能界面
_terrainInterface = new Terrain(this, map->getProfile(), map->isGeocentric());
_map->getProfile()->getSRS()->populateCoordinateSystemNode(this);   // 按 _map 的 SRS 设置坐标系与椭球模型，WKT/PROJ4 均有默认值

// 开始只是球体与底图显示，所以此时只添加底图
MapFrame mapf(map, Map::IMAGE_LAYERS);
_texCompositor->applyMapModelChange(
    MapModelChange(
        MapModelChange::ADD_IMAGE_LAYER,
        mapf.getRevision(),
        mapf.getImageLayerAt(i),
        i));

// 创建模型节点
_models = new osg::Group();
_models->setName("osgEarth::MapNode.modelsGroup");
addChild(_models.get());

// 创建 overlay 模型节点
_overlayModels = new osg::Group();
_overlayModels->setName("osgEarth::MapNode.overlayModelsGroup");
_overlayDecorator = new OverlayDecorator();
addTerrainDecorator(_overlayDecorator.get());

// 加载预先存在的模型图层
ModelLayerVector modelLayers;
_map->getModelLayers(modelLayers);
for (unsigned i = 0; i < modelLayers.size(); ++i)
    onModelLayerAdded(modelLayers[i].get(), i);   // 原图写作 onModelLayerAdded( modelLayers[k].get(), modelLayerIndex )

_mapCallback = new MapNodeMapCallbackProxy(this);   // 注册 MapNodeMapCallback 回调函数
_map->addMapCallback(_mapCallback.get());

// TerrainEngineNode.cpp 末尾
_map->addMapCallback(new TerrainEngineNodeCallbackProxy(this));   // 处理 map 模型改变的事件
// 开启背面剔除、获取状态、设置属性和模式、
// 增加 Uniform cameraElevationUniform、获取 OsgEarth::ImageLayerAttenuation 并设置
```

其中 `applyMapModelChange` 里首先 `getProfile`：因为影像尚未初始化，故需调用相应 driver 去读取数据信息、取得其 profile。此时生成的 `MapNode` 子树为：`osgEarth::MapNode : public osg::Group` → `osg::Group()`（地形节点 → `osgEarth::MpTerrainEngineNode`）＋ `osg::Group()`（模型节点）＋ `osg::Group()`（Overlay 模型节点）。

#### 3.4.2 第二步：渲染开始后的动态构建

osgViewer 开始渲染、创建漫游、相机添加场景后，继续构建后续节点，并用 `TileKey` 管理四叉树。此时根据视点范围与距离，通过 OSG 的 `PagedLOD` 分页机制动态调度、选择加载区域瓦片。

![构建 mapNode 第二步逻辑](../../技术笔记/assets/101531B007AE44C9A18669E15CF04315.png)

![第二步详细流程](../../技术笔记/assets/2BE018BB9C6F43B6BCA14857B5E0EB65.png)

图中 `MpTerrainEngineNode.cpp` 的关键代码：

```cpp
// 实质调用 createTerrain()
if (_tileModelFactory)
    _tileModelFactory->getHeightFieldCache()->clear();
_terrain = new TerrainNode(_deadTiles.get());
this->addChild(_terrain);

// 创建地形的第一层（LOD 为 0），收集包括地形根瓦片的 tile keys
std::vector<TileKey> keys;
_update_map->getProfile()->getAllKeysAtLOD(*_terrainOptions.firstLOD(), keys);

// 为每个 root tile key 创建一个 root 节点，在其上读取具体影像与高程并放到正确位置
for (unsigned i = 0; i < keys.size(); ++i)
{
    osg::Node* node = factory->createRootNode(keys[i]);
    if (node)
        _terrain->addChild(node);
}
```

此时的节点结构变为：`osgEarth::MapNode : public osg::Group` → `osg::Group()`（地形节点）→ `osgEarth::MpTerrainEngineNode` → `TerrainNode` → `RootNode`（`TileNodeGroup`）……，另有 `osg::Group()`（模型节点）与 `osg::Group()`（Overlay 模型节点）。

第二步的重点是创建第一层 `TileKey` 与根节点。每个 rootNode 包含：

- 用四叉树索引组织的 key（含范围信息）；
- 该范围内的影像、高程等数据。

![创建根节点逻辑](../../技术笔记/assets/70A54FD9581A46098B93477BDB98F047.png)

`createRootNode` 归纳为两件事：

1. 创建包含影像图层、高程图层与 `TileKey` 的瓦片模型；
2. 以 `PagedLOD` 形式把该模型加入根节点。

其中 `TileKey` 包含模型的空间信息，能决定数据在三维球上的正确位置；`PagedLOD` 根据视点范围决定加载哪些节点。

![createRootNode 流程](../../技术笔记/assets/D7AEBE37E75E42EBBDD0F5432AD51199.png)

图中 `SerialKeyNodeFactory.cpp` → `NodeFactory.cpp` → `TileModelFactory.cpp` 的代码要点：

```cpp
// SerialKeyNodeFactory.cpp
createRootNode(const TileKey& key)
    → _modelFactory->createTileModel(key, model, real, lodBlending);

// NodeFactory.cpp：一个瓦片节点 group 只能放一个单独的瓦片
osg::Group* root = new TileNodeGroup();
addTile(model.get(), real, lodBlending, root);
return root;

// TileModelFactory.cpp
MapFrame map(_map, Map::MASKED_TERRAIN_LAYERS);
const MapInfo& mapInfo = map.getMapInfo();
osg::ref_ptr<TileModel> model = new TileModel();
model->_map         = _map;
model->_tileKey     = key;
model->_tileLocator = GeoLocator::createForKey(key, mapInfo);
```

影像部分（`BuildColorData`）：

```cpp
BuildColorData build;
build.init(key, layer, order, mapInfo, _terrainOptions, model.get());
bool addedToModel = build.execute();          // 返回 bool，实际创建图片并加载到图层
```

高程部分（`BuildElevationData`，位于另一个函数中）：

```cpp
BuildElevationData build;
build.init(key, map, _terrainOptions, model.get(), _hfCache);
build.execute();                              // 读取高程，构建四叉树的 key
```

两个布尔参数的含义：

- `real`：是否含有 real data，即直接与当前 `key` 关联的数据；否则为 fallback data（由低一级 LOD key 中衍生出来的数据）。
- `lodBlending`：是否使用 lod blending。

`BuildElevationData` 的核心函数与 `BuildColorData`（image）相同，可对照分析 key 如何分层；区别是 `execute()` 无返回值。

在创建影像图层与高程图层时，相应 `execute` 函数分别调用 `createImage()` 与 `createHeightfields()`；这两个函数根据数据 option 调用对应 driver 插件，完成真正的数据读取。**若要编写自定义影像/高程插件，必须重载这两个函数。**

![GeoLocator 的空间信息处理](../../技术笔记/assets/B8610883336D4F28A5697BFA68F43833.png)

图中 `Locators.cpp` 的流程与代码：

```cpp
// GeoLocator::createForKey 创建地形块的 locator
// 1) 先根据 key 取出该地形块的范围
double xmin, ymin, xmax, ymax;
// 2) 由范围创建 Locator
SpatialReference::createLocator(double xmin, double ymin, double xmax, double ymax, bool plate_carre);
// 3) 空间参考系是地理坐标系（非投影），或 map 为平面时：
//    先把 xmin/ymin/xmax/ymax 由度转成弧度，再调用下面这个函数；否则直接用度调用
static osg::Matrixd
getTransformFromExtents(double minX, double minY, double maxX, double maxY)
{
    osg::Matrixd transform;
    transform.set(
        maxX-minX, 0.0,      0.0, 0.0,
        0.0,       maxY-minY, 0.0, 0.0,
        0.0,       0.0,      1.0, 0.0,
        minX,      minY,     0.0, 1.0);
    return transform;
}
```

矩阵的含义：从原点向左下角 `(minX, minY)` 移动，`x` 轴放大 `(maxX-minX)` 倍、`y` 轴放大 `(maxY-minY)` 倍。平移写在第 4 行，正是 1.4 节所述 OSG 行向量约定的体现。`Key` 中的 `extent` 存着相应 info，由 `createForExtent` 具体处理。

`GeoLocator` 负责确定瓦片模型的位置：先通过 `key` 取得地形块的范围，再把它移动到相应位置。

最终得到一个包含地形节点、模型节点与 overlay 模型节点的 `MapNode`。其中地形节点是主体，其影像图层、高程图层按四叉树组织，由分页 LOD 动态调度渲染。

> 更细的流程图见原笔记附件 `osgEarth数据加载流程PNG.md` 中的大图（即下图，原为 CSDN 文章《osgEarth数据加载及组织解析》的整页长截图）。

![osgEarth 数据加载流程总图](../../技术笔记/assets/902D69B583C74B228DBDAEB819667095.png)

> 上图是原文网页的整页长截图（含正文 1.1–1.4、2.1–2.3 及图 3.1–图 3.16、表 3.1），属于“图文混排长图”，故保留原图；其中的文字内容已完整体现在本节第 3、4 章，不再逐段重复转写。阅读时注意：截图中的“图 3.x”编号是本节的编号来源，图 3.13–3.16 分别对应 ECEF 坐标系、瓦片分层 LOD、缓存文件夹、cacheId 计算流程。

### 3.5 加入场景树

最后创建一个 Group，把 MapNode 加进去，再交给 osgViewer：

```cpp
osg::Group *root = new osg::Group();
root->addChild(mapNode.get());
viewer.setSceneData(root);
```

运行过程中，视野变化会触发 osgEarth 通过 `PagedLOD` 动态加载/卸载瓦片，重复上述 mapNode 构建步骤来生成新节点。

---

## 4. 空间信息、瓦片组织与缓存

### 4.1 空间参考系（SRS）

osgEarth 中每个 map、layer、TileSource 都有 `profile` 属性，`profile` 中的 `SRS` 记录空间参考系。相同经纬度在不同 SRS 下可能表示不同地点，因此每个数据对象都必须带 SRS。

一个空间参考系包含四部分：

1. 坐标系类型；
2. 水平基准（horizontal datum）；
3. 高程基准（vertical datum）；
4. 投影（projection）。

#### 4.1.1 坐标系类型

osgEarth 支持三类：

1. **地理坐标系（Geographic）**：用角度制经纬度表示，如 WGS84、NAD83。
2. **投影坐标系（Projected）**：把三维区域投影到二维 `(X, Y)` 平面，如 UTM、墨卡托。
3. **ECEF（Earth Centered Earth Fixed）**：标准的地心地固三维笛卡尔坐标系（osgEarth 内置实现），原点在球心，`X` 轴指向经纬度 `(0,0)`，`Y` 轴指向**东经 90°**（`(90,0)`，右手系下即 `X` 轴绕 `Z` 轴逆时针转 90°），`Z` 轴指向北极。其轴向与 OSG 世界坐标系一致。

![ECEF 坐标系](../../技术笔记/assets/0D4A5939AD4C42B0B810BFE9C9F55985.png)

#### 4.1.2 水平基准面

不同测量方法对应不同基准面；同一地点也可以有不同的基准。地球不是完美球体、甚至不是完美椭球，因此不同地区会用不同椭球拟合。北美常用 WGS84/NAD83，欧洲常用 ETRS89。

osgEarth 默认水平基准为 **WGS84**；数据未指定时按 WGS84 参数处理。

#### 4.1.3 高程基准面

osgEarth 支持两类：

- **大地基准面（geodetic）**：基于椭球，默认；
- **大地水准面（geoid）**：基于地球上的高程点。

内置四种：

1. `Geodetic`（默认，使用水平基准椭球）；
2. `EGM84 geoid`；
3. `EGM96 geoid`（常称 MSL，DTED/KML 常用）；
4. `EGM2008 geoid`。

默认高程基准是 geodetic，海拔按 **HAE（height above ellipsoid）** 测量。

#### 4.1.4 投影与表达

SRS 同时提供投影信息（三维点投到二维平面的方法）。osgEarth 依赖 GDAL 与 OGR，因此支持上千种投影（UTM、兰伯特等）。

SRS 可用三种通用方式表达：

- **WKT（Well Known Text）**
- **PROJ4**
- **EPSG 代码**

### 4.2 瓦片金字塔与 TileKey

osgEarth 实时建立分层瓦片集进行渲染。它通过记录 SRS 与范围信息的 `profile` 创建瓦片数据 `TileSource`，并用 `TileKey` 以四叉树管理。

世界范围以经纬度表示，把球面剖开铺平后：

- 最左：西经 180°；
- 最右：东经 180°；
- 最上：北纬 90°；
- 最下：南纬 90°。

`profile` 中与分层相关的属性（以一个全球影像数据为例）：

| 属性 | 值 |
|---|---|
| `extent.west` | -180 |
| `extent.east` | 180 |
| `extent.south` | -90 |
| `extent.north` | 90 |
| `numTilesWideAtLod0` | 2 |
| `numTilesHighAtLod0` | 1 |
| `fullSignature` | 3be5346 |
| `horizSignature` | 3be5346 |

- `numTilesWideAtLod0` / `numTilesHighAtLod0` 决定第 0 层的列数与行数，默认 2×1，正好把全球分成东西半球。
- 每向下深入一层，每个瓦片等分成四块，形成四叉树。
- 每个瓦片由 `TileKey` 管理：上一层 key 有 4 个子 key。
- **瓦片以左上角为原点**编码，`TileKey(lod, x, y)`，其中 `x` 向右、`y` 向下。
- 例：第 3 层阴影瓦片的 key 为 `TileKey(2, 5, 1)`，其范围为东经 45°、东经 90°、南纬 0°、北纬 45°（`x=5` 落在东半球第 6 列，跨度为 45°，不是西经 45°）。
- 当瓦片大小小于“视点到瓦片的距离”对应的阈值时，不再继续细分。
- 数据源在该范围内有数据才加载该瓦片，因此总加载块数受数据覆盖范围限制。

![瓦片分层 LOD 与 TileKey](../../技术笔记/assets/DDC5AC29E4CD4D7DA5BFB7AA2F18489A.png)

### 4.3 缓存编码

osgEarth 支持把网络数据提前缓存到本地，加速后续访问。缓存目录结构：

- 第一层：缓存名称（通常一个 cache 对应一个数据源；图中示例包含三个数据源）；
- 第二、三、四层：按 `TileKey` 的 `(lod, x, y)` 编码；
- 最里层：`.osgb` 文件，是 OSG 的二进制格式，存储该瓦片 TileSource 的数据；
- 每个数据源缓存目录下还有一个记录数据源属性的 JSON 文件。

第一层目录名看似乱码，实际由两段组成：

```
617891fe_3be5346
   ↑         ↑
 cacheId   horizSignature
```

- **cacheId**：在解析 conf、构建 map 时计算。只取 map 的 layer 中 TileSource 的 conf 属性，写成 JSON，再用 **MurmurHash** 压缩成字符串。
- **horizSignature**：在构建 mapNode、获取 map 的 profile 时计算。取能确定空间信息的水平基准、垂直基准、地图范围、`numTilesWideAtLod0`、`numTilesHighAtLod0` 等属性，同样写成 JSON 后用 MurmurHash 压缩。

![缓存文件夹结构](../../技术笔记/assets/1DAC5250A3374A39ACD335C381A92D65.png)

![cacheId 计算流程](../../技术笔记/assets/94676F83F1324F429FA032B34BCC6971.png)

图中 cacheId 的计算代码：

```cpp
Layer->setCache(this->getCache());

Config layerConf  = _runtimeOptions->getConfig(true);
Config driverConf = _runtimeOptions->driver()->getConfig();
Config hashConf   = driverConf - layerConf;      // 差集：扣除 layer 中已含的键

hashConf.remove("cache_only");
hashConf.remove("cache_enabled");
hashConf.remove("cache_policy");
hashConf.remove("cacheid");                      // 缓存自身相关属性不参与签名

cacheId = Stringify() << std::hex << osgEarth::hashString(hashConf.toJSON());
```

即：只取 tilesource 的 config，去掉与缓存开关相关的字段后转成 JSON，再用 MurmurHash 对其哈希，最后按十六进制转成 cacheId 字符串。

因此目录名并非乱码，而是同时编码了“数据源配置”与“地图空间信息”。改动数据源或空间参考系后签名会变化，从而自动生成新的缓存目录，避免读到旧缓存。

> 原笔记写作 “MumurHash”，正确拼写为 **MurmurHash**。

---

## 5. 工程经验与坑点

1. **回调忘记 `traverse`**：`NodeCallback::operator()` 末尾必须调用 `traverse(node, nv)`，否则后续回调与子树遍历中断。表现为节点不更新、子节点不渲染。
2. **`osg::LOD` 不等于省内存**：它一次性加载所有层级，只做选择性绘制；需要按需加载时用 `PagedLOD`。
3. **`PagedLOD` 的 index 必须成对**：`setFileName(i, ...)` 与 `setRange(i, ...)` 使用同一个 `i`，否则会出现“范围到了但没文件”或“文件加载了但从不显示”。
4. **PagedLOD 的注册只应在更新线程**：`registerPagedLODs` 的注释明确要求从 update thread 调用；在其它线程调用会破坏 `_activePagedLODList` 的线程安全。
5. **父节点用 `observer_ptr`**：DatabasePager 的 `_parentGroup` 是 `observer_ptr<osg::Group>`，因为分页线程加载期间父节点可能已从场景图摘除；用 `ref_ptr` 会阻止其释放，造成泄漏或悬挂引用。
6. **矩阵布局**：OSG 行主序 + 行向量约定，与 GLSL 列向量约定相反；按原内存上传（`transpose = GL_FALSE`）得到的转置矩阵正是 GLSL 所需，传 `GL_TRUE` 或多做一次 CPU 转置都会导致整个场景的变换出错。
7. **`PagedLOD` 过期回收**：节点离开场景后不会立刻释放，由 DatabasePager 按帧/策略回收；不要手动删除仍在加载队列中的节点。
8. **osgEarth 缓存签名**：修改数据源配置、SRS、LOD0 瓦片数后，`cacheId`/`horizSignature` 会变化并生成新目录；排查“改了配置但画面没变”时先检查缓存目录。
9. **earth 文件本质是配置**：它只描述数据源、图层、渲染与空间参考，不包含实际数据；真正的读取发生在 `preInitialize` 与后续 `PagedLOD` 调度中。
10. **自定义 driver**：影像插件需实现/重载 `createImage()`，高程插件需实现/重载 `createHeightfields()`，否则图层不会产生实际数据。

---

## 附录：工作日志中的补充记录

> 本节补录 `Log/` 中的零散技术片段：`opengl总结.md` 的 OSG 部分（三棵树、一帧三次遍历、DatabasePager 四列表），以及易航点云项目的类图索引与渲染侧结论。
> 原日志为「日期 + 计划/待办 + 勾选」的流水形式，这里只保留技术结论、代码与参数，删除日期、待办勾选、个人事务与本机路径。与正文重复处只给交叉引用。

### B.1 三棵树：场景树、状态树、渲染树

OSG 把 OpenGL 封装在场景图之后，后台同时维护三棵树，各自职责不同：

| 树 | 来源 | 职责 |
|---|---|---|
| **场景树**（scene graph） | 用户显式构建 | 组织空间与结构：变换节点、开关节点（`Switch`）、LOD 节点、叶节点等；叶节点持有可绘制对象（`Drawable`） |
| **状态树**（state graph） | 裁剪遍历时**自动生成** | 把各节点上设置的渲染状态集合成树，并对冗余指定做合并、对状态做排序，减少状态切换 |
| **渲染树**（render graph） | 裁剪遍历末尾生成 | 按是否透明对物体分类，并对不透明部分按深度排序，据此决定绘制顺序 |

几点结论：

1. **场景树是唯一的“输入”**，状态树与渲染树都是遍历的产物，随视点与场景变化每帧重建，不能当作持久结构去修改。
2. 场景遍历采用**访问者（Visitor）模式**，`osg::NodeVisitor` 上有两个**互相独立**的维度，不能混写：
   - **VisitorType**（`NODE_VISITOR` / `UPDATE_VISITOR` / `EVENT_VISITOR` / `CULL_VISITOR`）决定节点 `accept()` 的行为，即走哪条访问分支；
   - **TraversalMode**（`TRAVERSE_NONE` / `TRAVERSE_ALL_CHILDREN` / `TRAVERSE_ACTIVE_CHILDREN`）决定 `traverse()` 是否继续下探子节点，与遍历类型无关——任何 VisitorType 都可以配 `TRAVERSE_NONE` 只处理当前节点。

   状态树正是在 **cull 遍历**（`CULL_VISITOR`）中、按各节点设置的 `StateSet` 累积生成的。
3. **状态树的三项优化**：合并重复的状态指定（父子节点状态相同的部分只保留一份）、按 `StateSet` 的内容排序，从而为渲染阶段按状态连续绘制创造条件——最终目的都是减少 `gl*` 状态切换，与 `01_OpenGL基础.md` §1.3 所述“按状态排序绘制”的原则一致。
4. **渲染树按透明与否分类**：不透明物体先画且可任意顺序（深度测试兜底），透明物体后画并需按视点深度从远到近排序；渲染树就是承载这一排序结果的结构。

### B.2 一帧的三次遍历

一帧的绘制按三次遍历推进，顺序固定：

1. **事件遍历（event traversal）**：处理用户输入与系统事件，派发到事件回调（`osgGA::GUIEventHandler`）。
2. **更新遍历（update traversal）**：执行所有更新回调（`NodeCallback`、`Drawable::UpdateCallback`），并驱动**分页数据库**的更新（提交/合并已加载数据）。
3. **渲染遍历（rendering traversal）**：完成绘制。内部再分两遍——先 **cull（裁剪）**：视锥剔除、计算 LOD、生成状态树与渲染树；再 **draw（绘制）**：按渲染树顺序提交绘制命令。

> 原稿按“事件 / 更新 / 渲染”记作三次遍历；严格说 render 阶段内部还含 cull 与 draw 两遍，因此也有资料称四次遍历（原稿计数方式已澄清，`osgViewer::Viewer` 中对应 `eventTraversal()` / `updateTraversal()` / `renderingTraversal()`）。

对应的实际收益：**能把工作挪到正确遍历里的，就不要放在 draw 阶段做**。例如相机相关计算放 `updateTraversal` 的回调里，避免在 cull 之后才修改场景导致这帧白算。

### B.3 DatabasePager 的四个列表

分页数据库（`osgDB::DatabasePager`）是 OSG 最重要的特性之一：它实现场景数据的动态加载与卸载，从而支撑 osgEarth 这类大数据量应用。它维护四个列表，数据在四者之间流转：

| 列表 | 含义 | 成员名（依据 OSG DatabasePager 头文件，版本待核实） |
|---|---|---|
| **请求列表** | 新的对象、或进入视口/范围内需要加载的瓦片，等待加载线程读取 | `_fileRequestQueue` |
| **待编译列表** | 已读入内存、需要**提前编译**的对象（构造显示列表、上传纹理/VBO 等），以免首次绘制时卡顿 | `_dataToCompileList`（待核实具体成员名） |
| **待合并列表** | 已编译完成、即将被**附加到场景图**的对象，由更新遍历合并 | `_dataToMergeList`（待核实具体成员名） |
| **待删除列表** | **失效**的瓦片（移出范围、被更高层级替换），等待后台线程批量释放 | `_childrenToDeleteList` |

> 上表成员名取自 OSG `DatabasePager` 头文件的通行版本，具体名称与数量随 OSG 版本变化（待核实）。本文 §2.3.2 只展开了 `_fileRequestQueue`（及 `ReadQueue::_childrenToDeleteList`）的源码，**待编译列表与待合并列表在 §2.3.2 中没有对应源码**，此处不做源码级断言。

几个要点：

1. 四个列表的划分本质是**把耗时分摊到不同线程与不同帧**：读取在分页线程，编译可在分页线程或预编译线程，合并必须在更新遍历（主线程）完成，删除也在后台批量做，避免在渲染线程里释放大量对象造成掉帧。
2. 请求列表的入队来源就是 `PagedLOD` 在 cull 遍历中的请求；出队加载完的对象进入待编译/待合并，最终由 `addLoadedDataToSceneGraph()` 挂到场景图（见 §2.3.1）。
3. 待删除列表挂在 `_fileRequestQueue` 上（`ReadQueue::_childrenToDeleteList`），删除时需加锁（`_childrenToDeleteListMutex`），因为入队与清理发生在不同线程——这也是 §2.3.2 中该锁存在的原因。
4. 由此可得一条排查经验：**“瓦片迟迟不出现”先看请求/合并队列是否积压，“内存不降”先看待删除列表是否在为加载让路而延后释放**（待核实具体版本的回收策略）。

### B.4 易航点云项目：类图索引与渲染侧结论

易航是基于 OSG 的激光点云（LAS）可视化项目。原始笔记只留下一行**类名清单**，没有类间关系，因此下面只做“名字 → 职责”的索引，关系与细节标注「（待核实）」。

#### B.4.1 类图索引

| 类 / 组件 | 职责（依类名与开发日志推断，待核实） |
|---|---|
| `LasMap` | 地图/场景根对象，组织点云节点与其它图层 |
| `CameraFrustumEventHandler` | 相机视锥事件处理器：相机参数或窗口变化时触发视锥更新 |
| `Frustum cull` | 视锥剔除流程；清单中作为独立条目出现，可能是模块名而非类名 |
| `LasDataOctree` | 点云八叉树：负责空间索引与节点划分，是视锥剔除与缓存调度的基础（划分阈值属算法细节，此处不展开） |
| `LasDataGeometry` | 点云的可绘制几何体，应为 `osg::Geometry`（`Drawable` 的子类）派生，承载顶点数组与绘制调用 |
| `LasIndexNode` | 按索引网格号组织的索引节点，是缓存加载/卸载的调度单位 |
| `LasSection` | 点云分段/切片（用途待核实） |
| `PointsReader` | 点云文件读取器，负责 LAS 数据解码与属性解析 |
| `MazeManipulator` | 漫游/相机操作器，应为 `osgGA::CameraManipulator` 派生 |
| `GraphicsWindowQt` | 把 OSG 渲染窗口嵌入 Qt 控件（osgQt 的 `GraphicsWindowQt`），与项目后期的 Qt/HMI 化方向一致 |

工程侧的相关记录：视锥剔除曾改为**独立线程**执行，以避免占用渲染线程；每个节点上**附加了包围盒**，供剔除与求交使用（用途待核实）。

#### B.4.2 `CalcArea::rasterize` 的坐标约定

```cpp
void CalcArea::rasterize(std::vector<osg::Vec3d> &coords);
```

该函数把一串世界坐标点光栅化成计算区域。要点：

- 输入使用**该项目采用的场景坐标**（y 轴向上），地平面是 **xz 平面**，因此计算区域的**宽高取 x 与 z 的跨度**，而不是 x 与 y：
  ```
  width  = xmax - xmin
  height = zmax - zmin      // 不是 y 方向
  ```
- 原日志写作“也考虑到了 y 和 z 的问题，目前采用的是 OSG 的坐标，所以 xz 为宽高”；据此推测早期可能按 xy 计算，后改为 xz（待核实）。
- 这与 `01_OpenGL基础.md` §2.6（左手系/右手系）呼应：轴向上的约定必须在整条链路上统一，否则“宽高”这类派生量会静默出错。

#### B.4.3 拼接射线解决“超出点云范围”的拾取失败

单条射线做地形/点云求交时，若射线落点**超出点云覆盖范围**，求交直接失败（没有几何可命中），交互上表现为点击无响应。解决方式：

- **拼接（扩展）射线检测面，使检测范围覆盖点云之外**，从而在点云边界外也能得到有效结果（具体拼接方式与判定逻辑待核实）。
- 配套做法：**加大视口范围**以获得更大的“有效区”，用这个有效区去驱动缓存更新（待核实）。
  > 后续评估认为“缓存区域的加载会影响场景载入”，倾向**去掉缓存区域**、改为视口预加载（待核实最终是否落地）。

#### B.4.4 Linux + GLES/EGL 构建

- 目标是**把 OpenGL ES 绘制部分封装成库**，实现渲染层与 OS 解耦；气泡类 UI 组件（具体形态待核实）用 C++ 实现但对外提供 **C 接口**，同样是为跨平台/跨语言解耦。
- Linux 下的 GLES 环境走 **EGL + OpenGL ES**（配 GLFW 创建窗口与输入），相关参考资料为 Ubuntu 上获取 EGL/GLES 库的问答与 NVIDIA DRIVE 的 GLES 文档。
  > 原日志中“安装 linux 下 ogles 环境”一条标注为**失败**，最终可用方案未在原笔记中确认（待核实）。
- 构建组织：ARM 平台用 makefile、Ubuntu 用 CMakeLists，后续 makefile 编译改为 CMakeLists；另有把 `test_pbr` 工程适配到 Linux 的经验。
- 这条路线与 OSG 的关系：OSG 的 `GraphicsWindow` 抽象正是“渲染窗口与 OS 解耦”的成熟做法，`GraphicsWindowQt`（见 B.4.1）即其一个后端实现——自研 GLES 库等于在 OSG 之外重做同一层抽象。

---

## 原始笔记索引

- `OpenGL/OSG/osg笔记.md`
- `OpenGL/OSG/OSG笔记：.md`
- `OpenGL/OSG/osgEarth的数据加载流程.md`
- `OpenGL/OSG/osgEarth数据加载流程PNG.md`
- `Log/opengl总结.md`
- `Log/易航/LasMap.mindmap.md`
- `Log/易航/易航开发日志.md`
