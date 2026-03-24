# ozz-animation 示例教程

## 1. 基础示例：播放单个动画

最简单的动画播放示例。

```cpp
#include <ozz/animation/runtime/skeleton.h>
#include <ozz/animation/runtime/animation.h>
#include <ozz/animation/runtime/sampling_job.h>
#include <ozz/animation/runtime/local_to_model_job.h>
#include <ozz/base/io/archive.h>
#include <fstream>

class SimplePlayback {
private:
    // 资源
    ozz::unique_ptr<ozz::animation::Skeleton> skeleton_;
    ozz::unique_ptr<ozz::animation::Animation> animation_;
    
    // 运行时缓冲区
    ozz::vector<ozz::math::SoaTransform> localTransforms_;
    ozz::vector<ozz::math::Float4x4> modelMatrices_;
    
    // 采样上下文
    ozz::unique_ptr<ozz::animation::SamplingJob::Context> context_;
    
    // 播放状态
    float currentTime_ = 0.0f;
    
public:
    bool Load(const char* skeletonPath, const char* animationPath) {
        // 加载骨架
        {
            std::ifstream file(skeletonPath, std::ios::binary);
            ozz::io::IArchive archive(file);
            
            skeleton_ = make_unique<ozz::animation::Skeleton>();
            archive >> *skeleton_;
        }
        
        // 加载动画
        {
            std::ifstream file(animationPath, std::ios::binary);
            ozz::io::IArchive archive(file);
            
            animation_ = make_unique<ozz::animation::Animation>();
            archive >> *animation_;
        }
        
        // 分配缓冲区
        int numSoaJoints = skeleton_>num_soa_joints();
        int numJoints = skeleton_>num_joints();
        
        localTransforms_.resize(numSoaJoints);
        modelMatrices_.resize(numJoints);
        
        // 创建采样上下文
        context_ = make_unique<ozz::animation::SamplingJob::Context>(numJoints);
        
        return true;
    }
    
    void Update(float deltaTime) {
        // 更新时间
        currentTime_ += deltaTime;
        
        // 循环播放
        while (currentTime_ > animation_>duration()) {
            currentTime_ -= animation_>duration();
        }
        
        // 1. 采样动画
        ozz::animation::SamplingJob sampleJob;
        sampleJob.animation = animation_.get();
        sampleJob.context = context_.get();
        sampleJob.ratio = currentTime_ / animation_>duration();
        sampleJob.output = make_span(localTransforms_);
        sampleJob.Run();
        
        // 2. 局部空间转模型空间
        ozz::animation::LocalToModelJob l2mJob;
        l2mJob.skeleton = skeleton_.get();
        l2mJob.input = make_span(localTransforms_);
        l2mJob.output = make_span(modelMatrices_);
        l2mJob.Run();
    }
    
    const ozz::vector<ozz::math::Float4x4>& GetModelMatrices() const {
        return modelMatrices_;
    }
};
```

---

## 2. 动画混合示例

将行走和跑步动画平滑混合。

```cpp
#include <ozz/animation/runtime/blending_job.h>

class AnimationBlending {
private:
    // 动画资源
    ozz::unique_ptr<ozz::animation::Skeleton> skeleton_;
    ozz::unique_ptr<ozz::animation::Animation> walkAnim_;
    ozz::unique_ptr<ozz::animation::Animation> runAnim_;
    
    // 缓冲区
    ozz::vector<ozz::math::SoaTransform> walkTransforms_;
    ozz::vector<ozz::math::SoaTransform> runTransforms_;
    ozz::vector<ozz::math::SoaTransform> blendedTransforms_;
    ozz::vector<ozz::math::Float4x4> modelMatrices_;
    
    // 采样上下文
    ozz::unique_ptr<ozz::animation::SamplingJob::Context> walkContext_;
    ozz::unique_ptr<ozz::animation::SamplingJob::Context> runContext_;
    
    // 时间
    float walkTime_ = 0.0f;
    float runTime_ = 0.0f;
    
public:
    void Update(float deltaTime, float walkWeight) {
        walkTime_ += deltaTime;
        runTime_ += deltaTime * 1.5f;  // 跑步动画稍快
        
        // 规范化时间
        walkTime_ = fmod(walkTime_, walkAnim_>duration());
        runTime_ = fmod(runTime_, runAnim_>duration());
        
        // 采样两个动画
        ozz::animation::SamplingJob walkSample;
        walkSample.animation = walkAnim_.get();
        walkSample.context = walkContext_.get();
        walkSample.ratio = walkTime_ / walkAnim_>duration();
        walkSample.output = make_span(walkTransforms_);
        walkSample.Run();
        
        ozz::animation::SamplingJob runSample;
        runSample.animation = runAnim_.get();
        runSample.context = runContext_.get();
        runSample.ratio = runTime_ / runAnim_>duration();
        runSample.output = make_span(runTransforms_);
        runSample.Run();
        
        // 混合动画
        ozz::animation::BlendingJob::Layer layers[2];
        
        // 行走层
        layers[0].transform = make_span(walkTransforms_);
        layers[0].weight = walkWeight;
        
        // 跑步层
        layers[1].transform = make_span(runTransforms_);
        layers[1].weight = 1.0f - walkWeight;
        
        // 执行混合
        ozz::animation::BlendingJob blendJob;
        blendJob.layers = layers;
        blendJob.rest_pose = skeleton_>joint_rest_poses();
        blendJob.output = make_span(blendedTransforms_);
        blendJob.Run();
        
        // 局部转模型空间
        ozz::animation::LocalToModelJob l2mJob;
        l2mJob.skeleton = skeleton_.get();
        l2mJob.input = make_span(blendedTransforms_);
        l2mJob.output = make_span(modelMatrices_);
        l2mJob.Run();
    }
};
```

---

## 3. 部分混合示例

上半身和下半身播放不同动画。

```cpp
class PartialBlending {
private:
    // 资源
    ozz::unique_ptr<ozz::animation::Skeleton> skeleton_;
    ozz::unique_ptr<ozz::animation::Animation> lowerAnim_;  // 下半身动画 (行走)
    ozz::unique_ptr<ozz::animation::Animation> upperAnim_;  // 上半身动画 (射击)
    
    // 缓冲区
    ozz::vector<ozz::math::SoaTransform> lowerTransforms_;
    ozz::vector<ozz::math::SoaTransform> upperTransforms_;
    ozz::vector<ozz::math::SoaTransform> blendedTransforms_;
    ozz::vector<ozz::math::Float4x4> modelMatrices_;
    
    // 关节权重 (用于部分混合)
    ozz::vector<ozz::animation::BlendingJob::JointWeight> jointWeights_;
    
public:
    bool Initialize() {
        // 假设下半身关节索引是 0-9，上半身是 10-19
        int numSoaJoints = skeleton_>num_soa_joints();
        jointWeights_.resize(numSoaJoints);
        
        // 设置关节权重
        // 注意: SoA 格式下，每个 SoA 变换包含 4 个关节
        for (int i = 0; i < numSoaJoints; ++i) {
            // 这个示例简化处理，实际需要根据骨架结构调整
            float weight = (i < 3) ? 0.0f : 1.0f;  // 下半身不受上半身影响
            jointWeights_[i].weight = weight;
        }
        
        return true;
    }
    
    void Update(float deltaTime) {
        // 采样两个动画...
        // (省略采样代码，同上一个示例)
        
        // 部分混合配置
        ozz::animation::BlendingJob::Layer layers[2];
        
        // 下半身动画 (全身播放)
        layers[0].transform = make_span(lowerTransforms_);
        layers[0].weight = 1.0f;
        
        // 上半身动画 (仅影响上半身关节)
        layers[1].transform = make_span(upperTransforms_);
        layers[1].weight = 1.0f;
        layers[1].joint_weights = make_span(jointWeights_);  // 关键!
        
        // 执行混合
        ozz::animation::BlendingJob blendJob;
        blendJob.layers = layers;
        blendJob.rest_pose = skeleton_>joint_rest_poses();
        blendJob.output = make_span(blendedTransforms_);
        blendJob.Run();
        
        // 局部转模型空间...
    }
};
```

---

## 4. IK 示例：脚部适应地面

使用两骨骼 IK 让脚部适应地形。

```cpp
#include <ozz/animation/runtime/ik_two_bone_job.h>

class FootIK {
private:
    // 假设左腿关节索引
    static constexpr int kLeftThighIndex = 10;
    static constexpr int kLeftCalfIndex = 11;
    static constexpr int kLeftFootIndex = 12;
    
    ozz::unique_ptr<ozz::animation::Skeleton> skeleton_;
    ozz::vector<ozz::math::Float4x4> modelMatrices_;
    
public:
    void ApplyLeftFootIK(const ozz::math::Float3& targetPosition, 
                         float groundHeight) {
        // 获取当前关节矩阵
        const auto& thighMatrix = modelMatrices_[kLeftThighIndex];
        const auto& calfMatrix = modelMatrices_[kLeftCalfIndex];
        const auto& footMatrix = modelMatrices_[kLeftFootIndex];
        
        // 获取骨骼长度 (从绑定姿势计算)
        float thighLength = GetBoneLength(kLeftThighIndex, kLeftCalfIndex);
        float calfLength = GetBoneLength(kLeftCalfIndex, kLeftFootIndex);
        
        // 设置 IK 目标 (保持在地面高度)
        ozz::math::Float3 ikTarget = targetPosition;
        ikTarget.y = groundHeight;
        
        // 创建 IK 任务
        ozz::animation::IKTwoBoneJob ikJob;
        
        ikJob.target = ikTarget;
        ikJob.mid_axis = ozz::math::Float3::y_axis();  // 膝盖弯曲轴
        ikJob.pole_vector = ozz::math::Float3::y_axis();
        
        // 骨骼长度
        ikJob.lengths[0] = thighLength;
        ikJob.lengths[1] = calfLength;
        
        // 输入矩阵
        ikJob.start_joint = &thighMatrix;
        ikJob.mid_joint = &calfMatrix;
        ikJob.end_joint = &footMatrix;
        
        // 输出旋转修正
        ozz::math::Quaternion thighCorrection;
        ozz::math::Quaternion calfCorrection;
        bool reached = false;
        
        ikJob.start_joint_correction = &thighCorrection;
        ikJob.mid_joint_correction = &calfCorrection;
        ikJob.reached = &reached;
        
        // 执行 IK
        if (ikJob.Run()) {
            // 应用旋转修正到局部变换
            // 注意: 这里需要更新 localTransforms_ 然后重新计算 modelMatrices_
            ApplyRotationCorrection(kLeftThighIndex, thighCorrection);
            ApplyRotationCorrection(kLeftCalfIndex, calfCorrection);
            
            // 重新计算模型矩阵
            UpdateModelMatrices();
        }
    }
    
    float GetBoneLength(int parentIdx, int childIdx) {
        const auto& parentPos = modelMatrices_[parentIdx].cols[3];
        const auto& childPos = modelMatrices_[childIdx].cols[3];
        
        ozz::math::Float3 diff;
        diff.x = childPos[0] - parentPos[0];
        diff.y = childPos[1] - parentPos[1];
        diff.z = childPos[2] - parentPos[2];
        
        return std::sqrt(diff.x * diff.x + diff.y * diff.y + diff.z * diff.z);
    }
};
```

---

## 5. 完整游戏角色系统示例

整合所有功能的角色动画系统。

```cpp
class CharacterAnimationSystem {
public:
    enum class AnimationState {
        Idle,
        Walk,
        Run,
        Jump
    };
    
private:
    // 资源
    ozz::unique_ptr<ozz::animation::Skeleton> skeleton_;
    ozz::unique_ptr<ozz::animation::Animation> animations_[4];  // 对应状态
    
    // 运行时缓冲区
    ozz::vector<ozz::math::SoaTransform> sampledTransforms_;
    ozz::vector<ozz::math::SoaTransform> blendedTransforms_;
    ozz::vector<ozz::math::Float4x4> modelMatrices_;
    ozz::vector<ozz::math::Float4x4> skinningMatrices_;
    
    // 采样上下文
    ozz::vector<ozz::unique_ptr<ozz::animation::SamplingJob::Context>> contexts_;
    
    // 动画状态
    AnimationState currentState_ = AnimationState::Idle;
    AnimationState targetState_ = AnimationState::Idle;
    float stateBlendWeight_ = 0.0f;
    float currentTimes_[4] = {0};
    
    // IK 设置
    bool enableFootIK_ = true;
    float groundHeight_ = 0.0f;
    
public:
    void Update(float deltaTime, const ozz::math::Float3& moveInput) {
        // 1. 根据输入确定目标状态
        DetermineTargetState(moveInput);
        
        // 2. 状态过渡
        UpdateStateTransition(deltaTime);
        
        // 3. 采样当前动画
        SampleAnimations(deltaTime);
        
        // 4. 混合动画
        BlendAnimations();
        
        // 5. 应用 IK
        if (enableFootIK_) {
            ApplyFootIK();
        }
        
        // 6. 局部转模型空间
        LocalToModel();
        
        // 7. 计算蒙皮矩阵
        ComputeSkinningMatrices();
    }
    
private:
    void DetermineTargetState(const ozz::math::Float3& moveInput) {
        float speed = std::sqrt(moveInput.x * moveInput.x + moveInput.z * moveInput.z);
        
        if (speed < 0.1f) {
            targetState_ = AnimationState::Idle;
        } else if (speed < 0.5f) {
            targetState_ = AnimationState::Walk;
        } else {
            targetState_ = AnimationState::Run;
        }
    }
    
    void UpdateStateTransition(float deltaTime) {
        const float transitionSpeed = 5.0f;
        
        if (currentState_ != targetState_) {
            stateBlendWeight_ -= deltaTime * transitionSpeed;
            
            if (stateBlendWeight_ <= 0.0f) {
                currentState_ = targetState_;
                stateBlendWeight_ = 0.0f;
            }
        } else {
            stateBlendWeight_ += deltaTime * transitionSpeed;
            if (stateBlendWeight_ >= 1.0f) {
                stateBlendWeight_ = 1.0f;
            }
        }
    }
    
    void SampleAnimations(float deltaTime) {
        int currentIdx = static_cast<int>(currentState_);
        int targetIdx = static_cast<int>(targetState_);
        
        // 更新当前动画时间
        currentTimes_[currentIdx] += deltaTime;
        while (currentTimes_[currentIdx] > animations_[currentIdx]->duration()) {
            currentTimes_[currentIdx] -= animations_[currentIdx]->duration();
        }
        
        // 采样当前动画
        ozz::animation::SamplingJob sampleJob;
        sampleJob.animation = animations_[currentIdx].get();
        sampleJob.context = contexts_[currentIdx].get();
        sampleJob.ratio = currentTimes_[currentIdx] / animations_[currentIdx]->duration();
        sampleJob.output = make_span(sampledTransforms_);
        sampleJob.Run();
        
        // 如果需要过渡，也采样目标动画
        if (currentState_ != targetState_) {
            currentTimes_[targetIdx] += deltaTime;
            while (currentTimes_[targetIdx] > animations_[targetIdx]->duration()) {
                currentTimes_[targetIdx] -= animations_[targetIdx]->duration();
            }
            
            ozz::vector<ozz::math::SoaTransform> targetTransforms(skeleton_>num_soa_joints());
            
            ozz::animation::SamplingJob targetSample;
            targetSample.animation = animations_[targetIdx].get();
            targetSample.context = contexts_[targetIdx].get();
            targetSample.ratio = currentTimes_[targetIdx] / animations_[targetIdx]->duration();
            targetSample.output = make_span(targetTransforms);
            targetSample.Run();
            
            // 存储目标变换供后续混合使用
            targetTransforms_ = std::move(targetTransforms);
        }
    }
    
    ozz::vector<ozz::math::SoaTransform> targetTransforms_;
    
    void BlendAnimations() {
        if (currentState_ != targetState_ || stateBlendWeight_ < 1.0f) {
            // 状态间过渡
            ozz::animation::BlendingJob::Layer layers[2];
            
            layers[0].transform = make_span(sampledTransforms_);
            layers[0].weight = 1.0f - stateBlendWeight_;
            
            layers[1].transform = make_span(targetTransforms_);
            layers[1].weight = stateBlendWeight_;
            
            ozz::animation::BlendingJob blendJob;
            blendJob.layers = layers;
            blendJob.rest_pose = skeleton_>joint_rest_poses();
            blendJob.output = make_span(blendedTransforms_);
            blendJob.Run();
        } else {
            // 单动画，直接复制
            blendedTransforms_ = sampledTransforms_;
        }
    }
    
    void LocalToModel() {
        ozz::animation::LocalToModelJob l2mJob;
        l2mJob.skeleton = skeleton_.get();
        l2mJob.input = make_span(blendedTransforms_);
        l2mJob.output = make_span(modelMatrices_);
        l2mJob.Run();
    }
    
    void ComputeSkinningMatrices() {
        const auto& invBindPoses = skeleton_>inverse_bind_poses();
        
        for (int i = 0; i < skeleton_>num_joints(); ++i) {
            skinningMatrices_[i] = modelMatrices_[i] * invBindPoses[i];
        }
    }
    
    void ApplyFootIK() {
        // 简化的 IK 应用
        // 实际实现需要射线检测地面高度等
    }
};
```

---

## 6. 多线程动画示例

使用多线程处理多个角色的动画。

```cpp
#include <thread>
#include <vector>
#include <future>

class MultithreadedAnimation {
private:
    struct Character {
        ozz::unique_ptr<ozz::animation::Skeleton> skeleton;
        ozz::unique_ptr<ozz::animation::Animation> animation;
        ozz::vector<ozz::math::SoaTransform> localTransforms;
        ozz::vector<ozz::math::Float4x4> modelMatrices;
        ozz::unique_ptr<ozz::animation::SamplingJob::Context> context;
        float time = 0.0f;
    };
    
    ozz::vector<Character> characters_;
    unsigned int numThreads_;
    
public:
    void UpdateParallel(float deltaTime) {
        size_t numCharacters = characters_.size();
        size_t charactersPerThread = (numCharacters + numThreads_ - 1) / numThreads_;
        
        std::vector<std::future<void>> futures;
        
        for (unsigned int t = 0; t < numThreads_; ++t) {
            size_t start = t * charactersPerThread;
            size_t end = std::min(start + charactersPerThread, numCharacters);
            
            if (start >= end) break;
            
            futures.push_back(std::async(std::launch::async, [this, deltaTime, start, end]() {
                for (size_t i = start; i < end; ++i) {
                    UpdateCharacter(characters_[i], deltaTime);
                }
            }));
        }
        
        // 等待所有线程完成
        for (auto& future : futures) {
            future.wait();
        }
    }
    
    void UpdateCharacter(Character& character, float deltaTime) {
        // 更新时间
        character.time += deltaTime;
        while (character.time > character.animation->duration()) {
            character.time -= character.animation->duration();
        }
        
        // 采样
        ozz::animation::SamplingJob sampleJob;
        sampleJob.animation = character.animation.get();
        sampleJob.context = character.context.get();
        sampleJob.ratio = character.time / character.animation->duration();
        sampleJob.output = make_span(character.localTransforms);
        sampleJob.Run();
        
        // 局部转模型
        ozz::animation::LocalToModelJob l2mJob;
        l2mJob.skeleton = character.skeleton.get();
        l2mJob.input = make_span(character.localTransforms);
        l2mJob.output = make_span(character.modelMatrices);
        l2mJob.Run();
    }
};
```

---

## 编译和运行

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.22)
project(OzzExample)

set(CMAKE_CXX_STANDARD 17)

# 查找 ozz
find_package(ozz-animation REQUIRED)

# 可执行文件
add_executable(ozz_example main.cpp)

# 链接库
target_link_libraries(ozz_example
    ozz_animation
    ozz_base
    ozz_geometry
)
```

### 编译

```bash
mkdir build
cd build
cmake ..
make -j
```

### 运行

```bash
./ozz_example skeleton.ozz animation.ozz
```

---

*文档创建时间: 2026-03-20*
