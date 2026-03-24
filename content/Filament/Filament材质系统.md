# Filament 材质系统详解

## 概述

Filament 的材质系统基于**基于物理的渲染 (PBR)** 原理设计，提供了多种材质模型以适应不同的表面类型。

```
材质系统层次:

┌─────────────────────────────────────────────────────────┐
│                  Material Definition                     │
│                    (.mat 文件)                           │
│                      │                                   │
│                      ▼                                   │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Material Block                      │    │
│  │  • shadingModel  • parameters  • variants       │    │
│  └─────────────────────────────────────────────────┘    │
│                      │                                   │
│                      ▼                                   │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Vertex Block                        │    │
│  │  • 顶点变换  • 自定义顶点属性                      │    │
│  └─────────────────────────────────────────────────┘    │
│                      │                                   │
│                      ▼                                   │
│  ┌─────────────────────────────────────────────────┐    │
│  │             Fragment Block                       │    │
│  │  • BRDF 计算  • 材质参数设置                      │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## 材质模型 (Shading Models)

### 1. lit - 标准光照材质

最常用的 PBR 材质，模拟金属和非金属表面。

```mat
material {
    name : StandardMaterial,
    shadingModel : lit,
}

fragment {
    void material(inout MaterialInputs material) {
        prepareMaterial(material);
        
        material.baseColor.rgb = float3(0.8, 0.2, 0.2);
        material.roughness = 0.4;
        material.metallic = 0.0;
    }
}
```

**参数:**

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `baseColor` | float3 | [0, 1] | 漫反射颜色 (绝缘体) 或反射率 (金属) |
| `roughness` | float | [0, 1] | 表面粗糙度，0=镜面，1=完全粗糙 |
| `metallic` | float | [0, 1] | 金属度，0=绝缘体，1=纯金属 |
| `reflectance` | float | [0, 1] | 绝缘体的 F0 反射率 |
| `emissive` | float4 | [0, +∞] | 自发光颜色和强度 |
| `ambientOcclusion` | float | [0, 1] | 环境光遮蔽因子 |
| `normal` | float3 | [-1, 1] | 法线贴图扰动 |

---

### 2. unlit - 无光照材质

不受光照影响，用于 UI、粒子、全息图等。

```mat
material {
    name : UnlitMaterial,
    shadingModel : unlit,
}

fragment {
    void material(inout MaterialInputs material) {
        prepareMaterial(material);
        
        material.baseColor = texture(materialParams_albedo, getUV0());
        // 无 roughness, metallic 等参数
    }
}
```

**参数:**
- `baseColor` - 基础颜色
- `emissive` - 自发光 (可用于颜色强度 > 1 实现 HDR 发光)

---

### 3. subsurface - 次表面散射材质

模拟光线穿透表面并在内部散射的效果，如皮肤、蜡、玉等。

```mat
material {
    name : SkinMaterial,
    shadingModel : subsurface,
}

fragment {
    void material(inout MaterialInputs material) {
        prepareMaterial(material);
        
        material.baseColor.rgb = float3(1.0, 0.8, 0.7);
        material.roughness = 0.3;
        
        // 次表面散射专用参数
        material.subsurfaceColor = float3(1.0, 0.3, 0.2);
        material.subsurfacePower = 12.0;
        material.subsurfaceThickness = texture(materialParams_thickness, getUV0()).r;
    }
}
```

**额外参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `subsurfaceColor` | float3 | 散射颜色 |
| `subsurfacePower` | float | 散射衰减强度 |
| `subsurfaceThickness` | float | 材质厚度 |

---

### 4. cloth - 布料材质

模拟天鹅绒、丝绸等织物的表面。

```mat
material {
    name : VelvetMaterial,
    shadingModel : cloth,
}

fragment {
    void material(inout MaterialInputs material) {
        prepareMaterial(material);
        
        material.baseColor.rgb = float3(0.6, 0.1, 0.1);
        material.roughness = 0.8;
        
        // 布料专用参数
        material.sheenColor = float3(1.0, 0.9, 0.9);
        material.subsurfaceColor = float3(0.3, 0.05, 0.05);
    }
}
```

**额外参数:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `sheenColor` | float3 | 光泽颜色 (边缘发光效果) |
| `subsurfaceColor` | float3 | 漫反射散射颜色 |

---

### 5. clear_coat - 清漆材质

模拟汽车漆、涂层塑料、木地板等表面。

```mat
material {
    name : CarPaintMaterial,
    shadingModel : lit,
    variants : [ clearCoat ],  // 启用清漆层
}

fragment {
    void material(inout MaterialInputs material) {
        prepareMaterial(material);
        
        // 底层材质
        material.baseColor.rgb = float3(0.1, 0.2, 0.6);
        material.roughness = 0.4;
        
        // 清漆层
        material.clearCoat = 1.0;
        material.clearCoatRoughness = 0.1;
        material.clearCoatNormal = texture(materialParams_clearCoatNormal, getUV0()).xyz;
    }
}
```

**额外参数:**

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `clearCoat` | float | [0, 1] | 清漆层强度 |
| `clearCoatRoughness` | float | [0, 1] | 清漆层粗糙度 |
| `clearCoatNormal` | float3 | [-1, 1] | 清漆层法线 |

---

### 6. anisotropic - 各向异性材质

模拟拉丝金属、CD、头发等表面，反射随方向变化。

```mat
material {
    name : BrushedMetal,
    shadingModel : lit,
    variants : [ anisotropic ],
}

fragment {
    void material(inout MaterialInputs material) {
        prepareMaterial(material);
        
        material.baseColor.rgb = float3(0.9, 0.9, 0.9);
        material.metallic = 1.0;
        material.roughness = 0.3;
        
        // 各向异性参数
        material.anisotropy = 0.8;
        material.anisotropyDirection = float2(1.0, 0.0);
    }
}
```

**额外参数:**

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `anisotropy` | float | [-1, 1] | 各向异性强度和方向 |
| `anisotropyDirection` | float2 | 归一化 | 各向异性方向 (切线空间) |

---

## 材质属性详解

### 混合模式 (Blending)

```mat
material {
    blending : opaque,      // 不透明 (默认)
    // 或
    blending : transparent, // 透明 (Alpha Blend)
    // 或
    blending : fade,        // 淡出 (Alpha 影响光照)
    // 或
    blending : add,         // 叠加 (Additive)
    // 或
    blending : multiply,    // 正片叠底 (Multiply)
    // 或
    blending : screen,      // 滤色 (Screen)
}
```

| 模式 | 说明 |
|------|------|
| `opaque` | 不透明物体，启用深度测试和写入 |
| `transparent` | 标准透明混合，不写入深度 |
| `fade` | 类似 transparent，但 Alpha 也影响镜面反射 |
| `add` | 加法混合，用于发光效果 |
| `multiply` | 乘法混合，用于变暗效果 |
| `screen` | 屏幕混合，用于变亮效果 |

---

### 双面渲染

```mat
material {
    culling : back,     // 背面剔除 (默认)
    // 或
    culling : front,    // 正面剔除
    // 或
    culling : none,     // 双面渲染
}

// 或动态控制
fragment {
    void material(inout MaterialInputs material) {
        material.doubleSided = true;
        // 双面时，反面法线自动翻转
    }
}
```

---

### 深度控制

```mat
material {
    depthWrite : true,   // 写入深度缓冲 (默认)
    depthCulling : true, // 深度测试 (默认)
    
    // 自定义深度比较
    depthCulling : less,     // 小于通过 (默认)
    depthCulling : equal,    // 等于通过
    depthCulling : greater,  // 大于通过
    depthCulling : always,   // 总是通过
}
```

---

### 着色器变体 (Variants)

```mat
material {
    // 启用特定功能变体
    variants : [
        skinning,        // 骨骼动画
        shadowReceiver,  // 接收阴影
        shadowCascades,  // 级联阴影
        fog,             // 雾效
        vsm,             // 方差阴影
        ssr,             // 屏幕空间反射
        stereo,          // VR立体渲染
        
        // 材质特性
        clearCoat,       // 清漆层
        anisotropic,     // 各向异性
    ],
}
```

---

## 顶点属性

### 必需属性

```mat
material {
    requires : [
        uv0,        // 第一套 UV
        uv1,        // 第二套 UV (光照贴图等)
        color,      // 顶点颜色
        position,   // 顶点位置 (默认)
        tangents,   // 切线空间 (法线贴图必需)
    ],
}
```

### 自定义顶点属性

```mat
material {
    // 声明自定义顶点属性
    variables : [
        float3 myCustomData
    ],
}

vertex {
    void materialVertex(inout MaterialVertexInputs material) {
        // 访问顶点输入
        float3 localPos = getPosition();
        float2 uv = getUV0();
        float4 color = getColor();
        
        // 修改位置 (例如: 顶点动画)
        material.worldPosition.x += sin(getPosition().y * 10.0 + getTime()) * 0.1;
        
        // 传递数据到片元着色器
        material.myCustomData = computeSomething(localPos);
    }
}

fragment {
    void material(inout MaterialInputs material) {
        // 使用顶点传递的数据
        float3 data = getMaterialVertexMyCustomData();
    }
}
```

---

## 内置函数和变量

### 常用函数

```glsl
// 纹理采样
float4 texture(sampler2D sampler, float2 uv);

// UV 获取
float2 getUV0();                    // 第一套 UV
float2 getUV1();                    // 第二套 UV

// 法线计算
float3 getWorldTangentVector();     // 世界空间切线
float3 getWorldBitangentVector();   // 世界空间副切线
float3 getWorldNormalVector();      // 世界空间法线

// 位置
float3 getWorldCameraPosition();    // 相机世界位置
float3 getWorldPosition();          // 当前片元世界位置
float3 getPosition();               // 模型空间位置

// 时间
float getTime();                    // 当前时间 (秒)

// 雾
float getFogLevel();                // 雾强度 [0, 1]
```

### 结构体

```glsl
// 顶点输入
struct MaterialVertexInputs {
    float4 position;        // 裁剪空间位置 (必须设置)
    float3 worldPosition;   // 世界空间位置
    float4 color;           // 顶点颜色
    float2 uv0;             // UV0
    float2 uv1;             // UV1
    // ... 其他自定义变量
};

// 片元输入
struct MaterialInputs {
    float4 baseColor;           // 基础颜色
    float3 normal;              // 法线 (切线空间)
    float roughness;            // 粗糙度
    float metallic;             // 金属度
    float reflectance;          // 反射率
    float ambientOcclusion;     // 环境光遮蔽
    float4 emissive;            // 自发光
    float3 sheenColor;          // 光泽颜色 (cloth)
    float3 subsurfaceColor;     // 次表面颜色
    float subsurfacePower;      // 次表面强度
    float subsurfaceThickness;  // 次表面厚度
    float clearCoat;            // 清漆层
    float clearCoatRoughness;   // 清漆粗糙度
    float3 clearCoatNormal;     // 清漆法线
    float anisotropy;           // 各向异性
    float2 anisotropyDirection; // 各向异性方向
    float2 clearCoatRoughnessAnisotropy; // 清漆各向异性
    bool doubleSided;           // 双面
    bool flipNormal;            // 翻转法线
    bool hasClearCoatNormal;    // 是否有清漆法线
    bool hasRefraction;         // 是否有折射
};
```

---

## 完整材质示例

### 1. 标准 PBR 材质

```mat
material {
    name : StandardPBR,
    shadingModel : lit,
    
    parameters : [
        { type : sampler2d, name : albedo },
        { type : sampler2d, name : normalMap },
        { type : sampler2d, name : roughnessMap },
        { type : sampler2d, name : metallicMap },
        { type : sampler2d, name : aoMap },
        { type : float, name : normalIntensity },
        { type : float2, name : textureTiling }
    ],
    
    requires : [
        uv0,
        tangents
    ]
}

vertex {
    void materialVertex(inout MaterialVertexInputs material) {
        material.uv0 *= materialParams.textureTiling;
    }
}

fragment {
    void material(inout MaterialInputs material) {
        prepareMaterial(material);
        
        float2 uv = getUV0();
        
        // Albedo
        material.baseColor = texture(materialParams_albedo, uv);
        
        // Normal map
        float3 normal = texture(materialParams_normalMap, uv).xyz * 2.0 - 1.0;
        normal.xy *= materialParams.normalIntensity;
        material.normal = normalize(normal);
        
        // Roughness
        material.roughness = texture(materialParams_roughnessMap, uv).r;
        
        // Metallic
        material.metallic = texture(materialParams_metallicMap, uv).r;
        
        // Ambient Occlusion
        material.ambientOcclusion = texture(materialParams_aoMap, uv).r;
    }
}
```

### 2. 流动水面材质

```mat
material {
    name : Water,
    shadingModel : lit,
    
    parameters : [
        { type : sampler2d, name : normalMap },
        { type : float, name : waveSpeed },
        { type : float, name : waveScale },
        { type : float3, name : waterColor },
        { type : float, name : transparency }
    ],
    
    requires : [ uv0, tangents ],
    blending : transparent,
    
    variables : [
        float3 worldPosition
    ]
}

vertex {
    void materialVertex(inout MaterialVertexInputs material) {
        material.worldPosition = getWorldPosition();
    }
}

fragment {
    void material(inout MaterialInputs material) {
        prepareMaterial(material);
        
        float time = getTime();
        float2 uv = getUV0();
        
        // 两层流动的法线贴图
        float2 uv1 = uv + float2(time * materialParams.waveSpeed, 0.0);
        float2 uv2 = uv * 0.5 - float2(0.0, time * materialParams.waveSpeed * 0.7);
        
        float3 normal1 = texture(materialParams_normalMap, uv1).xyz * 2.0 - 1.0;
        float3 normal2 = texture(materialParams_normalMap, uv2).xyz * 2.0 - 1.0;
        
        material.normal = normalize(normal1 + normal2);
        material.normal.xy *= materialParams.waveScale;
        
        material.baseColor.rgb = materialParams.waterColor;
        material.roughness = 0.1;
        material.metallic = 0.0;
        
        material.baseColor.a = materialParams.transparency;
    }
}
```

### 3. 全息投影材质

```mat
material {
    name : Hologram,
    shadingModel : unlit,
    
    parameters : [
        { type : float3, name : hologramColor },
        { type : float, name : scanlineSpeed },
        { type : float, name : flickerIntensity },
        { type : sampler2d, name : scanlinePattern }
    ],
    
    requires : [ uv0 ],
    blending : add,
    depthWrite : false,
    culling : none
}

fragment {
    void material(inout MaterialInputs material) {
        prepareMaterial(material);
        
        float time = getTime();
        float2 uv = getUV0();
        
        // 扫描线效果
        float scanline = sin(uv.y * 50.0 + time * materialParams.scanlineSpeed);
        scanline = smoothstep(0.3, 0.7, scanline);
        
        // 闪烁效果
        float flicker = 1.0 + sin(time * 10.0) * materialParams.flickerIntensity;
        
        // 边缘发光
        float fresnel = 1.0 - abs(dot(getWorldNormalVector(), 
                                      normalize(getWorldCameraPosition() - getWorldPosition())));
        fresnel = pow(fresnel, 2.0);
        
        material.baseColor.rgb = materialParams.hologramColor * scanline * flicker * fresnel;
        material.baseColor.a = 1.0;
    }
}
```

---

## 编译和使用

### 编译材质

```bash
# 编译单个材质
matc -p mobile -a vulkan -o output.filamat input.mat

# 参数说明:
# -p, --platform : mobile | desktop | all
# -a, --api      : opengl | vulkan | metal | all
# -o, --output   : 输出文件路径
# -f, --optimize : 优化级别 (0-3)
```

### 运行时加载

```cpp
#include <filament/Material.h>
#include <filament/MaterialInstance.h>

// 加载编译后的材质
Material* loadMaterial(Engine* engine, const uint8_t* data, size_t size) {
    return Material::Builder()
        .package(data, size)
        .build(*engine);
}

// 创建实例并设置参数
MaterialInstance* createInstance(Material* material) {
    MaterialInstance* instance = material->createInstance();
    
    // 设置 uniform 参数
    instance->setParameter("roughness", 0.5f);
    instance->setParameter("baseColor", RgbType::sRGB, {1.0f, 0.0f, 0.0f});
    
    // 设置纹理
    Texture* albedoTexture = loadTexture(...);
    instance->setParameter("albedo", albedoTexture, sampler);
    
    return instance;
}
```

---

*文档整理时间: 2026-03-20*
