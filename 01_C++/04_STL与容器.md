# STL 与容器

> 导读：本篇按"容器 → 适配器 → 迭代器 → 算法/字符串处理"的顺序整理 STL 相关内容。重点在于弄清各容器的内存布局与复杂度差异（尤其是 `vector` 的扩容假象与 `deque` 的分段连续），以及 `iterator`/`const_iterator`、`hash_map`/`unordered_map` 这些容易混淆的概念。最后几节是日常使用频率很高的字符串与循环写法。

## 一、容器体系总览

```
STL
├─ 容器
│  ├─ 顺序容器
│  │  ├─ vector        随机访问迭代器，尾部增删均摊 O(1)
│  │  ├─ deque         随机访问迭代器，头尾增删 O(1)
│  │  ├─ list          双向迭代器，任意位置增删 O(1)
│  │  └─ string        随机访问，字符序列
│  ├─ 关联容器（红黑树，有序）
│  │  ├─ set / multiset
│  │  └─ map / multimap        双向迭代器，查找 O(log N)
│  └─ 无序容器（哈希表，C++11）
│     ├─ unordered_set / unordered_multiset
│     └─ unordered_map / unordered_multimap
├─ 容器适配器
│  ├─ stack            默认底层 deque
│  ├─ queue            默认底层 deque
│  └─ priority_queue   默认底层 vector + 堆算法
├─ 迭代器
│  ├─ 输入 / 输出
│  ├─ 前向
│  ├─ 双向            list、set、map
│  └─ 随机访问        vector、deque、string
└─ 算法
   ├─ 只读算法         find、count、accumulate
   ├─ 修改序列算法     copy、replace、fill、remove
   ├─ 排序/重排        sort、stable_sort、nth_element
   └─ 关系算法         equal、max/min
```

## 二、`vector`

### 2.1 初始化与赋值的几种方式

```cpp
// 1) 默认构造：size 为 0
std::vector<int> v1;

// 2) 指定 size：元素值初始化（int 为 0）
std::vector<int> v2(10);

// 3) 指定 size 和初值
std::vector<int> v3(10, 1);          // 10 个 1

// 4) 用数组区间初始化（左闭右开）
int a[5] = {1, 2, 3, 4, 5};
std::vector<int> v4(a, a + 5);

// 5) 拷贝构造
std::vector<int> v5(v3);

// 6) 用 insert 插入区间
std::vector<int> src(6, 6);
std::vector<int> v6;
v6.insert(v6.begin(), src.begin(), src.begin() + 3);   // 插入 3 个元素

// insert 也可以接数组地址区间
int b[6] = {6, 6, 6, 6, 6, 6};
std::vector<int> v7;
v7.insert(v7.begin(), b, b + 6);      // 原笔记写成 b + 7，越界

// 在指定位置插入 n 个相同值
v7.insert(v7.begin(), 6, 6);          // 开头插入 6 个 6

// 7) 用 copy 赋值（目标区间必须已存在足够空间）
std::vector<int> dst(10);
int c[5] = {2, 2, 2, 2, 2};
std::copy(v3.begin(), v3.end(), dst.begin());
std::copy(c, c + 5, dst.begin() + 5);
```

> 易混淆点：`std::vector<int> v(10)` 是"10 个 0"，而 `std::vector<int> v{10}` 是"1 个 10"（`initializer_list` 构造优先）。两者语义完全不同。

### 2.2 `insert`

```cpp
_vertices.insert(_vertices.end(), _vertices.begin(), _vertices.end());
```

三个参数都是迭代器：第一个是插入位置（元素插在该位置之前），后两个是源区间（左闭右开）。上面这行的字面意思是"把整个数组再追加一份"。

> 但要注意：标准对 `vector::insert(pos, first, last)` 要求 `first`/`last` **不能是同一个容器内的迭代器**，否则行为未定义。libstdc++ 等实现通常会先把区间拷贝到临时缓冲区而"碰巧能用"，但不应依赖。需要自我追加时，先拷贝一份，或改用下标循环配合 `reserve`：

```cpp
std::vector<int> tmp(_vertices);                 // 先复制
_vertices.insert(_vertices.end(), tmp.begin(), tmp.end());
```

### 2.3 `clear()` 与容量释放

```cpp
void clear();   // 删除所有元素，size 变为 0
```

- `clear()` **不改变 capacity**，也不释放内存，只是把元素析构、把 size 置 0；
- 若想真正释放容量，用 swap 技巧：

```cpp
std::vector<int>().swap(vec);   // 或 vec.swap(std::vector<int>());
```

- 如果容器里存的是**指针**，swap 之前要先遍历并 `delete` 每个元素指向的对象，否则只释放了指针数组本身。

### 2.4 在构造函数初始化列表中初始化 `vector`

成员是 `vector` 时，最自然的做法是在初始化列表里直接给定大小：

```cpp
class A
{
public:
    A() : aaa(30)          // 构造出 30 个 float（值初始化为 0）
    {
        aaa[0] = 1.0f;
        aaa[3] = 2.0f;
    }

    std::vector<float> aaa;
};

int main()
{
    A a;
    for (int i = 0; i < 30; ++i)
        std::cout << a.aaa[i] << std::endl;
}
```

若只写 `A() {}` 而不初始化 `aaa`，容器为空，随后 `aaa[0]` 是越界访问。

也可以在声明处直接给定大小：`std::vector<float> aaa = std::vector<float>(30);`。但不要写成 `std::vector<float> aaa{30};`——花括号会优先匹配 `initializer_list` 构造，得到的是"1 个值为 30 的元素"，与"30 个 0"完全不同。

## 三、`deque`

### 3.1 先从 `vector` 的"生长假象"说起

`vector` 可以向后生长，但这个生长是假象，实际过程是：

1. 申请一块更大的连续内存；
2. 把原有元素复制/移动到新内存；
3. 释放原内存。

因此 `vector` 的扩容是均摊 O(1)，但单次扩容可能是 O(N)，并且会导致所有迭代器、指针、引用失效。

### 3.2 `deque` 的分段连续结构

`deque`（double-ended queue）允许在两端高效增删：

- 支持随机访问，可用 `[]` 和 `at()`；
- 头尾插入/删除都是 O(1)，中部插入/删除较慢（O(N)）；
- 存储空间由一段段**等长的连续缓冲区**组成，各段之间不保证连续；
- 用一个**指针数组**（通常称为 `map`，注意不是关联容器 `map`）记录每段缓冲区的首地址，从而在分段内存上维护"整体连续"的假象。

<img src="../../技术笔记/assets/5860A21A703D41C9845996B0A796F08A.png" width="327" height="230">

需要在头部或尾部增加空间时：

1. 申请一段新的连续空间；
2. 在 `map` 数组的头/尾添加指向它的指针。

> 提示：
>
> 1. 这里的 `map` 是"主控数组"的名称，与 `std::map` 无关；
> 2. `map` 数组本身满了怎么办？按 `vector` 扩容的方式处理：申请更大的区域、拷贝旧指针、释放旧区域。
> 3. 缓冲区大小由实现决定。libstdc++ 中通常按 512 字节设计（元素较大时一个缓冲区只放一个元素），不同标准库实现并不一致。

### 3.3 `deque` 的迭代器

```cpp
template <class T, class Ref, class Ptr, ...>
struct __deque_iterator
{
    T* cur;            // 当前元素
    T* first;          // 当前缓冲区的首地址
    T* last;           // 当前缓冲区的末尾（最后一个元素的下一个位置）
    map_pointer node;  // 二级指针，指向 map 中“指向当前缓冲区的那个指针”
};
```

- `cur`：当前遍历到的元素；
- `first` / `last`：当前连续缓冲区的首尾，用于判断是否走到边界；
- `node`：类型等价于 `T**`，指向主控数组中保存当前缓冲区地址的槽位。

为什么需要 `first` / `last`？因为迭代器 `++`/`--` 到达缓冲区边缘时，必须跳到 `map` 中的下一段/上一段缓冲区，而不能简单地指针加减。

### 3.4 `deque` 的底层成员

```cpp
template <class T, class Alloc = std::allocator<T>>
class deque
{
protected:
    iterator start;    // 第一个缓冲区：cur 指向首个元素
    iterator finish;   // 最后一个缓冲区：cur 指向末元素的下一个位置
    map_pointer map;   // 主控指针数组
    size_type map_size;
};
```

<img src="../../技术笔记/assets/524578389E0A4087B3AA934ADBA99FD5.png" width="444" height="370">

`start` 和 `finish` 两个迭代器就足以描述整个 deque 的有效区间 `[start, finish)`。

### 3.5 复杂度小结

| 操作 | `vector` | `deque` | `list` |
|:---|:---|:---|:---|
| 随机访问 | O(1) | O(1) | O(N) |
| 尾部插入 | 均摊 O(1) | O(1) | O(1) |
| 头部插入 | O(N) | O(1) | O(1) |
| 中部插入 | O(N) | O(N) | O(1)（已定位） |
| 内存连续性 | 连续 | 分段连续 | 非连续 |

## 四、容器与容器适配器

`vector` 是容器，`stack` 是容器适配器。

`stack` 只暴露栈相关的接口：`push()`、`pop()`、`top()`、`empty()`、`size()`。使用者只关心这些语义，不关心底层用的是哪个容器；更换底层容器时，调用方代码无需修改。这符合 SOLID 中的接口隔离原则（Interface Segregation Principle）。

`std::stack` 可适配的标准容器有 `vector`、`list`、`deque`，默认是 `deque`——它提供 O(1) 的 `push_back`/`pop_back`，且在扩容时不像 `vector` 那样需要整体搬移。

同理：

- `queue` 需要 `front`、`back`、`push_back`、`pop_front`，可适配 `deque`、`list`，**不能**用 `vector`；
- `priority_queue` 需要随机访问以维护堆，底层默认 `vector`，也可用 `deque`。

## 五、迭代器

### 5.1 五种迭代器类别

```
        input        output
           \          /
            \        /
             forward
                |
           bidirectional
                |
          random access
```

这张图描述的不是继承关系，而是**接口能力的扩张**：下层迭代器拥有上层的全部能力，并额外支持更多操作。

| 类别 | 能力 |
|:---|:---|
| 输入迭代器（input） | 只读、单向、单遍；只能逐个读取，不能重复遍历同一序列 |
| 输出迭代器（output） | 只写、单向、单遍；只能逐个写入 |
| 前向迭代器（forward） | 可读可写、单向、可多遍遍历，能保存位置 |
| 双向迭代器（bidirectional） | 在前向基础上支持 `--` |
| 随机访问迭代器（random access） | 支持 `+n`、`-n`、`it1 - it2`、`[]`、比较大小 |

容器与迭代器类别：

| 容器 | 迭代器类别 |
|:---|:---|
| `vector` | 随机访问 |
| `deque` | 随机访问 |
| `string` | 随机访问 |
| `list` | 双向 |
| `set` / `multiset` | 双向 |
| `map` / `multimap` | 双向 |
| `stack` / `queue` / `priority_queue` | 不提供迭代器 |

### 5.2 `iterator`、`const_iterator` 与 `const iterator`

三者容易混淆，关键区别在于 **const 修饰的是"迭代器本身"还是"所指元素"**。

| 类型 | 能否改变指向 | 能否修改元素 |
|:---|:---|:---|
| `iterator` | 可以 | 可以 |
| `const_iterator` | 可以 | 不可以（解引用得到 `const T&`） |
| `const iterator` | 不可以（必须初始化） | 可以 |

```cpp
std::vector<int> ivec(10, 0);

std::vector<int>::const_iterator citer1 = ivec.begin();
const std::vector<int>::iterator citer2 = ivec.begin();

*citer1 = 1;   // 错误：*citer1 是 const int&
*citer2 = 1;   // 正确：citer2 指向的元素不是 const
++citer1;      // 正确：citer1 自身可以改指向
++citer2;      // 错误：citer2 自身是 const
```

进一步说明：

- 若传入的是 `const` 容器，只能用 `const_iterator` 遍历：

  ```cpp
  void Print(const std::vector<int>& v)
  {
      for (std::vector<int>::const_iterator it = v.begin(); it != v.end(); ++it)
          std::cout << *it << std::endl;
  }
  ```

- `const_iterator` 既可以用于 `const` 容器，也可以用于非 `const` 容器；`const iterator` 只能用于非 `const` 容器（因为它要修改元素）；
- `const iterator` 几乎没有任何实用价值：初始化后不能改指向，只能改它唯一指向的那个元素，因此强烈不推荐使用。

### 5.3 为什么 `set` / `map` 的迭代器是"常量"的

```cpp
std::set<int> s{1, 2, 3};
std::set<int>::iterator it = s.begin();
// *it = 10;      // 错误

std::map<int, int> m{{1, 2}};
auto mit = m.begin();
// mit->first = 3;   // 错误：key 是 const
mit->second = 3;     // 正确：value 可以改
```

`set` 的 `iterator` 与 `const_iterator` 实质上是同一种类型（标准规定 `set::iterator` 是常量迭代器）；`map` 的 `value_type` 是 `std::pair<const Key, T>`，因此 key 不可改，mapped value 可以改。

> 原始笔记把这归因于"Visual Studio 2010 以上版本"的实现细节，实际这是 C++ 标准的要求，与编译器无关。

原因：`set`/`map` 底层是红黑树，元素位置由 key 决定。若允许原地修改 key，会破坏树的有序性，导致查找、遍历全部失效。

那如何"修改"元素？只能 `erase()` 旧元素再 `insert()` 新元素，复杂度 O(log N)；这也是 `map` 把 key 设为 const、而 value 保持可变的直接原因。

## 六、哈希容器

### 6.1 `hash_map` 与 `unordered_map`

早期 C++ 标准库没有哈希表容器，各家实现自造了非标准的 `hash_map`（SGI STL、MSVC、GCC 扩展等），接口和性能保证存在细微差别。

C++11 将其纳入标准，但为了避免与已有代码冲突，采用了新名字 `unordered_map`。这个名字也更准确：它明确表示元素是**无序**的。

### 6.2 `unordered_map` 的原理

底层结构是 **hash table + bucket**：

<img src="../../技术笔记/assets/41780173C0F245468B9AAC615BC2E3D5.png" width="620" height="346">

插入过程：

1. 取得 key；
2. 通过 hash 函数得到哈希值；
3. 计算桶号（通常是哈希值对桶数取模）；
4. 把 key/value 放入桶中，冲突时用比较函数处理（链地址法等）。

查找过程：

1. 取得 key；
2. 计算哈希值；
3. 定位桶号；
4. 在桶内逐个比较 key；
5. 相等则取出 value，否则表示不存在。

特点：

- **关联性**：通过 key 检索 value，而不是像顺序容器那样依赖绝对位置；
- **无序性**：内部顺序不确定，遍历顺序与插入顺序无关；
- **键唯一性**：`unordered_map` 中 key 唯一，`unordered_multimap` 允许重复 key；
- **动态内存管理**：桶数组与节点由容器自行管理。

### 6.3 `map` 与 `unordered_map` 的对比

| 维度 | `map` | `unordered_map` |
|:---|:---|:---|
| 底层结构 | 红黑树（平衡二叉搜索树） | 哈希表（桶数组 + 链表/开放寻址） |
| 元素顺序 | 按 key 有序（中序遍历即排序结果） | 无序 |
| 查找 | O(log N) | 平均 O(1)，最坏 O(N) |
| 插入/删除 | O(log N) + 平衡开销 | 平均 O(1)，可能触发 rehash |
| 最差情况 | 稳定为 O(log N) | 全部冲突时退化为 O(N) |
| 迭代器稳定性 | 插入不影响其他迭代器 | rehash 可能导致迭代器失效 |
| 适用场景 | 需要有序遍历、范围查询、稳定最坏性能 | 只做点查、追求平均性能 |

`map` 的 O(log N) 保证了良好的最差情况性能；`unordered_map` 则是"上限很高、下限很低"——哈希分布好时接近 O(1)，哈希质量差或桶数不足时可能退化。

### 6.4 `hash_set` 与 `unordered_set`

- `unordered_set` 在 C++11 进入标准库，`hash_set` 没有；
- MSVC 同时提供了两者；libstdc++ 只在 `__gnu_cxx` 扩展中提供 `hash_set`（头文件 `<ext/hash_set>`，已废弃），标准接口只有 `unordered_set`；
- 因此在 GCC/Clang 环境下使用 `hash_set` 可能找不到头文件，跨平台代码应统一使用 `unordered_set`。

## 七、常用字符串与算法操作

### 7.1 `string::find`

`find` 返回 `std::string::size_type`，表示子串首次出现的位置（从 0 开始）；未找到时返回 `std::string::npos`。

```cpp
std::string str("abcdefg");
std::string::size_type pos = str.find("abc");

if (pos != std::string::npos)
    std::cout << "found at " << pos << std::endl;
else
    std::cout << "not found" << std::endl;
```

`npos` 的定义等价于 `size_type(-1)`，即该无符号类型的全 1 值：

- 32 位 `size_type` 下是 `4294967295`；
- 64 位平台（如 LP64 的 Linux/macOS）下是 `18446744073709551615`。

所以不要把它当成固定的 `4294967295`。

> 原始笔记中的示例把条件写反了：`if (pos != npos)` 时是**找到了**，却输出了 "Not find"。上面已修正。

不推荐的写法：

```cpp
int pos = str.find("abc");
if (pos < 0) { /* ... */ }        // 依赖 unsigned -> int 的截断，不可移植

if (str.find("abc") < 0) { }      // 错误：无符号数与 0 比较永远为 false
```

第二种写法是真正的 bug：`find` 返回无符号类型，`< 0` 永远不成立。应始终与 `npos` 比较，并用 `size_type` 接收返回值，以适配不同平台。

相关函数：

| 函数 | 含义 |
|:---|:---|
| `s.find(s2)` | 子串第一次出现的位置 |
| `s.rfind(s2)` | 子串最后一次出现的位置 |
| `s.find_first_of(s2)` | `s2` 中任一字符第一次出现的位置 |
| `s.find_last_of(s2)` | `s2` 中任一字符最后一次出现的位置 |
| `s.find_first_not_of(s2)` | 第一个不在 `s2` 中的字符位置 |
| `s.find_last_not_of(s2)` | 最后一个不在 `s2` 中的字符位置 |

### 7.2 `std::remove_if`

```cpp
template <class ForwardIt, class UnaryPredicate>
ForwardIt remove_if(ForwardIt first, ForwardIt last, UnaryPredicate p);
```

语义：把 `[first, last)` 中**不满足** `p` 的元素依次前移，保持相对顺序；满足 `p`（即"要删除"）的元素不会被搬到尾部，只是被前面保留的元素覆盖。返回新的逻辑尾迭代器 `new_end`，`[new_end, last)` 中的值未指定，通常是那些被前移元素的残留旧副本。

> 原始笔记写成"回调返回为真则将当前元素移到尾部"，方向容易理解反：谓词返回 `true` 表示**该元素要被移除**，真正被前移保留的是谓词返回 `false` 的元素。

`remove_if` 只接收迭代器，拿不到容器本身，因此**无法真正改变容器大小**，必须配合容器的 `erase` 使用，这就是 erase–remove 惯用法：

```cpp
vec.erase(std::remove_if(vec.begin(), vec.end(), pred), vec.end());
```

参考实现：

```cpp
template <class ForwardIt, class UnaryPredicate>
ForwardIt remove_if(ForwardIt first, ForwardIt last, UnaryPredicate p)
{
    first = std::find_if(first, last, p);
    if (first != last)
        for (ForwardIt i = first; ++i != last; )
            if (!p(*i))
                *first++ = std::move(*i);
    return first;
}
```

示例一：删除字符串中的所有空格。

```cpp
#include <algorithm>
#include <iostream>
#include <string>

bool isSpace(char x) { return x == ' '; }

int main()
{
    std::string s("Text with    spaces");
    std::cout << "before: " << s << std::endl;
    s.erase(std::remove_if(s.begin(), s.end(), isSpace), s.end());
    std::cout << "after : " << s << std::endl;   // Textwithspaces
}
```

示例二：删除 `vector` 中所有等于 `"banana"` 的元素。

```cpp
#include <algorithm>
#include <iostream>
#include <string>
#include <vector>

int main()
{
    std::vector<std::string> str = {"apple", "banana", "key", "cat", "dog", "orange", "banana"};
    const std::string target = "banana";

    str.erase(std::remove_if(str.begin(), str.end(),
                             [&target](const std::string& n) { return n == target; }),
              str.end());

    for (const auto& s : str)
        std::cout << s << " ";    // apple key cat dog orange
}
```

> 原始笔记的示例二把 `erase(remove_if(...))` 写了两遍，第一次的 `sd` 已经完成删除，第二次再对已缩短的区间操作属于重复逻辑，上面的版本只保留一次。
>
> 另外，若只是删除"等于某值"的元素，直接用 `std::remove` 更贴切；`remove_if` 适合条件更复杂的情况。

### 7.3 字符串分割

方式一：用 `find` + `substr` 按字符串分隔符切分。

```cpp
void SplitString(const std::string& s,
                 std::vector<std::string>& v,
                 const std::string& delim)
{
    std::string::size_type pos1 = 0;
    std::string::size_type pos2 = s.find(delim);

    while (pos2 != std::string::npos)
    {
        v.push_back(s.substr(pos1, pos2 - pos1));
        pos1 = pos2 + delim.size();
        pos2 = s.find(delim, pos1);
    }

    if (pos1 != s.length())
        v.push_back(s.substr(pos1));
}
```

特点：不修改原串，支持多字符分隔符；但连续分隔符会产生空串，且末尾分隔符不会产生尾部空串（按需处理）。

方式二：C 风格 `strtok`，可一次指定多个分隔字符。

```c
#include <string.h>
#include <stdio.h>

int main(void)
{
    char s[] = "a,b*c,d";
    const char* sep = ",*";
    char* p = strtok(s, sep);

    while (p)
    {
        printf("%s ", p);
        p = strtok(NULL, sep);
    }
    printf("\n");       // 输出: a b c d
    return 0;
}
```

注意：

- `strtok` 会**原地修改**输入串，因此不能传字符串字面量（上面用 `char s[]` 而非 `char* s = "..."`）；
- 它使用内部静态状态保存进度，**不可重入、非线程安全**；多线程或嵌套分割应使用 `strtok_r`（POSIX）或 `std::string` 方案。

### 7.4 range-for 中的 `auto` 与引用

```cpp
std::string s("hello world");

for (auto c : s)      // 按值拷贝，改的是副本
    c = 't';
std::cout << s << std::endl;    // hello world（未改变）

for (auto& c : s)     // 引用，改的是原元素
    c = 't';
std::cout << s << std::endl;    // ttttttttttt
```

选择原则：

| 写法 | 语义 | 适用 |
|:---|:---|:---|
| `for (auto x : c)` | 拷贝 | 元素很小（`int`、`char`、指针） |
| `for (const auto& x : c)` | 只读引用 | 只读遍历，尤其元素较大（`string`、自定义类） |
| `for (auto& x : c)` | 可写引用 | 需要修改元素 |
| `for (auto&& x : c)` | 转发引用 | 泛型代码，保留值类别 |

range-for 的本质是 `begin()`/`end()` 的语法糖；遍历过程中不要修改容器大小（增删元素会导致迭代器失效）。

### 7.5 循环中的几点优化

**（1）循环内定义 vs 循环外定义对象**

```cpp
// 写法 A：循环外构造
ClassTest ct;
for (int i = 0; i < 100; ++i)
{
    ct = a;
    // do something
}

// 写法 B：循环内构造
for (int i = 0; i < 100; ++i)
{
    ClassTest ct = a;
    // do something
}
```

- A：1 次构造 + 100 次赋值 `operator=`；
- B：100 次拷贝构造 + 100 次析构。

哪种更快取决于类本身：若赋值比"构造 + 析构"便宜，A 更快；否则 B 更快。现代 C++ 更强调"把变量定义在离使用最近的地方"，可读性和正确性优先；确有性能问题时再测量后调整。

**（2）大循环拆分与 cache**

```cpp
// 写法 A
for (int i = 0; i < n; ++i) { fun1(); fun2(); }

// 写法 B（fun1、fun2 无关）
for (int i = 0; i < n; ++i) fun1();
for (int i = 0; i < n; ++i) fun2();
```

若 `fun1`/`fun2` 代码量很小，A 少做 n 次循环自增与判断，通常更快；若两者代码量都很大（超过指令 cache 容量），B 更容易保持指令与数据的局部性，减少 cache 抖动，可能显著更快。这不是语言层面的规则，而是与硬件 cache 相关的经验结论，应以实测为准。

**（3）循环内的重复计算**

把循环不变量提到循环外：

```cpp
// 差
for (size_t i = 0; i < vec.size(); ++i) { /* ... */ }

// 好（若循环中不改变大小）
const size_t n = vec.size();
for (size_t i = 0; i < n; ++i) { /* ... */ }
```

对 `vector::size()` 这种 O(1) 内联函数，编译器通常能自行提升，此写法主要是表达意图、避免在循环中无意触发重新计算。

**（4）关于 `nums[i] = nums[j]` 的说明**

原始笔记中有这样一段：

```cpp
int i = 0;
for (int j = 0; j < nums.size(); j++)
{
    nums[i] = nums[j];
    i++;
}
```

笔记称"`nums[i]` 等于 `nums[j]`，指向同一地址，复制不会增加额外负担"——这个说法不准确。这是**值拷贝**：每个元素被逐个赋给前面的槽位（典型的原地压缩写法），并非"指向同一地址"。对 `int` 这类标量，拷贝成本可以忽略，编译器还可能做向量化；但若元素是较大的对象，就应改用移动语义，或先判断 `if (i != j)` 避免无谓的自赋值。

## 八、预处理技巧

### 8.1 用宏表示字符串

把标识符转成字符串要用 `#`（stringizing）：

```cpp
#define BUBBLE_PREFIX ""              // 前缀
#define STRINGIFY(x)  #x              // 字符串化
#define VERTFILE(f)   STRINGIFY(f.glslv)

static const char shad[] = BUBBLE_PREFIX VERTFILE(bubble_vert);
// 展开为 ""
//        "bubble_vert.glslv"
// 相邻字符串字面量会自动拼接，最终 shad = "bubble_vert.glslv"
```

> 原始笔记注释里写成 `bubble_vertf.glslv`，多了一个 `f`，属于笔误。

两个注意点：

1. `#x` 只做字符串化，**不会展开参数里的宏**：

   ```cpp
   #define VERSION 3
   STRINGIFY(VERSION)   // "VERSION"
   ```

   需要先展开再字符串化时，加一层间接宏：

   ```cpp
   #define XSTR(x) STRINGIFY(x)
   XSTR(VERSION)        // "3"
   ```

2. 依赖 `f.glslv` 这种"参数 + `.` + 后缀"的拼接可读性较差，更稳妥的写法是让后缀成为独立的字符串字面量：

   ```cpp
   #define VERTFILE(f) STRINGIFY(f) ".glslv"
   ```

### 8.2 `#undef`

`#undef` 取消一个宏定义，之后的代码再使用该名字就不再是宏：

```cpp
#include <iostream>
using namespace std;

#define MODI 10

int main()
{
    cout << MODI << endl;      // 10

#undef MODI

    // cout << MODI + 1 << endl;   // 编译错误：MODI 未定义
    return 0;
}
```

典型用途是**限定宏的作用范围**：

```cpp
#define MAX 50
#include "common.h"
#undef MAX
```

这样 `MAX` 只在 `common.h` 内部有效，不会污染后续代码，也能避免与别处同名宏冲突。写库或跨模块头文件时，这是控制宏污染的常用手段。

## 九、在链表中插入节点

先看一段错误代码：

```cpp
ImageLoaderBase* p = _loaderHead;
while (p)
{
    p = p->_next;
}
p = loader;          // 错误：只是把局部指针 p 指向了 loader
```

问题在于：循环结束时 `p` 是 `nullptr`，`p = loader` 只修改了**局部变量 p 的值**，链表最后一个节点的 `_next` 仍然是 `nullptr`，新节点根本没有接上。

正确做法是让循环停在最后一个节点，再修改它的 `_next`：

```cpp
ImageLoaderBase* p = _loaderHead;
while (p->_next)          // 停在最后一个节点
{
    p = p->_next;
}
p->_next = loader;
```

完整、健壮的尾插还应处理空链表，并把新节点的 `_next` 置空：

```cpp
void AppendLoader(ImageLoaderBase* loader)
{
    if (loader == nullptr)
        return;

    loader->_next = nullptr;

    if (_loaderHead == nullptr)
    {
        _loaderHead = loader;
        return;
    }

    ImageLoaderBase* p = _loaderHead;
    while (p->_next)
        p = p->_next;

    p->_next = loader;
}
```

要点：**改链表靠的是修改节点的 `_next` 指针，而不是修改遍历用的局部指针**。同理，若要在中间插入，需要先定位到"前驱节点"，再执行 `loader->_next = prev->_next; prev->_next = loader;`。

## 原始笔记索引

- `C++/C++/vector的几种初始化及赋值方式.md`
- `C++/C++/vector对象在显示列表的初始化.md`
- `C++/C++/vector insert函数的使用.md`
- `C++/C++/deque.md`
- `C++/C++/STL container 和 adapter.md`
- `C++/C++/STL.mindmap.md`
- `C++/C++/STL迭代器.md`
- `C++/C++/iterator 和 const_iterator.md`
- `C++/C++/iterator与const_iterator及const iterator区别.md`
- `C++/数据结构/hash_map ≈ unordered_map.md`（位于 `C++/数据结构/`）
- `C++/C++/hash_set与 unordered_set.md`
- `C++/C++/string find函数.md`
- `C++/C++/std  remove_if (begin,end,op).md`
- `C++/C++/for循环中auto的使用.md`
- `C++/C++/for循环优化.md`
- `C++/C++/字符串分割.md`
- `C++/C++/用宏表示字符串.md`
- `C++/C++/#undef 的用法及其意义.md`
- `C++/C++/在链表中插入节点.md`