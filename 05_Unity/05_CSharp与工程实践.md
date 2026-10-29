# C# 与 Unity 工程实践

> 本文整合 C# 语言基础（值/引用类型、参数修饰符、只读、泛型约束、委托与事件、可空类型）、Unity 常用 API 与生命周期，以及编辑器/工程侧的实用技巧。

## 1. C# 基础

### 1.1 值类型与引用类型

| | 值类型 | 引用类型 |
|---|---|---|
| 基类 | `System.ValueType` | `System.Object` |
| 分配位置 | 栈（作为字段时随宿主） | 托管堆 |
| 赋值 | 复制值 | 复制引用地址 |
| 参数传递 | 默认按值 | 默认传递引用地址 |
| 默认构造函数 | 总有 | 未声明任何构造函数时由编译器生成 public 无参构造 |
| 继承 | 不支持 | 支持 |
| 典型用途 | 存储数据 | 面向对象建模 |

值类型包括：`byte`、`short`、`int`、`long`、`float`、`double`、`decimal`、`char`、`bool`、`struct`、`enum`。
引用类型包括：`string`、`class`、`interface`、`delegate`、数组。

### 1.2 struct 与 class 的差异

| | class | struct |
|---|---|---|
| 类型 | 引用类型 | 值类型 |
| 创建 | 托管堆上，只能 `new SomeClass()` | 栈上，可 `new SomeStruct()`，也可 `SomeStruct s;` |
| 赋值 | 赋引用地址 | 赋值 |
| 参数传递 | 传引用地址 | 传值 |
| 默认构造函数 | 未声明任何构造函数时由编译器生成 | 总有（隐式无参构造） |
| 继承 | 支持 | 不支持 |
| 常用场景 | 行为/对象 | 纯数据 |

工程建议：struct 应保持小体积（经验值 ≤ 16 字节）且不可变，否则按值传递的拷贝成本会超过收益；需要频繁装箱（转 `object`/接口）时不要用 struct。

### 1.3 ref / out / in

参数用 `ref`、`out`、`in` 修饰后按**引用**传递，而非按值。

共同点：

1. 方法定义与方法调用处都必须显式写出关键字；
2. 参数按引用传递。

不同点：

| 修饰符 | 调用前是否必须初始化 | 方法内是否必须赋值 | 方法内是否可修改 |
|---|---|---|---|
| `ref` | 是 | 否 | 可以修改，也可以不修改 |
| `out` | 否 | **必须**赋值 | 可以修改 |
| `in` | 是 | 否 | **不能**修改（只读引用） |

```csharp
void Foo(ref int a) { a++; }
void Bar(out int b) { b = 1; }        // 必须赋值
void Baz(in Vector3 v) { /* 只读 */ }
```

`in` 的主要价值是让大 struct 避免拷贝，同时向调用方承诺不被修改。对引用类型使用 `in` 只是防止重新赋值引用，没有性能收益。

### 1.4 readonly 修饰数组与列表

```csharp
public readonly List<int> tt = new List<int>();
public readonly int[] aa = new int[] { 1, 2, 3 };
```

`readonly` 限制的是**字段本身的引用**：初始化后（声明处或构造函数中）不能再把该字段指向别的数组/列表。但**元素仍然可以增删改**。

```csharp
aa[0] = 99;      // 合法
tt.Add(1);       // 合法
aa = new int[5]; // 编译错误
```

需要元素级只读时，使用 `IReadOnlyList<T>`、`ReadOnlyCollection<T>`，或返回副本。

### 1.5 readonly 与 const

| | `const` | `readonly` |
|---|---|---|
| 求值时机 | **编译期**，值直接替换到调用处 | **运行期**（声明时或构造函数中） |
| 类型限制 | 只能是基元类型、string、null | 任意类型 |
| 跨程序集 | 值被内联，改值后调用方需重新编译 | 运行时读取，改值后无需重编调用方 |

```csharp
public const float Pi = 3.14159f;          // 编译期常量
public readonly Vector3 SpawnPos;          // 可在构造函数中赋值

public MyClass(Vector3 pos) { SpawnPos = pos; }
```

Unity 中一个实际影响：`const` 的 public 字段不会出现在 Inspector，也不参与序列化；`readonly` 字段同样不参与序列化。

### 1.6 泛型约束

.NET 支持五种类型参数约束：

```csharp
where T : struct            // T 必须是值类型（不含可空类型）
where T : class             // T 必须是引用类型
where T : new()             // T 必须有无参构造函数
where T : NameOfBaseClass   // T 必须继承自指定类
where T : NameOfInterface   // T 必须实现指定接口
```

组合时 `new()` 必须放在最后：`where T : class, new()`。`struct` 与 `class` 互斥，不能同时使用。

```csharp
public T Create<T>() where T : ScriptableObject
{
    return ScriptableObject.CreateInstance<T>();
}
```

`ScriptableObject` 及各类 `UnityEngine.Object` 派生类型必须用 `CreateInstance<T>()` 创建：`new T()` 只在托管侧建出对象，没有对应的原生对象。因此这类约束里也不要写 `new()`。

## 2. 委托与事件

一句话概括：**委托是类型安全的委托引用（多播委托内部为委托引用数组），事件是建立在其上的订阅列表。**

### 2.1 委托的使用

```csharp
// 1. 声明委托，说明签名（参数个数与类型）
public delegate int SomeDelegate(string s, bool b);

// 2. 定义符合签名的函数
private int SomeFunction(string s, bool b) { return 0; }

// 3. 创建委托，把函数名作为参数传进去
SomeDelegate sd = new SomeDelegate(SomeFunction);

// 4. 通过委托实例调用
sd("somestring", true);
```

C# 2.0 之后可以简写为 `SomeDelegate sd = SomeFunction;`，方法组会自动转换。

### 2.2 事件的使用

```csharp
// 1. 声明委托
public delegate int SomeDelegate(string s, bool b);

// 2. 定义符合签名的函数
private int SomeFunction(string s, bool b) { return 0; }

// 3. 声明事件，指定它能接受的签名
public event SomeDelegate SomethingHappened;

// 4. 通过事件保存符合签名的函数（可以有多个）
myObj.SomethingHappened += new SomeDelegate(SomeFunction);

// 5. 在合适时机调用事件中保存的所有函数
if (SomethingHappened != null)
{
    foreach (SomeDelegate sd in SomethingHappened.GetInvocationList())
    {
        int result = sd("somestring", true);
    }
}
```

### 2.3 工程要点

- 事件只能在**声明它的类内部**触发（`+=`/`-=` 外部可用，`=` 和调用不行），这是 `event` 相对裸 delegate 字段的核心价值。
- `event` 用 `+=` 订阅，**必须成对 `-=` 取消**。Unity 中对象销毁而事件源仍存活时，未取消的订阅会阻止 GC 并引发回调到已销毁对象（`MissingReferenceException`）。典型做法是在 `OnEnable` 订阅、`OnDisable` 取消。
- 多播委托的返回值只保留最后一个订阅者的结果，需要每个返回值时必须用 `GetInvocationList()` 手动遍历（如上例）。
- 订阅顺序即调用顺序，不要依赖它表达逻辑优先级。
- 现代写法优先用 `Action` / `Func<T>` / `UnityEvent`，减少自定义 delegate 声明。

```csharp
public event Action<int> OnScoreChanged;

private void OnEnable()  { OnScoreChanged += HandleScore; }
private void OnDisable() { OnScoreChanged -= HandleScore; }
```

## 3. 可空类型

### 3.1 可空类型修饰符 `?`

引用类型可以用空引用表示"不存在"，值类型通常不行：

```csharp
string str = null;   // 合法
int i = null;        // 编译错误
int? i = null;       // 合法
DateTime? t = null;  // 合法
```

`T?` 是 `System.Nullable<T>` 的缩写，编译后即为 `System.Nullable<int>`。它只适用于值类型；引用类型上的 `?`（C# 8 可空引用类型）是编译期静态检查注解，不改变运行时行为。

判断与取值：

```csharp
if (i.HasValue) { int v = i.Value; }
int x = i ?? 0;              // 空合并
int y = i.GetValueOrDefault();
```

### 3.2 与 Unity 序列化的冲突

**Unity 不能序列化 `Nullable<T>`**。Inspector 上不会显示 `int?` 字段，也不会被保存。需要"可选值"语义时：

```csharp
[SerializeField] private bool hasRadius;
[SerializeField] private float radius;
```

或用一个可序列化的包装 struct。详见 `04_序列化与资源管理.md` 第 2.5 节。

### 3.3 三元运算符 `?:`

```csharp
x ? y : z
```

`x` 为 `true` 返回 `y`，否则返回 `z`，是省略 `if/else` 的表达式形式。注意 `y` 与 `z` 的类型会做统一转换，混用 `int` 与 `null` 时需要显式写成 `(int?)null`。

### 3.4 空合并运算符 `??`

用于为可空类型和引用类型提供默认值：左操作数不为 `null` 返回左，否则返回右。

```csharp
a ?? b
```

`??` 是**右结合**运算符，`a ?? b ?? c` 按 `a ?? (b ?? c)` 计算。

### 3.5 空条件运算符 `?.`

```csharp
int firstX = points.First().X;   // 未做 null 检查
```

传统写法需要层层判空，可读性很差：

```csharp
int? firstX = null;
if (points != null)
{
    var first = points.FirstOrDefault();
    if (first != null)
        firstX = first.X;
}
```

C# 6.0 引入 `?.` 后可写成：

```csharp
int? firstX = points?.FirstOrDefault()?.X;
```

对象为 `null` 时不执行后续成员访问，直接返回 `null`。

**易错点**：`?.` 的返回类型会变。当成员是 `struct` 时，`obj?.Member` 的结果是 `Nullable<T>`，而 `obj.Member` 是 `T`，两者不能直接赋值或参与同一重载。例如 `transform?.position` 得到 `Vector3?`。

配套的 `?[]` 用于索引器：`array?[0]`。

## 4. Unity 常用函数与运行时行为

### 4.1 游戏物体生命周期

| 回调 | 触发时机 |
|---|---|
| `Awake` | 物体实例化后、处于激活状态时调用；即使脚本组件未启用也会调用；总在 `Start` 之前 |
| `OnEnable` | 物体与脚本组件启用时调用（**可反复触发**） |
| `Start` | 物体与脚本组件启用后、第一次 `Update` 之前调用（只调用一次；关闭再打开不会重复触发） |
| `FixedUpdate` | 按固定帧率调用，主要用于刚体/物理运算 |
| `Update` | 每帧一次，帧率随硬件与逻辑复杂度变化 |
| `LateUpdate` | 每帧一次，在 `Update` 之后 |
| `OnDisable` | 物体/组件关闭时调用（**可反复触发**） |
| `OnDestroy` | 物体销毁时调用 |

核心结论：

- `OnEnable` / `OnDisable` 在 Inspector 勾选/取消勾选、以及进入/退出 Play Mode 时都会触发，适合做事件订阅与清理。
- `Awake` / `Start` 只触发一次。
- `FixedUpdate` 的频率由 `Edit > Project Settings > Time > Fixed Timestep` 决定。**每秒调用次数相等不代表每次调用的真实时间间隔相等**，可用 `Time.realtimeSinceStartup` 验证。
- `Maximum Allowed Timestep` 是平衡机制：限定单次 `FixedUpdate` 链路的最长耗时，避免物理追帧导致卡死。

### 4.2 深度参数与屏幕坐标

`_ZBufferParams` 与 `_ProjectionParams` 的完整推导见 `02_Shader与材质.md` 第 10 节。这里只列结论：

```hlsl
float LinearEyeDepth(float z, float4 zBufferParam);  // -> [near, far]
float Linear01Depth(float z, float4 zBufferParam);   // -> [0, 1]
float4 ComputeScreenPos(float4 clipPos);             // -> [0, w] 的齐次屏幕坐标
```

- `ComputeScreenPos` 把裁剪空间 xy 从 `[-w, w]` 映射到 `[0, w]`，zw 保持不变：

```hlsl
float4 o;
o.xy = clipPos.xy * 0.5 + clipPos.w * 0.5;
o.zw = clipPos.zw;      // 关键：w 必须原样带回，否则后续除法没有意义
```

- `o.xy` 是**齐次坐标**，范围 `[0, w]`，与分辨率无关；它不是像素坐标。换算关系：

```text
uv    = o.xy / o.w;                  // 归一化到 [0, 1]
pixel = o.xy / o.w * _ScreenParams.xy;  // 再乘分辨率才是像素坐标
```

即 `screenPos.x = x * 0.5 + w * 0.5`，而 `uv.x = screenPos.x / w`——直接把后者写成 `screenPosX / width * w` 会把分辨率与齐次 `w` 混为一谈。

- `tex2Dproj` 在采样前会除以 w，等价于手动 `screenPos.xy / screenPos.w` 后 `tex2D`。

### 4.3 相机矩阵

- `Matrix4x4.LookAt` 的结果与 `Camera.worldToCameraMatrix` **不一致**：后者是 OpenGL 风格，相机朝向 -z，物体顶点在相机空间中的 z 均为负值。
- 需要与 Shader 的 `UNITY_MATRIX_V` 对齐时，应使用 `camera.worldToCameraMatrix`，而不是自己拼 `LookAt`。

### 4.4 HDR 渲染

- Unity 通过 RenderTarget 的属性决定颜色空间转换；希望渲染 HDR 纹理时，需要把 RT 设置为**线性读写**（`GraphicsFormat.R16G16B16A16_SFloat` 等）。
- `Camera.Render()` 可以精确控制绘制时机，立即执行一次渲染。
- HDR RT 本身存的就是线性值；挂 tonemapping 后处理 override 是为了在输出阶段把高动态范围压到可显示的 LDR。不挂并不是"结果不是线性空间"，而是读回时表现为过曝、高光溢出（或按显示管线被直接截断）。

### 4.5 碰撞检测的固定间隔

Unity 默认约 **0.02s** 检测一次碰撞（与 Fixed Timestep 相关）。高速运动的物体若在两次检测之间完全穿过碰撞体，就会检测不到。解决方式：把 **Rigidbody 的 `collisionDetectionMode`** 设为 `Continuous` / `ContinuousDynamic`（这是 Rigidbody 上的属性，Collider 上没有），或加射线预判，或减小 Fixed Timestep。

### 4.6 三角函数与纹理坐标

- Unity 纹理坐标原点在**左下角**。
- `float acos(float x)`：反余弦，输入 `[-1, 1]`，输出 `[0, pi]`。
- `float atan2(float y, float x)`：返回 y/x 的反正切，弧度区间 `[-pi, +pi]`。

### 4.7 实例化与销毁对象

```csharp
public GameObject enemyPrefab;

void Start()
{
    for (int i = 0; i < 5; i++) Instantiate(enemyPrefab);   // 5 个实例都重叠在预制件原始位置
}
```

- 父节点与坐标：单参数版本没有父节点，实例落在场景根层级；带 parent 的重载默认保持世界坐标（`worldPositionStays = true`），要按父节点本地坐标摆放需传 `false`，或改用带 position 的重载：`Instantiate(prefab, pos, Quaternion.identity, transform)`。
- 时机：`Instantiate` 内部**同步**执行新对象的 `Awake`/`OnEnable`（对象激活时，返回时已跑完），`Start` 要等到下一帧 `Update` 前；所以 `Awake` 里不要依赖别人在 `Start` 中完成的初始化。
- `Destroy` 是**延时**的：真正删除发生在当前帧 Update 循环结束后，本帧内对象仍然存在、仍会收到回调。`Destroy(gameObject, 0.5f)` 只是"0.5s 后的帧末再删"，期间照常可访问；重复调用 `Destroy(obj, t)` 不会追加一次销毁（不会叠加排队，待核实）；但**已排定的延时无法取消**，需要可取消的延时请用协程 + 标志位。`Destroy(this)` 只移除组件、保留 GameObject，编辑器内应改用 `DestroyImmediate`。
- 延时按**缩放时间**推进：`Destroy(obj, t)` 的 `t` 与 `Time.timeScale` 相关，`timeScale = 0`（暂停）时计时暂停，销毁不会发生；恢复后才继续倒计时。

### 4.8 创建预制件与挂载脚本

预制件（Prefab）就是"存成资源的游戏对象模板"。

**创建预制件资源**：

- 把 Hierarchy 窗口中的游戏对象**拖入 Project 窗口**，即生成一个预制件资源（`.prefab`）；
- 这一操作同时把原始对象变成该预制件的**实例**：Hierarchy 中图标变蓝，Inspector 顶部出现 Prefab 栏。此后对实例的修改以 override 形式记录（字段旁的加粗/蓝条），可 Revert 回退或 Apply 写回资源。

**创建实例**：

- 把预制件资源从 Project 视图拖到 Hierarchy 或 Scene 视图，即创建它的一个实例；
- 也可以在运行时用脚本创建，见上一节的 `Instantiate`（注意父节点与世界坐标的默认行为）。

**脚本的挂载**：

- 脚本用于创建自定义组件，但**脚本在附加到游戏对象之前不会激活任何代码**——只新建 `.cs` 文件而不挂载，`Awake`/`Start`/`Update` 全都不执行；
- 附加方法：把脚本资源从 Project 拖到 Hierarchy 中的目标游戏对象上（也可拖到该对象 Inspector 的空白处，或用 Add Component 搜脚本名）。序列化字段只有在挂载后才会出现在 Inspector 中。

### 4.9 碰撞回调

`OnCollisionEnter` 要求双方都有 Collider、至少一方带**非 kinematic** 的 Rigidbody、且都不是 Trigger；Collider 勾选 `Is Trigger` 后改走 `OnTriggerEnter`。两个静态碰撞体（都没有 Rigidbody）之间、以及 kinematic 刚体撞静态碰撞体，都不会触发碰撞回调——物理层面也不产生接触。

```csharp
void OnCollisionEnter(Collision collision)
{
    if (collision.gameObject.CompareTag("Missile"))   // 优于 tag == "Missile"，无字符串分配
        Destroy(gameObject, 0.5f);
}
```

坑：tag 未在 Tag Manager 中登记会抛 `UnityException`；高速物体穿透见 4.5，接触点取 `collision.contacts[0].point`。

### 4.10 获取组件与跨脚本访问

组件是类的实例，先用 `GetComponent` 拿到引用，之后即可像在 Inspector 中一样读写属性、调用方法：

```csharp
void Start()
{
    Rigidbody rb = GetComponent<Rigidbody>();
    if (rb == null) return;          // 组件不存在时返回 null
    rb.mass = 10f;                   // 改属性（等同 Inspector 操作）
    rb.AddForce(Vector3.up * 10f);   // 调方法（Inspector 做不到）
}
```

- 同一物体上的自定义脚本同样用 `GetComponent<Enemy>()`；查找有开销，热路径应在 `Awake`/`Start` 中缓存。访问**其他**对象则声明 `public GameObject player;` 后从 Hierarchy 拖到 Inspector；组件间强依赖可用 `[RequireComponent(typeof(Rigidbody))]` 自动补齐。

### 4.11 输入：虚拟轴与按钮

```csharp
void Update()
{
    float h = Input.GetAxis("Horizontal"), v = Input.GetAxis("Vertical");   // 带平滑；原始值用 GetAxisRaw
    transform.Translate(new Vector3(h, v, 0) * moveSpeed * Time.deltaTime);
}
```

坑：`Transform.Translate(Vector3)` 有 `Space.Self` 与 `Space.World` 两个重载，**默认是 `Space.Self`**，所以 `(h, v, 0)` 被解释为物体的**本地坐标**——它落在本地 x/y 轴上，而不是前进方向。结果是物体不会"朝前走"，而是沿自身局部轴斜向平移，并且会随物体旋转一起转。三种正确写法：

```csharp
transform.Translate(Vector3.forward * v * moveSpeed * Time.deltaTime);                  // 本地 z 即朝向
transform.position += transform.forward * v * moveSpeed * Time.deltaTime;              // 等价，绕过 Translate
transform.Translate(new Vector3(h, 0, v) * moveSpeed * Time.deltaTime, Space.World);   // 明确要世界方向
```

### 4.12 协程

协程是 `IEnumerator`，`yield return` 标记暂停点，用 `StartCoroutine` 启动；事件型输入用 `Input.GetButtonDown`（只在按下那一帧为 `true`）判断，再 `StartCoroutine(Fade())`。

```csharp
IEnumerator Fade()
{
    Renderer r = GetComponent<Renderer>();   // 先取 Renderer 组件，再访问 material
    for (float a = 1f; a >= 0f; a -= 0.1f)
    {
        Color c = r.material.color;
        c.a = a;
        r.material.color = c;
        yield return null;                   // 暂停到下一帧再继续
    }
}
```

（原稿有误，已更正：`renderer.material` → `GetComponent<Renderer>().material`）

`renderer`、`transform` 这类组件简写在旧版本 C# 里可直接用，但 `renderer` 等快捷属性自 Unity 5 起已废弃（组件名冲突与性能原因），新代码一律显式 `GetComponent<Renderer>()`；`transform` 仍保留。

- `yield return null` 等一帧；`new WaitForSeconds(0.5f)` 等指定时长，受 `Time.timeScale` 影响（暂停时不推进），真实时间用 `WaitForSecondsRealtime`。
- 以字符串启动（`StartCoroutine("Fade")`）只能再用同名字符串停止，代价是每次按名字做**字符串查找**（并支持 `StopAllCoroutines` 这类粗粒度操作），且无法按句柄精确停止某一条；推荐保存 `Coroutine` 句柄后 `StopCoroutine(handle)`。
- 协程依附 MonoBehaviour：**GameObject 被禁用/销毁、或 MonoBehaviour 被销毁**时协程终止，重新启用不会续跑；单纯把组件的 `enabled` 置为 `false` 并不会终止协程，它仍会继续推进。`r.material` 会给每个渲染器实例化材质副本（破坏合批、可能泄漏），只读访问用 `sharedMaterial`，逐对象改参优先 `MaterialPropertyBlock`。

## 5. 编辑器与工程技巧

### 5.1 自定义 Inspector

```csharp
public class CloseUpShadowTest : MonoBehaviour
{
    public Vector3 CloseUpOffset;
    public bool inTest;
    public void StartTest() { }
    public void StopTest() { }
}

[CustomEditor(typeof(CloseUpShadowTest))]
public class CloseUpShadowTestEditor : Editor
{
    private SerializedProperty vector3Value;
    private GUIContent btnContent = new GUIContent();
    private Color backGround;

    private void OnEnable()
    {
        vector3Value = serializedObject.FindProperty("CloseUpOffset");
    }

    public override void OnInspectorGUI()
    {
        CloseUpShadowTest targetComp = (CloseUpShadowTest)target;

        serializedObject.Update();                 // 同步字段数值
        EditorGUILayout.PropertyField(vector3Value);

        backGround = GUI.backgroundColor;
        if (!targetComp.inTest)
        {
            btnContent.text = "开始测试";
            GUI.backgroundColor = Color.green;
        }
        else
        {
            btnContent.text = "停止测试";
            GUI.backgroundColor = Color.red;
        }

        if (GUILayout.Button(btnContent))
        {
            targetComp.inTest = !targetComp.inTest;
            if (targetComp.inTest) targetComp.StartTest();
            else                   targetComp.StopTest();
        }

        GUI.backgroundColor = backGround;
        serializedObject.ApplyModifiedProperties(); // 应用字段数值
    }
}
```

要点：

- `[CustomEditor(typeof(X))]` 把编辑器类关联到目标脚本。
- 修改字段优先走 `SerializedProperty` + `serializedObject.Update()/ApplyModifiedProperties()`，这样才能正确支持 Undo、Prefab Override 与多选编辑。
- 直接改 `target` 上的字段不会进 Undo 系统。

### 5.2 Shader 增加 Toggle 开关

```hlsl
Properties
{
    [Toggle] _LambertLit("Lambert Lit", Float) = 1.0
}
```

Unity 会自动生成"属性名全大写 + `_ON`"的宏 `_LAMBERTLIT_ON`，Pass 中声明即可：

```hlsl
#pragma shader_feature _ _LAMBERTLIT_ON
```

### 5.3 Hierarchy 按类型搜索

Hierarchy 搜索框支持 `t:` 前缀按类型过滤：

- `t:light`：搜索包含 Light 组件的对象
- `t:collider`：搜索包含 Collider 的对象

### 5.4 运行时获取 RendererFeature

```csharp
public UniversalRendererData rendererData;
private FSR2RendererFeature _fsr2RendererFeature;

public override void Create()
{
    if (rendererData == null)
    {
        Debug.LogError("Failed to configure UniversalRendererData");
        return;
    }

    _fsr2RendererFeature = rendererData.rendererFeatures
        .OfType<FSR2RendererFeature>()
        .FirstOrDefault();

    if (_fsr2RendererFeature == null)
    {
        Debug.LogError("Failed to get FSR2RendererFeature");
        return;
    }
}
```

### 5.5 ScriptableRenderPass 的执行时机

```csharp
public enum RenderPassEvent
{
    BeforeRendering = 0,
    BeforeRenderingShadows = 50,
    AfterRenderingShadows = 100,
    BeforeRenderingPrepasses = 150,
    AfterRenderingPrePasses = 200,
    BeforeRenderingGbuffer = 210,
    AfterRenderingGbuffer = 220,
    BeforeRenderingDeferredLights = 230,
    AfterRenderingDeferredLights = 240,
    BeforeRenderingOpaques = 250,
    AfterRenderingOpaques = 300,
    BeforeRenderingSkybox = 350,
    AfterRenderingSkybox = 400,
    BeforeRenderingTransparents = 450,
    AfterRenderingTransparents = 500,
    BeforeRenderingPostProcessing = 550,
    AfterRenderingPostProcessing = 600,
    AfterRendering = 1000,
}
```

标准顺序（枚举中的 210/220/230/240 四项只在 Gbuffer/Deferred 路径下才会命中）：

```text
Shadow -> PrePass -> Opaque -> Skybox -> Transparent -> PostProcessing
```

为什么是 `Opaque -> Skybox -> Transparent`：

- 先画 Opaque 再画 Skybox，符合不透明物体由近及远、可充分利用 Pre-Z 的优化思路；
- Transparent 必须排在 Skybox 之后，因为它要与已经确定的背景色混合。

### 5.6 Soft Shadow 生效条件

1. 产生阴影的光源（通常是主光源）的 Light 组件中，`Shadow Type` 设为 `Soft Shadows`；
2. `UniversalRenderPipelineAsset` 中勾选 `Soft Shadows`，对应字段：

```csharp
UniversalRenderPipelineAsset.supportsSoftShadows
```

（`supportsMainLightShadows` 是主光阴影的总开关，与软/硬阴影模式无关，不要拿它判断 Soft Shadow。）

相应参数（URP Asset 的 Shadows 分组）：

| 参数 | 值 |
|---|---|
| Max Distance | 100 |
| Working Unit | Metric |
| Cascade Count | 4 |
| Split 1 / 2 / 3 | 6 / 24 / 50 |
| Depth Bias | 1 |
| Normal Bias | 1 |
| Soft Shadows | ✔ |

### 5.7 其他实用结论

- **后处理参数是否需要插值**要区分对待：贴图的 ST 参数就不该参与插值。
- 场景模型勾选 `Static` 后，MeshRenderer 会自动增加 lightmapping 配置项。
- Unity 有两套 GI 系统：Baked GI（Lightmapping / Light Probe / Reflection Probe）与 Realtime GI（即将弃用）。
- 多相机（Base + Overlay）时，Overlay 相机没有 `Output` 属性，输出 Display 跟随所属 Base Camera；`FinalBlitPass` 只在最后一个 Overlay 相机执行，若开启后处理则没有 `FinalBlitPass`。
- SRP 中每个 Pass 可独立配置执行前是否 Clear 相关 RT。**FrameDebugger 中没有出现某个 Pass，通常是该 Pass 的对象都被过滤了，但它配置的 Clear 依然会执行**。

## 原始笔记索引

- `Unity/C#.md`
- `Unity/C# 委托 事件.md`
- `Unity/可空类型.md`
- `Unity/Unity函数.md`
- `Unity/unity使用.md`
- `杂项/unity记录.md`