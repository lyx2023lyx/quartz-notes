# ozz-animation 核心 API 详解

## 核心数据类型

### 1. 骨架 (Skeleton)

骨架是骨骼动画的基础，ozz 使用**深度优先（Depth-First）**存储关节层级。

```cpp
#include <ozz/animation/runtime/skeleton.h>

// 骨架结构
class Skeleton {
public:
    // 关节数量
    int num_joints() const;
    
    // 关节层级数据
    span<const int16_t> joint_parents() const;  // 父关节索引
    span<const char> joint_names() const;        // 关节名称
    
    // 绑定姿势
    span<const math::Float4x4> inverse_bind_poses() const;
    
    // 遍历关节
    template <typename _Fn>
    void IterateJointsDF(_Fn&& _callback) const;  // 深度优先
    
    static constexpr int kMaxJoints = 1024;  // 最大关节数
};
```

**内存布局**:
```
Skeleton 内存布局:

┌────────────────────────────────────────────────────────┐
│  Joint 0 (Root)                                        │
│    ├─ Joint 1                                          │
│    │    └─ Joint 2                                     │
│    ├─ Joint 3                                          │
│    │    ├─ Joint 4                                     │
│    │    └─ Joint 5                                     │
│    └─ Joint 6                                          │
│                                                          │
│  存储顺序: 0, 1, 2, 3, 4, 5, 6 (深度优先)               │
└────────────────────────────────────────────────────────┘

关节数据数组:
┌─────────┬───────────┬───────────────┬──────────────────┐
│ Index   │ Parent    │ Name          │ Inv Bind Pose    │
├─────────┼───────────┼───────────────┼──────────────────┤
│ 0       │ -1        │ "Root"        │ Matrix[0]        │
│ 1       │ 0         │ "Spine"       │ Matrix[1]        │
│ 2       │ 1         │ "Chest"       │ Matrix[2]        │
│ 3       │ 0         │ "L_Thigh"     │ Matrix[3]        │
│ ...     │ ...       │ ...           │ ...              │
└─────────┴───────────┴───────────────┴──────────────────┘
```

---

### 2. 动画 (Animation)

运行动画是压缩后的动画数据，包含所有关节的动画轨道。

```cpp
#include <ozz/animation/runtime/animation.h>

class Animation {
public:
    // 动画时长 (秒)
    float duration() const;
    
    // 动画名称
    const char* name() const;
    
    // 关键帧数量
    int num_tracks() const;  // 通常等于关节数量
    
    // 每关节3个轨道 (平移、旋转、缩放)
    // 数据压缩存储，不直接暴露
};
```

**动画数据结构**:
```
Animation 内部结构:

┌────────────────────────────────────────────────────────────┐
│ Animation Header                                           │
│   - duration: 2.5 seconds                                  │
│   - name: "walk"                                           │
│   - num_tracks: 60                                         │
└────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌────────────────────────────────────────────────────────────┐
│ Keyframes (压缩存储)                                        │
│                                                            │
│  时间戳数组 (共享)                                          │
│  ┌─────────┬─────────┬─────────┬─────────┐                │
│  │ 0.00s   │ 0.17s   │ 0.33s   │ 0.50s   │ ...            │
│  └─────────┴─────────┴─────────┴─────────┘                │
│                                                            │
│  关节0 平移轨道: [x0,y0,z0] [x1,y1,z1] [x2,y2,z2] ...     │
│  关节0 旋转轨道: [qx0,qy0...] [qx1,qy1...] ...            │
│  关节0 缩放轨道: [sx0,sy0,sz0] ...                        │
│  关节1 平移轨道: ...                                       │
│  ...                                                       │
└────────────────────────────────────────────────────────────┘
```

---

### 3. 变换 (Transform)

ozz 使用 SoA (Structure of Arrays) 格式存储变换，便于 SIMD 优化。

```cpp
#include <ozz/base/maths/transform.h>
#include <ozz/base/maths/soa_transform.h>

// AoS (Array of Structures) - 单个变换
struct Transform {
    Float3 translation;      // 平移
    Quaternion rotation;     // 旋转 (四元数)
    Float3 scale;           // 缩放 (通常统一)
};

// SoA (Structure of Arrays) - 批量变换，SIMD友好
struct SoATransform {
    SoaFloat3 translation;   // 平移数组
    SoaQuaternion rotation;  // 旋转数组
    SoaFloat3 scale;        // 缩放数组
};
```

---

## 核心任务 (Jobs)

ozz 采用**任务（Job）**模式组织计算，所有动画操作都是独立的 Job。

### 1. 采样任务 (SamplingJob)

从动画中采样特定时间的关节变换。

```cpp
#include <ozz/animation/runtime/sampling_job.h>

// 使用方法
void SampleAnimation(
    const Animation& animation,
    const Skeleton& skeleton,
    float time,
    vector<SoaTransform>& output  // 输出: 局部空间变换
) {
    // 创建采样上下文 (缓存优化)
    SamplingJob::Context context(skeleton.num_joints());
    
    // 配置采样任务
    SamplingJob job;
    job.animation = &animation;
    job.context = &context;
    job.ratio = time / animation.duration();  // 归一化到 [0, 1]
    job.output = make_span(output);
    
    // 执行任务
    if (!job.Run()) {
        // 处理错误
    }
}
```

**采样流程**:
```
SamplingJob 执行流程:

Input:                                       Output:
┌─────────────┐                              ┌─────────────────┐
│ Animation   │                              │ SoaTransform[]  │
│  + time     │──▶ SamplingJob::Run() ──────▶│ (局部空间变换)   │
└─────────────┘                              └─────────────────┘

内部步骤:
1. 确定当前时间所在的关键帧区间
2. 对每个关节的每个轨道进行插值
   - 平移: 线性插值 (LERP)
   - 旋转: 四元数球面插值 (SLERP)
   - 缩放: 线性插值 (LERP)
3. 输出 SoA 格式的变换数组
```

**采样上下文优化**:
```cpp
// 采样上下文缓存关键帧状态，加速连续采样
SamplingJob::Context context(skeleton.num_joints());

// 第1帧采样 (从头计算)
job.ratio = 0.0f;
job.Run();  // 较慢

// 第2帧采样 (利用缓存，跳过已处理的关键帧)
job.ratio = 0.016f;  // 约1帧后
job.Run();  // 更快

// 注意: 上下文不是线程安全的，每个线程需要独立的上下文
```

---

### 2. 混合任务 (BlendingJob)

混合多个动画的采样结果。

```cpp
#include <ozz/animation/runtime/blending_job.h>

void BlendAnimations(
    const Skeleton& skeleton,
    const vector<SoaTransform>& anim1,  // 动画1采样结果
    const vector<SoaTransform>& anim2,  // 动画2采样结果
    float weight1,
    float weight2,
    vector<SoaTransform>& output
) {
    BlendingJob job;
    
    // 混合层配置
    BlendingJob::Layer layers[2];
    
    // 第1层
    layers[0].transform = make_span(anim1);
    layers[0].weight = weight1;
    
    // 第2层
    layers[1].transform = make_span(anim2);
    layers[1].weight = weight2;
    
    // 配置任务
    job.layers = layers;
    job.rest_pose = skeleton.joint_rest_poses();  // 绑定姿势作为默认
    job.output = make_span(output);
    
    job.Run();
}
```

**部分混合（关节遮罩）**:
```cpp
void PartialBlend(
    const Skeleton& skeleton,
    const vector<SoaTransform>& upperBodyAnim,
    const vector<SoaTransform>& lowerBodyAnim,
    vector<SoaTransform>& output
) {
    // 创建关节权重数组
    vector<BlendingJob::JointWeight> jointWeights(skeleton.num_soa_joints());
    
    // 设置上半身权重 (Joint 10-30 是上半身)
    for (int i = 10; i < 30; ++i) {
        jointWeights[i].weight = 1.0f;
    }
    
    BlendingJob::Layer layers[2];
    
    // 下半身动画 (全权重)
    layers[0].transform = make_span(lowerBodyAnim);
    layers[0].weight = 1.0f;
    
    // 上半身动画 (关节遮罩)
    layers[1].transform = make_span(upperBodyAnim);
    layers[1].weight = 1.0f;
    layers[1].joint_weights = make_span(jointWeights);
    
    BlendingJob job;
    job.layers = layers;
    job.rest_pose = skeleton.joint_rest_poses();
    job.output = make_span(output);
    
    job.Run();
}
```

---

### 3. 局部转模型任务 (LocalToModelJob)

将关节的局部空间变换转换为模型空间变换。

```cpp
#include <ozz/animation/runtime/local_to_model_job.h>

void ComputeModelMatrices(
    const Skeleton& skeleton,
    const vector<SoaTransform>& localTransforms,
    vector<Float4x4>& modelMatrices
) {
    LocalToModelJob job;
    
    job.skeleton = &skeleton;
    job.input = make_span(localTransforms);
    job.output = make_span(modelMatrices);
    
    // 可选: 只更新部分骨骼 (IK优化)
    // job.from = 5;  // 从第5个关节开始更新
    
    job.Run();
}
```

**转换过程**:
```
LocalToModel 转换:

输入 (局部空间):                    输出 (模型空间):
                                   
Joint 0: T0, R0, S0                Joint 0: M0 = T0 * R0 * S0
  │                                  │
Joint 1: T1, R1, S1                Joint 1: M1 = M0 * T1 * R1 * S1
  │                                  │
Joint 2: T2, R2, S2                Joint 2: M2 = M1 * T2 * R2 * S2

计算: ModelMatrix = ParentModelMatrix * LocalMatrix
```

---

### 4. IK 任务

#### 两骨骼 IK (IKTwoBoneJob)

用于手臂和腿部的 IK 求解。

```cpp
#include <ozz/animation/runtime/ik_two_bone_job.h>

void SolveLegIK(
    const Float4x4& rootMatrix,      // 根关节模型矩阵
    const Float4x4& upperMatrix,    // 大腿模型矩阵
    const Float4x4& lowerMatrix,    // 小腿模型矩阵
    const Float3& footTarget,       // 脚部目标位置
    const Float3& bendHint,         // 膝盖弯曲方向
    float upperLength,               // 大腿长度
    float lowerLength,               // 小腿长度
    Quaternion& upperRotation,       // 输出: 大腿旋转修正
    Quaternion& lowerRotation        // 输出: 小腿旋转修正
) {
    IKTwoBoneJob job;
    
    job.target = footTarget;
    job.mid_axis = bendHint;  // 膝盖弯曲方向提示
    job.pole_vector = bendHint;
    
    // 骨骼长度
    job.lengths[0] = upperLength;
    job.lengths[1] = lowerLength;
    
    // 输入矩阵
    job.start_joint = &rootMatrix;
    job.mid_joint = &upperMatrix;
    job.end_joint = &lowerMatrix;
    
    // 输出
    job.start_joint_correction = &upperRotation;
    job.mid_joint_correction = &lowerRotation;
    job.reached = nullptr;  // 可选: 输出是否可达
    
    job.Run();
}
```

#### 瞄准 IK (IKAimJob)

用于头部、武器等需要瞄准目标的场景。

```cpp
#include <ozz/animation/runtime/ik_aim_job.h>

void SolveHeadLookAt(
    const Float4x4& headMatrix,
    const Float3& targetPosition,
    const Float3& forwardVector,  // 头部朝向向量
    Quaternion& headRotation
) {
    IKAimJob job;
    
    job.target = targetPosition;
    job.forward = forwardVector;
    job.joint = &headMatrix;
    job.joint_correction = &headRotation;
    
    // 可选: 限制旋转角度
    // job.pole_vector = upVector;
    // job.twist_angle = 0.0f;
    
    job.Run();
}
```

---

## 蒙皮 (Skinning)

ozz 提供 CPU 蒙皮任务，将蒙皮矩阵应用到顶点。

```cpp
#include <ozz/geometry/runtime/skinning_job.h>

struct Mesh {
    vector<Float3> positions;        // 顶点位置
    vector<Float3> normals;          // 法线
    vector<Float3> tangents;         // 切线
    vector<uint8_t> joint_indices;   // 影响关节索引
    vector<float> joint_weights;     // 权重
    int num_joints_per_vertex;       // 每顶点影响关节数
};

void SkinMesh(
    const Mesh& mesh,
    const vector<Float4x4&gt;& skinningMatrices,
    vector<Float3>& outputPositions,
    vector<Float3>& outputNormals
) {
    SkinningJob job;
    
    // 输入顶点数据
    job.vertex_count = mesh.positions.size();
    job.in_positions = make_span(mesh.positions);
    job.in_normals = make_span(mesh.normals);
    job.in_tangents = nullptr;  // 可选
    
    // 蒙皮权重
    job.joint_indices = make_span(mesh.joint_indices);
    job.joint_indices_stride = sizeof(uint8_t) * mesh.num_joints_per_vertex;
    job.joint_weights = make_span(mesh.joint_weights);
    job.joint_weights_stride = sizeof(float) * mesh.num_joints_per_vertex;
    job.influences_count = mesh.num_joints_per_vertex;
    
    // 蒙皮矩阵
    job.joint_matrices = make_span(skinningMatrices);
    
    // 输出
    job.out_positions = make_span(outputPositions);
    job.out_normals = make_span(outputNormals);
    job.out_tangents = nullptr;
    
    // 可选: 从模型空间变换到世界空间
    // job.joint_matrixPalette = ...;
    
    job.Run();
}
```

---

## 完整动画更新流程

```cpp
class AnimationSystem {
private:
    const Skeleton* skeleton_;
    const Animation* walkAnim_;
    const Animation* runAnim_;
    
    // 运行时缓冲区
    vector<SoaTransform> walkTransforms_;
    vector<SoaTransform> runTransforms_;
    vector<SoaTransform> blendedTransforms_;
    vector<Float4x4> modelMatrices_;
    vector<Float4x4> skinningMatrices_;
    
    // 采样上下文
    unique_ptr<SamplingJob::Context> walkContext_;
    unique_ptr<SamplingJob::Context> runContext_;
    
public:
    void Init(const Skeleton* skeleton, 
              const Animation* walk, 
              const Animation* run) {
        skeleton_ = skeleton;
        walkAnim_ = walk;
        runAnim_ = run;
        
        int soaCount = skeleton->num_soa_joints();
        walkTransforms_.resize(soaCount);
        runTransforms_.resize(soaCount);
        blendedTransforms_.resize(soaCount);
        modelMatrices_.resize(skeleton->num_joints());
        skinningMatrices_.resize(skeleton->num_joints());
        
        walkContext_ = make_unique<SamplingJob::Context>(skeleton->num_joints());
        runContext_ = make_unique<SamplingJob::Context>(skeleton->num_joints());
    }
    
    void Update(float deltaTime, float walkWeight, float runWeight) {
        static float walkTime = 0.0f;
        static float runTime = 0.0f;
        
        walkTime += deltaTime;
        runTime += deltaTime;
        
        // 1. 采样行走动画
        SamplingJob walkSample;
        walkSample.animation = walkAnim_;
        walkSample.context = walkContext_.get();
        walkSample.ratio = fmod(walkTime, walkAnim_->duration()) / walkAnim_->duration();
        walkSample.output = make_span(walkTransforms_);
        walkSample.Run();
        
        // 2. 采样跑步动画
        SamplingJob runSample;
        runSample.animation = runAnim_;
        runSample.context = runContext_.get();
        runSample.ratio = fmod(runTime, runAnim_->duration()) / runAnim_->duration();
        runSample.output = make_span(runTransforms_);
        runSample.Run();
        
        // 3. 混合动画
        BlendingJob::Layer layers[2];
        layers[0].transform = make_span(walkTransforms_);
        layers[0].weight = walkWeight;
        layers[1].transform = make_span(runTransforms_);
        layers[1].weight = runWeight;
        
        BlendingJob blend;
        blend.layers = layers;
        blend.rest_pose = skeleton_>joint_rest_poses();
        blend.output = make_span(blendedTransforms_);
        blend.Run();
        
        // 4. 局部转模型空间
        LocalToModelJob l2m;
        l2m.skeleton = skeleton_;
        l2m.input = make_span(blendedTransforms_);
        l2m.output = make_span(modelMatrices_);
        l2m.Run();
        
        // 5. 计算蒙皮矩阵
        const auto& invBindPoses = skeleton_->inverse_bind_poses();
        for (int i = 0; i < skeleton_>num_joints(); ++i) {
            skinningMatrices_[i] = modelMatrices_[i] * invBindPoses[i];
        }
    }
    
    const vector<Float4x4>& GetSkinningMatrices() const {
        return skinningMatrices_;
    }
};
```

---

## 内存管理

ozz 使用自定义内存分配器，支持重定向到引擎的内存系统。

```cpp
#include <ozz/base/memory/allocator.h>

// 默认分配器
class Allocator {
public:
    virtual void* Allocate(size_t _size, size_t _alignment) = 0;
    virtual void Deallocate(void* _block) = 0;
};

// 自定义分配器示例
class MyAllocator : public ozz::Allocator {
public:
    void* Allocate(size_t size, size_t alignment) override {
        return myEngineAlloc(size, alignment);
    }
    
    void Deallocate(void* block) override {
        myEngineFree(block);
    }
};

// 设置全局分配器
MyAllocator myAlloc;
ozz::memory::SetAllocator(&myAlloc);
```

---

## 序列化

ozz 使用二进制存档格式存储和加载资源。

```cpp
#include <ozz/base/io/archive.h>

// 保存骨架
void SaveSkeleton(const Skeleton& skeleton, const char* filename) {
    ofstream file(filename, ios::binary);
    ozz::io::OArchive archive(file);
    archive << skeleton;
}

// 加载骨架
unique_ptr<Skeleton> LoadSkeleton(const char* filename) {
    ifstream file(filename, ios::binary);
    ozz::io::IArchive archive(file);
    
    auto skeleton = make_unique<Skeleton>();
    archive >> *skeleton;
    
    return skeleton;
}

// 动画加载
unique_ptr<Animation> LoadAnimation(const char* filename) {
    ifstream file(filename, ios::binary);
    ozz::io::IArchive archive(file);
    
    auto animation = make_unique<Animation>();
    archive >> *animation;
    
    return animation;
}
```

---

## 错误处理

ozz 使用返回值表示错误，不抛出异常。

```cpp
// 检查任务是否成功
if (!job.Run()) {
    // 处理错误
    // 常见错误: 缓冲区大小不匹配、无效输入等
}

// 验证骨架加载
if (!skeleton->Validate()) {
    // 骨架数据损坏
}
```

---

*文档创建时间: 2026-03-20*
*参考: ozz-animation 源码和文档*
