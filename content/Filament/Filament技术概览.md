# Filament 技术概览

## 简介

**Filament** 是 Google 开发的开源**实时基于物理的渲染引擎（PBR）**，专为 Android 设计，同时支持 iOS、Linux、macOS、Windows 和 WebGL2。

- **GitHub**: https://github.com/google/filament
- **文档**: https://google.github.io/filament/
- **许可证**: Apache 2.0
- **开发**: Google LLC

### 核心定位

```
┌─────────────────────────────────────────────────────────────────┐
│                    Filament 定位                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │               实时 PBR 渲染引擎                            │  │
│  │                                                            │  │
│  │  • 移动优先设计 (Android/iOS)                               │  │
│  │  • 跨平台支持 (Desktop/Web)                                │  │
│  │  • 轻量级、高性能                                           │  │
│  │  • 现代图形 API (OpenGL/Vulkan/Metal/WebGPU)               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│          ┌───────────────────┼───────────────────┐              │
│          ▼                   ▼                   ▼              │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐      │
│  │  游戏引擎    │     │  3D 应用     │     │  产品可视化  │      │
│  │  集成        │     │  开发        │     │  渲染        │      │
│  └─────────────┘     └─────────────┘     └─────────────┘      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 主要特性

### 1. 渲染特性

| 特性 | 说明 |
|------|------|
| **Clustered Forward Renderer** | 集群前向渲染器，高效处理多光源 |
| **PBR 材质系统** | 基于物理的材质工作流 |
| **IBL** | 基于图像的照明 |
| **HDR/线性光照** | 高动态范围渲染 |
| **多种阴影** | EVSM、PCSS、DPCF、PCF、级联阴影 |
| **屏幕空间效果** | SSAO、SSR、折射、接触阴影 |
| **后处理** | 泛光、景深、TAA、FXAA、MSAA |
| **色彩管理** | 多种色调映射器 (ACES、AgX、Filmic) |

### 2. 材质模型

| 材质 | 用途 |
|------|------|
| **Standard (Lit)** | 标准 PBR 材质 |
| **Clear Coat** | 清漆涂层 (汽车漆等) |
| **Anisotropic** | 各向异性 (拉丝金属等) |
| **Subsurface** | 次表面散射 (皮肤、蜡等) |
| **Cloth/Sheen** | 布料/光泽 (天鹅绒等) |
| **Unlit** | 自发光/无光照 |

### 3. 平台支持

| 平台 | API 支持 |
|------|----------|
| **Android** | OpenGL ES 3.0+, Vulkan 1.0+ |
| **iOS** | OpenGL ES 3.0+, Metal |
| **macOS** | OpenGL 4.1+, Metal, Vulkan |
| **Linux** | OpenGL 4.1+, Vulkan 1.0+ |
| **Windows** | OpenGL 4.1+, Vulkan 1.0+ |
| **Web** | WebGL 2.0, WebGPU |

---

## 架构设计

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Filament 架构                                       │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                              应用层 (Application)                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  C++ API    │  │ Java/JNI    │  │ JavaScript  │  │    glTF I/O         │ │
│  │  (Native)   │  │ (Android)   │  │  (Web)      │  │   (模型加载)         │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              引擎核心 (Engine)                               │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         Engine (渲染引擎)                               │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │  │
│  │  │  Renderer   │  │    View     │  │   Scene     │  │   Camera    │  │  │
│  │  │  (渲染器)   │  │   (视图)    │  │   (场景)    │  │   (相机)    │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     RenderableManager (可渲染对象管理)                  │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │  │
│  │  │ VertexBuffer│  │ IndexBuffer │  │  Material   │  │   Entity    │  │  │
│  │  │  (顶点缓冲)  │  │  (索引缓冲)  │  │   (材质)    │  │   (实体)    │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     LightManager (光源管理)                             │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │  │
│  │  │ Point Light │  │  Spot Light │  │ Directional│                   │  │
│  │  │  (点光源)   │  │  (聚光灯)   │  │   Light    │                   │  │
│  │  └─────────────┘  └─────────────┘  │ (方向光)   │                   │  │
│  │                                    └─────────────┘                   │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     TransformManager (变换管理)                         │  │
│  │  处理实体的位置、旋转、缩放，维护场景图层级                               │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            渲染后端 (Backend)                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   OpenGL    │  │   Vulkan    │  │    Metal    │  │      WebGPU         │ │
│  │   (ES 3.0+) │  │   (1.0+)    │  │   (Apple)   │  │      (Web)          │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            工具链 (Toolchain)                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │    matc     │  │    cmgen    │  │   filamesh  │  │      mipgen         │ │
│  │  (材质编译)  │  │  (环境贴图)  │  │ (网格转换)  │  │    (纹理生成)        │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 核心组件详解

### 1. Engine (引擎)

Engine 是 Filament 的核心，管理所有资源的生命周期。

```cpp
#include <filament/Engine.h>

using namespace filament;

// 创建引擎 (选择后端)
Engine* engine = Engine::create(Backend::VULKAN);  // 或 OPENGL, METAL

// 销毁引擎
delete engine;
// 或
Engine::destroy(&engine);
```

### 2. Renderer (渲染器)

Renderer 负责执行实际的渲染操作。

```cpp
#include <filament/Renderer.h>

// 创建渲染器
Renderer* renderer = engine->createRenderer();

// 渲染一帧
void renderFrame() {
    // 开始帧
    if (renderer->beginFrame(swapChain)) {
        // 渲染视图
        renderer->render(view);
        
        // 结束帧
        renderer->endFrame();
    }
}
```

### 3. View (视图)

View 定义了渲染的视口和相机。

```cpp
#include <filament/View.h>

// 创建视图
View* view = engine->createView();

// 配置视图
view->setCamera(camera);      // 设置相机
view->setScene(scene);        // 设置场景
view->setViewport({0, 0, width, height});  // 设置视口

// 后处理配置
view->setPostProcessingEnabled(true);
view->setAmbientOcclusion(View::AmbientOcclusion::SSAO);
view->setBloomOptions({...});
```

### 4. Scene (场景)

Scene 包含了所有可渲染的实体和光源。

```cpp
#include <filament/Scene.h>

// 创建场景
Scene* scene = engine->createScene();

// 添加实体到场景
scene->addEntity(renderable);
scene->addEntity(light);

// 移除实体
scene->remove(entity);
```

### 5. Entity (实体)

Entity 是场景中的对象标识符，使用实体组件系统（ECS）。

```cpp
#include <utils/EntityManager.h>

using namespace utils;

// 创建实体
Entity entity = EntityManager::get().create();

// 销毁实体
EntityManager::get().destroy(entity);
```

---

## 渲染流程

```
渲染流程图:

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  应用逻辑    │────▶│  更新场景   │────▶│  相机/光源  │
│  (Update)   │     │  (Scene)    │     │  设置       │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                                               ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  交换缓冲区  │◀────│  结束帧     │◀────│  渲染视图   │
│  (Swap)     │     │  EndFrame   │     │  Render     │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                    ┌──────────────────────────┘
                    ▼
           ┌─────────────────┐
           │  渲染管线阶段    │
           │                 │
           │ 1. Shadow Pass  │ (阴影渲染)
           │ 2. Color Pass   │ (主场景渲染)
           │ 3. Post-Process │ (后处理)
           └─────────────────┘
```

**代码示例**:

```cpp
void mainLoop() {
    while (running) {
        // 1. 更新逻辑
        updateScene();
        
        // 2. 开始渲染帧
        if (renderer->beginFrame(swapChain)) {
            // 3. 渲染视图
            renderer->render(view);
            
            // 4. 结束帧
            renderer->endFrame();
        }
    }
}
```

---

## 材质系统

### 材质定义

Filament 使用自定义的材质描述语言，通过 `matc` 编译器编译为二进制。

```mat
// simple.mat
material {
    name : Simple,
    shadingModel : lit,  // 或 unlit, subsurface, cloth, etc.
    
    parameters : [
        { type : float3, name : baseColor },
        { type : float,  name : roughness },
        { type : float,  name : metallic }
    ],
    
    requires : [
        uv0
    ]
}

fragment {
    void material(inout MaterialInputs material) {
        prepareMaterial(material);
        
        material.baseColor.rgb = materialParams.baseColor;
        material.roughness = materialParams.roughness;
        material.metallic = materialParams.metallic;
    }
}
```

### 编译材质

```bash
# 编译材质
matc -p mobile -a opengl -o simple.filamat simple.mat

# 参数说明:
# -p platform    : mobile | desktop | all
# -a api         : opengl | vulkan | metal | all
# -o output      : 输出文件
```

### 使用材质

```cpp
#include <filament/Material.h>
#include <filament/MaterialInstance.h>

// 加载材质
Material* material = Material::Builder()
    .package((void*)BAKED_MATERIAL_PACKAGE, sizeof(BAKED_MATERIAL_PACKAGE))
    .build(*engine);

// 创建材质实例
MaterialInstance* instance = material->createInstance();

// 设置参数
instance->setParameter("baseColor", RgbType::sRGB, {0.8f, 0.2f, 0.2f});
instance->setParameter("roughness", 0.4f);
instance->setParameter("metallic", 0.8f);
```

---

## 几何体渲染

### 创建顶点缓冲

```cpp
#include <filament/VertexBuffer.h>
#include <filament/IndexBuffer.h>

// 顶点数据
struct Vertex {
    float position[3];
    float normal[3];
    float uv[2];
};

// 创建顶点缓冲
VertexBuffer* vb = VertexBuffer::Builder()
    .vertexCount(100)
    .bufferCount(1)
    .attribute(VertexAttribute::POSITION, 0, VertexBuffer::AttributeType::FLOAT3, 0, sizeof(Vertex))
    .attribute(VertexAttribute::NORMAL, 0, VertexBuffer::AttributeType::FLOAT3, 12, sizeof(Vertex))
    .attribute(VertexAttribute::UV0, 0, VertexBuffer::AttributeType::FLOAT2, 24, sizeof(Vertex))
    .build(*engine);

// 设置数据
vb->setBufferAt(*engine, 0, VertexBuffer::BufferDescriptor(vertices, vertexCount * sizeof(Vertex)));

// 创建索引缓冲
IndexBuffer* ib = IndexBuffer::Builder()
    .indexCount(300)
    .bufferType(IndexBuffer::IndexType::USHORT)
    .build(*engine);

ib->setBuffer(*engine, IndexBuffer::BufferDescriptor(indices, indexCount * sizeof(uint16_t)));
```

### 创建可渲染实体

```cpp
#include <filament/RenderableManager.h>

// 创建实体
Entity renderable = EntityManager::get().create();

// 构建可渲染对象
RenderableManager::Builder(1)  // 1个 primitive
    .boundingBox({{-1, -1, -1}, {1, 1, 1}})
    .material(0, materialInstance)
    .geometry(0, RenderableManager::PrimitiveType::TRIANGLES, vb, ib, 0, indexCount)
    .culling(true)
    .castShadows(true)
    .receiveShadows(true)
    .build(*engine, renderable);

// 添加到场景
scene->addEntity(renderable);
```

---

## 光源系统

### 方向光 (Sun)

```cpp
#include <filament/LightManager.h>

Entity sun = EntityManager::get().create();

LightManager::Builder(LightManager::Type::SUN)
    .color(Color::toLinear<sRGB>({0.98f, 0.92f, 0.89f}))
    .intensity(110000)  // 物理单位: lux
    .direction({0.0f, -1.0f, -1.0f})
    .castShadows(true)
    .shadowOptions({
        .mapSize = 2048,
        .cascadeSplitFactor = 0.5f
    })
    .build(*engine, sun);

scene->addEntity(sun);
```

### 点光源

```cpp
Entity pointLight = EntityManager::get().create();

LightManager::Builder(LightManager::Type::POINT)
    .color(Color::toLinear<sRGB>({1.0f, 0.9f, 0.8f}))
    .intensity(1000)  // lumen
    .position({2.0f, 3.0f, 0.0f})
    .falloff(10.0f)   // 衰减半径
    .build(*engine, pointLight);

scene->addEntity(pointLight);
```

### 聚光灯

```cpp
Entity spotLight = EntityManager::get().create();

LightManager::Builder(LightManager::Type::SPOT)
    .color(Color::toLinear<sRGB>({1.0f, 1.0f, 1.0f}))
    .intensity(2000)
    .position({0.0f, 5.0f, 0.0f})
    .direction({0.0f, -1.0f, 0.0f})
    .spotLightCone(30.0f, 45.0f)  // 内角, 外角
    .castShadows(true)
    .build(*engine, spotLight);

scene->addEntity(spotLight);
```

---

## IBL (基于图像的照明)

Filament 支持 HDR 环境贴图照明。

```cpp
#include <filament/IndirectLight.h>
#include <filament/Skybox.h>

// 加载 IBL (使用 cmgen 生成)
Texture* iblTexture = Texture::Builder()
    .width(256)
    .height(256)
    .levels(9)
    .sampler(Texture::Sampler::SAMPLER_CUBEMAP)
    .format(Texture::InternalFormat::R11F_G11F_B10F)
    .build(*engine);

// 加载 IBL 数据...

// 创建间接光
IndirectLight* indirectLight = IndirectLight::Builder()
    .reflections(iblTexture)
    .irradiance(3, iblSH)  // 球谐系数
    .intensity(30000)
    .build(*engine);

scene->setIndirectLight(indirectLight);

// 创建天空盒
Texture* skyboxTexture = ...;

Skybox* skybox = Skybox::Builder()
    .environment(skyboxTexture)
    .build(*engine);

scene->setSkybox(skybox);
```

---

## 工具链

### 1. matc - 材质编译器

```bash
# 编译材质
matc -p mobile -a all -o material.filamat material.mat
```

### 2. cmgen - 环境贴图生成器

```bash
# 生成 IBL
cmgen -f ktx -x ./output/ environment.exr

# 参数:
# -f format   : ktx | png
# -x output   : 输出目录
# --extract-blur=0.1  : 模糊背景
```

### 3. filamesh - 网格转换器

```bash
# 转换模型
filamesh input.obj output.filamesh
```

### 4. gltfio - glTF 加载库

```cpp
#include <gltfio/AssetLoader.h>
#include <gltfio/ResourceLoader.h>

// 创建加载器
AssetLoader* loader = AssetLoader::create({engine});

// 加载 glTF
FilamentAsset* asset = loader->createAsset(buffer, bufferSize);

// 加载资源
ResourceLoader({engine}).loadResources(asset);

// 添加到场景
scene->addEntities(asset->getEntities(), asset->getEntityCount());
```

---

## 平台特定

### Android (Java/Kotlin)

```kotlin
// Gradle 依赖
implementation 'com.google.android.filament:filament-android:1.70.0'

// 初始化
val engine = Engine.create()
val renderer = engine.createRenderer()
val scene = engine.createScene()
val view = engine.createView()
val camera = engine.createCamera(EntityManager.get().create())

view.camera = camera
view.scene = scene
```

### iOS (CocoaPods)

```ruby
# Podfile
pod 'Filament', '~> 1.70.0'
```

### Web (JavaScript)

```javascript
// 加载 Filament.js
import * as Filament from 'filament.js';

// 初始化
const engine = Filament.Engine.create(canvas);
const scene = engine.createScene();
```

---

## 性能优化建议

1. **使用材质实例**: 共享材质，使用实例设置不同参数
2. **合批渲染**: 减少 Draw Call
3. **合理的阴影质量**: 根据设备调整阴影贴图大小
4. **LOD 系统**: 远距离使用简化模型
5. **纹理压缩**: 使用 ETC2/ASTC 压缩纹理
6. **动态分辨率**: 使用 FSR 等技术动态调整分辨率

---

*文档整理时间: 2026-03-20*
*参考: Google Filament GitHub 官方文档*
