# PBR 与 IBL

> 本文整合 PBR 白皮书要点、Unity/URP 中的 PBR 实现、金属流工作流、IBL（Diffuse/Specular）、球谐光照（SH）、环境贴图与 Cubemap 立体角、matcap 等内容。BRDF 的数学推导见《03_光照模型与BRDF》。

---

## 1. PBR 白皮书要点

基于物理的渲染（Physically Based Rendering）是指：使用基于物理原理与微平面理论建模的着色/光照模型，并使用从现实中测量的表面参数来准确表示材质的渲染理念。

![](../../技术笔记/assets/870FD55558B143C0A92D91F613BF6B0C.png)

它的五根支柱：

1. **微平面理论（Microfacet Theory）**：把物体表面建模为微观尺度上无数理想镜面小平面。
2. **能量守恒（Energy Conservation）**：出射光能量不超过入射光能量。
3. **菲涅尔反射（Fresnel）**：不同角度、不同物质有不同的反射率。
4. **线性空间（Linear Space）**：光照计算必须在线性空间执行；gamma 空间输入的贴图需先解码到线性空间。
5. **色调映射（Tone Mapping）**：把 HDR 亮度映射到显示设备有限的色域。

补充两条物质层面的认识：

- 现实物质可分为导体、半导体、绝缘体（渲染中简化为金属/非金属）。
- **非金属的镜面反射颜色是单色/灰的（F0 为 float），金属的镜面反射颜色是彩色的（F0 为 float3）**。

![](../../技术笔记/assets/EC4671D9EB684042BC26656E34BC100B.png)

PBR 的收益：物理真实、材质与光照解耦（同一套材质参数可适配不同 HDR 环境）、接口直观（粗糙度/金属度而非经验参数）、易于排错与扩展。

![](../../技术笔记/assets/FE670818219F4901B018014BD98B682C.jpg)

---

## 2. 材质工作流：金属流 vs 高光流

### 2.1 BaseColor

BaseColor（albedo）表示材质表面的反射率，其 RGB 三个通道分别缩放光线的 RGB 分量。例如 (1,0,0) 表示把白光三分量中的 R 保留、GB 置零。

金属-粗糙度工作流把金属与非金属统一到同一张 BaseColor 上，用 metallic 决定它的角色：

```hlsl
float3 f0           = 0.04;
float3 diffuseColor = baseColor.rgb * (1.0 - f0) * (1.0 - metallic);
float3 specularColor = lerp(f0, baseColor.rgb, metallic);
```

| metallic | diffuseColor | specularColor(F0) |
| --- | --- | --- |
| 0（非金属） | $baseColor \times (1-0.04)$ | 0.04 |
| 1（金属） | 0 | $baseColor$ |

所以：**BaseColor 在金属中充当镜面反射颜色（F0），在非金属中充当漫反射颜色**。

> 说明：乘 $(1-F_0)$ 是 Lagarde/URP 风格的近似（把部分能量让给镜面项），严格做法是逐像素用 $k_d=(1-F)(1-\text{metallic})$。两者在高光边缘会有可见差异，但前者省一次逐像素乘法，且能让 average albedo 保持在合理区间（待核实：该近似在金属度中间值时对 albedo 的偏移量）。

### 2.2 参数语义

- **Metallic**：0 = 电介质，1 = 导体。允许取中间值以模拟"金属表面覆盖灰尘/氧化层/油漆"的混合材质。
- **Roughness**：0 = 完美镜面，1 = 完全粗糙。直接决定 NDF 的 $\alpha$。
- **Smoothness**：Roughness 的反向表述，Unity 常用（`perceptualRoughness = 1 - smoothness`；注意 NDF 里的 linear roughness 是它的**平方**，见下文换算）。

三套粗糙度的换算（Unity/Unreal 混用时要特别小心）：

```hlsl
// PerceptualRoughness（美术滑块值） <-> Roughness（线性粗糙度）
half PerceptualRoughnessToRoughness(half pr) { return pr * pr; }
half RoughnessToPerceptualRoughness(half r)  { return sqrt(r); }

// Smoothness -> 上述两者
half SmoothnessToRoughness(half s)          { return (1 - s) * (1 - s); }
half SmoothnessToPerceptualRoughness(half s){ return (1 - s); }
```

对应关系：$r_{linear} = \alpha$，$r_{perceptual} = \sqrt{\alpha}$。**NDF 公式里的 $\alpha$ 用的是 linear roughness**（即美术滑块值的平方），搞混会让高光"过锐"或"过糊"。

常见贴图组合：BaseColor(RGB) + Metallic(R) + Roughness(G) + AO(B) 打包在一张图（ORM/MRAO）；或 BaseColor + 单独的 Smoothness(A) + Metallic(R)。

---

## 3. Unity / URP 中的 PBR 实现

### 3.1 数据结构

`UniversalFragmentPBR` 是 URP 的前向 PBR 入口。它吃两份数据：

```hlsl
struct SurfaceData          // 材质侧：由纹理采样与材质属性算得（简化示意；URP 实际结构体还含 specular/normalTS/clearCoat* 等字段）
{
    half3 albedo;
    half  metallic;
    half  smoothness;
    half3 emission;
    half  occlusion;
    half  alpha;
};

struct InputData             // 几何/插值侧：顶点着色器插值而来的片元属性
{
    float3  positionWS;
    float4  positionCS;
    float3  normalWS;        // 顶点法线 或 bumpMap 结果
    half3   viewDirectionWS;
    float4  shadowCoord;
    half    fogCoord;
    half3   vertexLighting;
    half3   bakedGI;
    float2  normalizedScreenSpaceUV;
    half4   shadowMask;
    half3x3 tangentToWorld;
};
```

最终输出再叠加雾效，并按是否 transparent 处理 alpha。

### 3.2 BRDF 参数拆分

```hlsl
float3 brdfDiffuse  = albedo * (1 - 0.04) * (1 - metallic);
float3 brdfSpecular = lerp(float3(0.04, 0.04, 0.04), albedo, metallic);
```

Unity 内部还会计算 `oneMinusReflectivity`（约 $1-\max(F_0)$）来约束 diffuse 的能量，并区分 BRDF1（GGX + Smith，高质量）与 BRDF2（Blinn-Phong 近似，低端平台）两条路径。

### 3.3 粗糙反射与环境贴图 Mip

用 cubemap 的 mip 链近似"粗糙度带来的模糊反射"，mip 级别不是线性映射：

```hlsl
float roughness = 1 - _Smoothness;
roughness *= 1.7 - 0.7 * roughness;                       // Unity 的 mip 映射
float4 envSample = UNITY_SAMPLE_TEXCUBE_LOD(
        unity_SpecCube0, reflectionDir,
        roughness * UNITY_SPECCUBE_LOD_STEPS);
```

`UNITY_SPECCUBE_LOD_STEPS` 决定 mip 范围。内置管线里它是**编译期常量 6**（定义在 `UnityStandardConfig.cginc`），与 cubemap 分辨率无关，对应 128px 探针的 mip 0–6（完整 mip 链是 0–7，即 128…1，6 只是最大卷积索引）；所以即便探针是 1024，`roughness=1` 也只会取到第 6 级，糊得不够。URP 的 Lit 主路径实际调用的是一参版 `PerceptualRoughnessToMipmapLevel(pr)`，内部仍是 `pr*(1.7-0.7*pr)*6`；两参版 `PerceptualRoughnessToMipmapLevel(pr, maxMipLevel)` 才按实际 mip 数归一。$1.7r - 0.7r^2$ 是经验拟合，用来让"模糊程度"随 roughness 视觉上更均匀。

![](../../技术笔记/assets/WEBRESOURCEc9934a0e85881e61ae272440478cf0d7.png)
![](../../技术笔记/assets/WEBRESOURCE0e06928d3085dbe862c36802792fad07.png)

### 3.4 金属与非金属的观感差异

- 两者都能产生清晰反射；但非金属的镜面反射不主导外观，diffuse 仍然可见。
- 金属会**给反射着色**，非金属不会——这对高光与环境反射都成立。

![](../../技术笔记/assets/WEBRESOURCE90ca0129c0c204d0c56290fa6e30131c.png)
![](../../技术笔记/assets/WEBRESOURCE5690c07ac847aee9baf8f9c8a25c400e.png)

### 3.5 反射与阴影

间接反射与直接光照无关，因此在阴影区依然存在：非金属表现为"整体更亮但仍能看到直射阴影"；金属则因间接反射占主导，随着光滑度上升，直接光与阴影几乎消失——**完美镜面上没有阴影**。现实中镜面上的灰尘、污渍会重新显现阴影，可通过 metallic 取中间值或单独的遮罩来模拟。

![](../../技术笔记/assets/WEBRESOURCE1a830842528505c8731b61779375ea5c.png)
![](../../技术笔记/assets/WEBRESOURCE31e5d08296bd39df30b2e9a2b3723904.png)
![](../../技术笔记/assets/WEBRESOURCEc78da6d254f21ca3cd28513d53ed36b7.png)

### 3.6 Box Projection（反射探针的局部校正）

Skybox 可以认为在无限远，反射方向相同的两个像素采样同一个 texel 是合理的。但**烘焙的 Reflection Probe 就在场景内**，必须做 box projection，否则近处反射会明显错位。

原理：把反射射线与探针的 AABB 求交，用"探针中心 → 交点"的向量作为采样方向。

```hlsl
float3 BoxProjection(float3 direction, float3 position,
                     float3 cubemapPosition, float3 boxMin, float3 boxMax)
{
    float3 factors = ((direction > 0 ? boxMax : boxMin) - position) / direction;
    float  scalar  = min(min(factors.x, factors.y), factors.z);
    return direction * scalar + (position - cubemapPosition);
}
```

探针的 box 始终是轴对齐的（忽略旋转与缩放），它有两个用途：决定物体使用哪个探针、做 box projection。

缺点：

1. 有畸变——只有从探针中心看才是正确的；
2. 反射范围与探针边界不一致时会错位，因此只有封闭场景（室内）效果稳定。

![](../../技术笔记/assets/WEBRESOURCE3b56134faf29ed1b971159de847ea704.png)
![](../../技术笔记/assets/WEBRESOURCE897ed02bc2db953a7e9782df79bcea42.png)
![](../../技术笔记/assets/WEBRESOURCE42a42aa0cfe86a53b721bbc6386ff89f.png)
![](../../技术笔记/assets/WEBRESOURCEe6a6d0394f7ec4a3ad86821841616cfc.png)

---

## 4. 环境贴图与 IBL 的基本假设

### 4.1 为什么需要 IBL

物体不会孤立存在，周围物体反射过来的光同样照亮它。直接光照系统只能用常数 Ambient 粗略模拟，IBL（Image Based Lighting）用一张环境贴图保存周围环境，从而得到方向相关的环境光照。

IBL 的核心简化：**环境（天空球）被视为无限大**，因此可以忽略位置、只保留法线信息——无论模型多大，着色点都当作位于天空球中心。这个假设是环境光预计算可行的前提。

### 4.2 环境贴图的三种形式

- **Cube Map**：6 张面，硬件支持采样与 mip，业界最常用；
- **Equirectangular Map（ERP，经纬展开图）**：单张 2:1 图，`atan/asin` 映射，非等面积也非保角，畸变较大；
- **Dual-Paraboloid Map**：两张抛物面图，单次采样但边缘质量差。

![](../../技术笔记/assets/452CFF433F434E60A2EFDB34CC2F09AC.jpg)
![](../../技术笔记/assets/33103C614078495D8E256A28E3A1D531.jpg)
![](../../技术笔记/assets/44778DF82EB44CD98A1DC639D6FB86E9.jpg)

ERP 采样成 CubeMap 的经典写法：

```glsl
vec2 sampling_equirectangular_map(vec3 n)
{
    float u = atan(n.z, n.x);           // [-PI, PI]
    u = (u + PI) / (2.0 * PI);          // [0, 1]

    float v = asin(n.y);                // [-PI/2, PI/2]
    v = (v * 2.0 + PI) / (2.0 * PI);    // [0, 1]
    return vec2(u, v);
}

// 等价写法
const vec2 invAtan = vec2(0.1591, 0.3183);
vec2 SampleSphericalMap(vec3 v)
{
    vec2 uv = vec2(atan(v.z, v.x), asin(v.y));
    uv *= invAtan;
    uv += 0.5;
    return uv;
}
```

实际做法：画一个球体贴上 ERP，再用 FOV=90° 的相机分别朝 ±X/±Y/±Z 渲染 6 个面，得到 HDR CubeMap。

![](../../技术笔记/assets/3D00228C85994409A568F022D5BEA095.png)
![](../../技术笔记/assets/40DDF33E0FCC4838B17DF11C0FEE91DA.png)
![](../../技术笔记/assets/2922F27C4889410FB516591C2CA3643B.png)

### 4.3 Cubemap 每个 texel 的立体角

对球面积分时，cubemap 上不同位置的像素投影到球面的面积不同，**不能等权求和**，必须计算每个 texel 的 solid angle。

单位球、面位于 $z=1$ 的平面上，texel 的立体角就是它在球面上的投影面积：

$$
\Omega = \iint \frac{dx\,dy}{(x^2+y^2+1)^{3/2}}
$$

积分有闭式解。先求从原点 $(0,0)$ 到面内任意点 $(s,t)$ 的定积分：

$$
f(s,t) = \int_{y=0}^{t}\int_{x=0}^{s} \frac{1}{(x^2+y^2+1)^{3/2}}\,dx\,dy
= \tan^{-1}\!\frac{st}{\sqrt{s^2+t^2+1}}
$$

写成带符号处理的四象限形式（$x,y$ 可为负，用 $\operatorname{atan2}$ 而不是 $\arctan$）：

$$
F(x,y) = \operatorname{atan2}\!\left(xy,\ \sqrt{x^2+y^2+1}\right)
$$

于是四边形 ABCD 的立体角为

$$
\Omega = F(A) - F(B) + F(C) - F(D)
$$

```cpp
double sphereQuadrantArea(double x, double y) {
    return std::atan2(x * y, std::sqrt(x * x + y * y + 1));
}

double solidAngle(size_t dim, size_t u, size_t v)
{
    const double iDim = 1.0f / dim;
    double s = ((u + 0.5) * 2 * iDim) - 1;   // texel 中心 -> [-1,1]
    double t = ((v + 0.5) * 2 * iDim) - 1;
    const double x0 = s - iDim, y0 = t - iDim;   // 半个 texel
    const double x1 = s + iDim, y1 = t + iDim;
    return sphereQuadrantArea(x0, y0) -
           sphereQuadrantArea(x0, y1) -
           sphereQuadrantArea(x1, y0) +
           sphereQuadrantArea(x1, y1);
}
```

![](../../技术笔记/assets/WEBRESOURCEadf2068bc50a89f0493c4ba34dd211f5.png)

用途：diffuse irradiance 卷积、SH 系数投影、任何"对整球求和/积分"的离线预计算。

---

## 5. IBL Diffuse

### 5.1 从渲染方程出发

$$
L_o = \int_\Omega f_r\, L_i\,(n\cdot\omega_i)\,d\omega_i
$$

直接光照只有有限个方向，环境光照需要对整个半球积分。对 diffuse 部分，$f_d = c/\pi$ 是常数，可以提出积分号：

$$
L_o^{diffuse} = k_d \cdot \frac{c}{\pi} \int_\Omega L_i\,(n\cdot\omega_i)\,d\omega_i
= k_d \cdot \frac{c}{\pi}\, E(n)
$$

其中 $E(n)$ 只与法线有关，可以离线预计算成一张 **irradiance cubemap**，运行时按法线采样即可。

### 5.2 Riemann Sum 预计算

球面坐标下 $d\omega = \sin\theta\,d\theta\,d\phi$，离散化：

$$
E(n) = \int_0^{2\pi}\!\!\int_0^{\pi/2} L_i\cos\theta\sin\theta\,d\theta\,d\phi
\;\approx\; \Delta\theta\,\Delta\phi \sum_{k} L_i(\theta_k,\phi_k)\cos\theta_k\sin\theta_k
$$

在 $\theta\in[0,\pi/2]$、$\phi\in[0,2\pi)$ 上均匀步进时 $\Delta\theta\Delta\phi = \pi^2/N$（仅在步长整除区间时严格成立：`samplingStep=0.025` 时偏低约 0.28%，0.1 时偏低约 1.95%）。下面代码最后乘的是 $\pi/N$ 而不是 $\pi^2/N$，即它存进 cubemap 的是 $E/\pi$——**近似**把 Lambert BRDF 的 $1/\pi$ 预先吸收进去，运行时直接 `kD * albedo * irradiance` 即可，无需再除以 $\pi$；要更准可改用中点采样（每个格子取中心方向）。

```glsl
const float PI = 3.14159265358979;

vec3 calc_cartesian(float phi, float theta)
{
    return vec3(sin(theta) * cos(phi), sin(theta) * sin(phi), cos(theta));
}

vec3 convolution_cube_map(samplerCube cube, int faceIndex, vec2 uv)
{
    // 由面索引 + uv 得到该 texel 对应的法线
    vec3 n = normalize(calc_normal(faceIndex, uv));
    // 构造切空间基（n 为 z 轴）
    vec3 up = vec3(0.0, 1.0, 0.0);
    vec3 r  = normalize(cross(up, n));
    vec3 u  = normalize(cross(n, r));

    float samplingStep = 0.025;
    int   sampler = 0;
    vec3  l = vec3(0.0);

    for (float phi = 0.0; phi < 2.0 * PI; phi += samplingStep) {
        for (float theta = 0.0; theta < 0.5 * PI; theta += samplingStep) {
            vec3 d = calc_cartesian(phi, theta);
            d = d.x * r + d.y * u + d.z * n;     // 切空间 -> 世界空间
            // L * (n·l) * sin(theta) d(theta) d(phi)
            l += filtering_cube_map(cube, normalize(d)) * cos(theta) * sin(theta);
            sampler++;
        }
    }
    return PI * l / float(sampler);              // 取平均后乘 PI
}
```

`calc_normal(face, uv)` 把 6 个面的 uv 映射为方向（+X: `n.x=1; n.zy=uv`，-X: `n.x=-1; n.z=-uv.x; n.y=uv.y`，以此类推）。

预计算尺寸可以很小（如 32×32）：diffuse 是低频信号，双线性插值足够，尺寸小可大幅降低预计算开销。

![](../../技术笔记/assets/CB1EB887312D4F38871359E65DACE75C.jpg)

### 5.3 运行时合成

```glsl
vec3 calc_ibl_diffuse(vec3 n, vec3 v, vec3 albedo, float roughness, float metalic)
{
    vec3 F0 = mix(vec3(0.04), albedo, metalic);
    vec3 F  = calc_fresnel_roughness(n, v, F0, roughness);

    vec3 kD = (vec3(1.0) - F) * (1.0 - metalic);
    vec3 irradiance = filtering_cube_map(glb_IrradianceMap, n);
    return kD * albedo * irradiance;
}
```

IBL 下没有唯一的半程向量，通常用 $n$ 代替 $h$ 计算 Fresnel；为了不丢掉粗糙度的影响，采用带 roughness 的变体：

$$
F = F_0 + \left(\max(1-r,\,F_0) - F_0\right)(1 - n\cdot v)^5
$$

```glsl
vec3 calc_fresnel_roughness(vec3 n, vec3 v, vec3 F0, float roughness)
{
    float ndotv = max(dot(n, v), 0.0);
    return F0 + (max(vec3(1.0 - roughness), F0) - F0) * pow(1.0 - ndotv, 5.0);
}
```

物理含义：粗糙表面的微平面朝向分散，掠射角下的"完美镜面"增强被削弱，所以 $F$ 的上限从 1 降到 $1-r$。

---

## 6. IBL Specular 与 Split-Sum 近似

### 6.1 问题

镜面项的积分同时包含 $L_i$、$D$、$G$、$F$，且依赖 $n$、$v$、roughness（反射方向 $r$ 由 $n$、$v$ 导出，不是独立维度），无法像 diffuse 一样直接预计算成一张小图。

### 6.2 Split-Sum 近似

把积分拆成两项相乘（Epic 的近似，核心假设是 **$n=v=r$**——此时 BRDF 项才只剩 $(F_0,\text{roughness})$ 两个自变量，可以打成 2D LUT）：

$$
L_o^{spec} \approx
\underbrace{\frac{\displaystyle\int_\Omega L_i(l)\,D(h)\,(n\cdot l)\,d\omega_i}
{\displaystyle\int_\Omega D(h)\,(n\cdot l)\,d\omega_i}}_{\text{预滤波环境图 }(r,\ \alpha)}
\times
\underbrace{\left(F_0\,A + B\right)}_{\text{BRDF LUT}}
$$

其中 $A, B$ 是对 $f_r$ 做 GGX 重要性采样后、把 $F$ 拆成 $F_0 + (1-F_0)(1-v\cdot h)^5$ 得到的两个标量积分，与 $F_0$ 无关，因此可以预先算成一张二维纹理。

- **预滤波环境图（Prefiltered Environment Map）**：对每个 roughness 级别，用 GGX 重要性采样对环境图做卷积，存进 cubemap 的 mip 链；
- **BRDF 积分 LUT**：二维表，横轴 $n\cdot v$，纵轴 roughness，输出 $(A,B)$ 两个通道，运行时 $F_0 A + B$。

两项相乘后得到完整的环境镜面反射：$L_{spec} = \text{PrefilteredEnv}(r, \text{rough}) \cdot (F_0 A + B)$。

### 6.3 GGX 重要性采样

GGX 法线分布函数（NDF，$\theta$ 为半程向量 $m$ 与法线的夹角）：

$$
D_{GGX}(m) = \frac{\alpha^2}{\pi\left((\alpha^2-1)\cos^2\theta + 1\right)^2}
$$

按 $D(h)(n\cdot h)$ 的重要性采样半程向量：

$$
\theta = \arctan\!\left(\alpha\sqrt{\frac{u_1}{1-u_1}}\right),\qquad
\phi = 2\pi u_2
$$

同一分布的 $\cos\theta$ 形式（与上式等价，由 $\tan^2\theta=\alpha^2 u_1/(1-u_1)$ 解 $\cos^2\theta = (1-u_1)/(1+u_1(\alpha^2-1))$ 即得）：

$$
\theta=\arccos\!\left(\sqrt{\frac{1-u_1}{u_1(\alpha^2-1)+1}}\right)
$$

（原图有误，已更正：原图分子写作 $\alpha^2$。按 $p_h = D(h)(n\cdot h)$ 反解 CDF 时分子应为 $1-u_1$；写成 $\alpha^2$ 相当于把 $u_1$ 与 $1-u_1$ 混用，采样密度不再匹配 NDF，蒙特卡罗估计会有偏。更正后的式子与本节 `SampleGGX` 中 `cosT = sqrt((1.0 - u.y) / (1.0 + (a2 - 1.0) * u.y))` 一致。）

然后由 $h$ 反射得到入射方向 $l = 2(v\cdot h)h - v$。这里要区分两个 PDF：

$$
p_h(h) = D(h)\,(n\cdot h),
\qquad
p(\omega_i) = \frac{D(h)\,(n\cdot h)}{4\,(v\cdot h)}
$$

$p_h$ 是半程向量上的密度（按投影面积加权的 NDF）；$p(\omega_i)$ 才是入射方向上的密度，由 $p_h$ 乘雅可比 $1/(4(v\cdot h))$ 得到。蒙特卡罗估计时除以哪个，取决于样本是在 $h$ 上抽的还是转成 $l$ 之后用——上式（第二式）含雅可比，**是对入射方向**的 PDF，不能标成"对半程向量"。

```hlsl
float3 SampleGGX(float roughness, float2 u, float3 n)
{
    float a  = roughness * roughness;
    float a2 = a * a;

    float phi   = 2.0 * PI * u.x;
    float cosT  = sqrt((1.0 - u.y) / (1.0 + (a2 - 1.0) * u.y));
    float sinT  = sqrt(1.0 - cosT * cosT);

    float3 h = float3(sinT * cos(phi), sinT * sin(phi), cosT);  // 切空间，z 轴为 n
    // Unity 没有 UNITY_MATRIX_T_MW；这里用 n 现场构造一组正交基把切空间转到世界空间
    // （有切线插值时直接用 TransformTangentToWorld(h, tangentToWorld)）
    float3 up = abs(n.y) < 0.999 ? float3(0, 1, 0) : float3(1, 0, 0);
    float3 tx = normalize(cross(up, n));
    float3 ty = cross(n, tx);
    return normalize(mul(h, float3x3(tx, ty, n)));              // 转到世界空间
}

float PdfGGX(float3 n, float3 h, float3 v, float roughness)
{
    float a2  = pow(roughness, 4.0);
    float NoH = saturate(dot(n, h));
    float VoH = saturate(dot(v, h));
    float d   = (NoH * a2 - NoH) * NoH + 1.0;
    float D   = a2 / (PI * d * d);
    return D * NoH / (4.0 * VoH);
}
```

> 早期笔记中的 SampleGGX 代码把 PDF 写成 `2 * alpha2 * NoH * sin / (a1*a1)`（slope-space 形式），与上式只是归一化空间不同，混用会导致蒙特卡罗估计偏亮/偏暗，**该实现待核实**。

### 6.4 重要性采样的作用

朴素均匀采样在光滑表面（$D$ 极尖锐）时几乎采不到高光方向，方差极大；重要性采样让样本集中在 $D$ 的峰值附近，同样的样本数下噪声显著降低。这也是离线预滤波能在可接受时间内完成的原因。

---

## 7. matcap（Material Capture）

Matcap 是用一张预渲染的"材质球"贴图，按**视空间法线的 XY** 采样，从而一次性获得光照 + 反射信息。它不是 PBR，但在固定视角、固定光照风格下非常高效。

原理：法线是单位向量，丢掉 Z 分量不会丢信息（朝向相机的半球内，Z 可由 $z=\sqrt{1-x^2-y^2}$ 推回。注意 $x,y$ 落在单位圆外时没有对应法线；而 $z<0$ 的**背向**法线，其 $x,y$ 仍落在圆内，会和正面贴图混叠——这是 matcap 只适用于近似朝前表面的原因）。XY 投影权重越大 → 越靠近贴图边缘；XY 越小（法线越正对相机）→ 越靠近贴图中心。

```hlsl
v2f vert (appdata v)
{
    v2f o;
    o.vertex = UnityObjectToClipPos(v.vertex);
    // UNITY_MATRIX_IT_MV：逆转置矩阵，把法线从模型空间变换到视空间
    float3 viewnormal = normalize(mul(UNITY_MATRIX_IT_MV, v.normal));
    o.matcapuv = viewnormal.xy * 0.5 + 0.5;   // [-1,1] -> [0,1]
    return o;
}

fixed4 frag (v2f i) : SV_Target
{
    return tex2D(_MatcapTex, i.matcapuv);
}
```

要点与局限：

- 必须 `normalize`，否则模型带缩放时法线超出 [-1,1]，采样会跑出贴图；
- 与视角强绑定：相机一动，反射就"粘"在物体上，不适合可自由旋转的相机；
- 无法响应真实光源与阴影，不能做能量守恒。

---

## 8. 球谐光照（SH）

### 8.1 为什么用 SH

环境光（尤其 diffuse irradiance）是低频函数，用球谐基函数做投影可以只存 9 个（3 阶，L2）系数就能很好地重建，代价远低于 cubemap 卷积，且天然支持"在空间中的插值"——这正是 Light Probe 的原理。

### 8.2 数学要点

用实球谐基 $Y_l^m(\omega)$ 展开辐射函数：

$$
L(\omega) \approx \sum_{l=0}^{2}\sum_{m=-l}^{l} c_l^m\, Y_l^m(\omega),
\qquad
c_l^m = \int_{S^2} L(\omega)\,Y_l^m(\omega)\,d\omega
$$

- Band 0（1 项）：常数项，即平均亮度；
- Band 1（3 项）：线性梯度，即主方向（类似一个方向光）；
- Band 2（5 项）：二次项，可表达"上下亮中间暗"等分布；
- Unity 的 `SphericalHarmonicsL2` 用 L2（9 系数 × RGB = 27 个 float）。

irradiance 的 SH 重建还要乘上余弦瓣的卷积系数（$l=0$ 保留，$l\ge1$ 衰减，$\hat{A}_0=\pi$、$\hat{A}_1=2\pi/3$、$\hat{A}_2=\pi/4$），所以运行时只需一次 9 项点积。

### 8.3 Unity 中的两个入口

**`RenderSettings.ambientProbe`**：自动创建的环境探针，捕获环境照明，保证默认的环境光影响场景与 GameObject。若某个对象被显式关联到 Light Probe，Unity 就不再使用这个全局探针（因为烘焙的 probe/lightmap 已包含环境光贡献）。

**`LightProbes.bakedProbes`**：烘焙后的探针系数数组，可读可写，用于程序化修改探针光照。

```csharp
SphericalHarmonicsL2[] bakedProbes = LightmapSettings.lightProbes.bakedProbes;
Vector3[] probePositions            = LightmapSettings.lightProbes.positions;
int probeCount                      = LightmapSettings.lightProbes.count;

for (int i = 0; i < probeCount; i++) bakedProbes[i].Clear();

// 添加环境光
for (int i = 0; i < probeCount; i++)
    bakedProbes[i].AddAmbientLight(m_Ambient);

// 添加平行光 / 点光源
foreach (Light l in m_Lights)
{
    if (l.type == LightType.Directional)
        for (int i = 0; i < probeCount; i++)
            bakedProbes[i].AddDirectionalLight(-l.transform.forward, l.color, l.intensity);
    else if (l.type == LightType.Point)
        for (int i = 0; i < probeCount; i++)
            SHAddPointLight(probePositions[i], l.transform.position, l.range,
                            l.color, l.intensity, ref bakedProbes[i]);
}

LightmapSettings.lightProbes.bakedProbes = bakedProbes;
```

点光源对探针的处理：从探针视角看，点光源与平行光没有区别，只是要乘距离衰减。

`AddDirectionalLight(direction, color, intensity)` 的 `direction` 参数是**从探针指向光源**的归一化方向（Unity 文档原文："The direction from the light probe to the light"）。由此：

- 平行光的 `transform.forward` 是"光源 → 表面"的传播方向，**指向光源**的方向是它的反向，所以要传 `-l.transform.forward`；
- 点光源的 `position - probePosition` 本身就是"探针 → 光源"，与平行光的 `-forward` 符号一致，不需要再取负。

两种光源传入的都是"探针 → 光源"，此处不存在符号矛盾；容易混淆的是把它记成"光线传播方向（光源 → 表面）"，那样会把平行光的符号写反。

```csharp
void SHAddPointLight(Vector3 probePosition, Vector3 position, float range,
                     Color color, float intensity, ref SphericalHarmonicsL2 sh)
{
    Vector3 probeToLight = position - probePosition;
    float attenuation = 1.0F / (1.0F + 25.0F * probeToLight.sqrMagnitude / (range * range));
    sh.AddDirectionalLight(probeToLight.normalized, color, intensity * attenuation);
}
```

（`1/(1+25d²/r²)` 是 Unity 的点光衰减近似式，具体系数随版本可能不同，**待核实**。）

### 8.4 Unity 环境光的来源

`Environment Lighting → Source` 有三种：**Skybox**（天空盒作为光源）、**Gradient**（三色天光）、**Color**（纯色）。环境光指的就是环境**漫反射**光，Unity 内部统一用 SH 表示。

- 环境 SH 系数始终存在：以 Skybox 为例，设置好天空盒后 3 阶系数就已经产生；
- 若场景中放置了 Light Probe，则必须烘焙，才会在这些位置生成新的 SH 系数。

![](../../技术笔记/assets/F3E43C9EB3B14A9ABE817FD2BEB80821.png)

### 8.5 SH Visualizer：把基函数画出来

Unity 运行时实际只用到 9 个系数（$l\le2$）；下面的可视化工具为了看清高阶形状，把 16 个基函数（Band 0~3）全部画出来。

用全屏 Ray Marching + SDF 可视化这 16 个基函数的 3D 形状，分 5 步：

1. 全屏三角形 + 构建相机射线（转相机时屏幕颜色随视角变化）；
2. Ray Marching 球体 + 法线光照（中央出现有明暗的球）；
3. 加 AABB 包围盒（球被盒子切割）；
4. 实现 16 个 SH 基函数，按金字塔排列，从上到下复杂度递增（球 → 偶极子 → 四叶 → 六瓣）；
5. 正负瓣着色 + 天光照射 + 环境光遮蔽。

![](../../技术笔记/assets/WEBRESOURCEf49d286c1521e9a209098b9cbdde48d5.png)
![](../../技术笔记/assets/WEBRESOURCEc4903d1ee7981efb3d52b4cadb5a0474.png)
![](../../技术笔记/assets/WEBRESOURCE7dddcd25b5beb1feccf04ee1ce115d10.png)
![](../../技术笔记/assets/WEBRESOURCE40fa3597edfbca182cdcc6f51ea1d39c.png)
![](../../技术笔记/assets/WEBRESOURCE611fb07a270b3ea93ee5cf5fb02080f0.png)
![](../../技术笔记/assets/WEBRESOURCE0b978338087d253089e4f4a4b2e0a2ea.png)

> 更完整的推导与实现见 [15_球谐光照](15_球谐光照.md)。

---

## 9. 全局光照中的采样与积分（补充）

- **Monte Carlo**：$\int f(x)dx \approx \frac{1}{N}\sum f(x_i)/p(x_i)$，样本服从 $p$；$p$ 越接近 $f$ 的形状，方差越小。
- **Riemann Sum**：把定义域均匀切分求和，实现简单，适合低频/离线（IBL diffuse 预计算）；粗糙度低（高频）时效率很差。
- **Importance Sampling**：按 $D(h)(n\cdot h)$ 采样，用于 IBL specular 与路径追踪的 BRDF 采样。

递归光线追踪的基本结构（UC Davis 课程）：`First_hit` 找最近交点 → 对每个光源发 shadow ray 判断可见性 → 累加直接光照 → 递归计算反射方向。这对应渲染方程的 Neumann 级数展开，只是把"无限次弹射"截断。

---

## 10. 实现清单（自查）

1. 环境贴图是 HDR（float16/32），不是 8bit LDR；
2. diffuse 与 specular 使用不同的预计算产物（irradiance map vs prefiltered env + BRDF LUT）；
3. prefilter 的 mip 与 roughness 的映射要和运行时一致（否则高光"发飘"）；
4. IBL 的 $G$ 用 $k=\alpha/2=r^2/2$，直接光用 $k=(\alpha+1)^2/8=(r^2+1)^2/8$（$r$ 为感知粗糙度，$\alpha=r^2$，两种光照的 $k$ 不是同一个量）；$k$ 越大 $G_1$ 越小，两种 $k$ 只在 $\alpha=1$ 处相等（均为 $0.25$）；$\alpha\to0$ 时分别趋于 $1/8$ 与 $0$，比值发散；
5. 金属没有 diffuse，$k_d$ 必须乘 $(1-\text{metallic})$；
6. Reflection Probe 非无限远时必须做 box projection；
7. 最终输出在线性空间完成 tone mapping，再做 sRGB 编码。

---

## 原始笔记索引

- `Graphics/PBR/PBR白皮书.md`
- `Graphics/BaseColor.md`
- `Graphics/PBR/Roughness and Smoothness.md`
- `Graphics/PBR/PBR.md`
- `根目录笔记/URP中的PBR.md`
- `Unity/pbr in unity.md`
- `Unity/Unity PBR金属流.md`
- `Graphics/IBL/IBL的环境光.md`
- `Graphics/IBL/基于图像的光照(Image Based Lighting)（Diffuse篇）.md`
- `Graphics/IBL/matcap.md`
- `Graphics/PBR/Equirectangular Projection(ERP).md`
- `Graphics/Solid Angle Of Cubemap Texel.md`
- `Graphics/SH/SH.md`
- `Graphics/SH/LightProbes.bakedProbes.md`
- `Graphics/SH/RenderSettings.ambientProbe.md`
- `根目录笔记/SH Visualizer.md`
- `Graphics/Global illumination/SampleGGX.md`
- `Graphics/Global illumination/UC Davis.md`
- `Graphics/PBR/蒙特卡罗尔积分.md`