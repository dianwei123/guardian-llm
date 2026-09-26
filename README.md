#guardian-llm

Android 端本地大模型推理应用：llama.cpp 运行时 + Vulkan GPU 加速 + Hexagon NPU 后端 + 温度/内存熔断保护。

> **作者：dianwei114514，欢迎转载**
>
> ⚠️ 测试版本，问题不受理（issue 可能不回复，欢迎自行 fork 修改）

## 功能

- **GGUF 识别**：Kotlin 与 C++ 双解析器，支持 v2/v3、全部 13 种元数据类型、张量表、对齐、分片与截断检测
- **本地推理**：llama.cpp 静态链接进 APK，支持多轮对话模板、流式输出
- **GPU 加速**：Vulkan 后端，层数可调（0–99），实测骁龙 8 Gen 3 上 `offloaded 33/33 layers to GPU`
- **NPU 后端**：llama.cpp 的 Hexagon 后端（HTP skel v73/v75/v79/v81），受设备 ROM 策略限制，详见下方限制
- **守护层**：CPU 温度与内存采样，三级熔断（降速 → 暂停保留 KV Cache → 释放上下文），支持打断推理
- **模型管理**：在线下载（国内源优先，失败自动换源）、本地导入、删除、切换
- **可调参数**：GPU 层数、上下文上限、温度/内存阈值

## 系统要求

| 项目 | 要求 |
|---|---|
| 系统版本 | Android 9（API 28）及以上 |
| 架构 | arm64-v8a |
| GPU 加速 | 需要 Vulkan 1.1+（Android 9 起基本满足） |
| NPU 加速 | 需要 ROM 允许应用加载 `libcdsprpc.so`（部分国产 ROM 拦截） |

## 下载

见本仓库的 [Releases](../../releases) 页面。`release` 为日常使用版，`debug` 额外包含模型下载、设备探测、后端日志与诊断报告。

## 从源码构建

```bash
# 1. 依赖：JDK 17-21、Android SDK 35、Build-Tools 35、NDK 27.2、CMake 3.22.1
export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/27.2.12479018
export JAVA_HOME=/path/to/jdk-21

# 2. llama.cpp 运行时（静态库，供 APK 链接）
git clone --depth 1 https://github.com/ggml-org/llama.cpp ../llama.cpp

# 纯 CPU + Vulkan GPU 后端：
git clone --depth 1 https://github.com/KhronosGroup/Vulkan-Headers ../Vulkan-Headers
git clone --depth 1 https://github.com/KhronosGroup/SPIRV-Headers ../SPIRV-Headers
GGML_VULKAN=ON ./tools/build-llamacpp.sh          # 产出 ../llama-build-arm64-vk

# 3. 构建 APK
echo "sdk.dir=/path/to/Android/Sdk" > local.properties
./gradlew :app:testDebugUnitTest :app:assembleRelease
```

Hexagon NPU 后端需要高通 Hexagon SDK（官方渠道需注册账号，**不可随仓库分发**），
参见 `tools/build-llamacpp.sh` 里的说明与 `work/hexagon-notes/`（若随仓库提供）。

## 项目结构

```
app/src/main/java/com/guardian/lm/
  gguf/         GGUF 解析（元数据、张量表、量化类型、KV Cache 估算）
  inference/    推理引擎、加速器调度、显存/层数规划
  guardian/     温度与内存采样、熔断状态机、前台守护服务
  model/        模型仓库、下载器、内置示例
  nativebridge/ JNI 接口
  ui/           主界面、模型界面、设置
app/src/main/cpp/
  llama_runtime.cpp   llama.cpp 调用、日志捕获、FastRPC 预加载
  accel_probe.cpp     NNAPI 设备枚举、厂商库探测、算子基准
  vulkan_probe.cpp    应用内 Vulkan 实例与物理设备枚举
  gguf_probe.cpp      原生 GGUF 头部解析
  sim_generate.cpp    流式输出与中止链路验证
```

## 已知限制

- **NPU 在部分机型不可用**：应用侧访问 Hexagon 需要 `libcdsprpc.so`，而 Linux 命名空间策略可能拒绝（实测荣耀机型）；NNAPI 也可能只暴露 `nnapi-reference`(CPU)。此时可用加速路径为 GPU(Vulkan) 与 CPU
- **NPU 量化限制**：HTP 仅支持 Q4_0 / Q8_0 / MXFP4 / F32
- **大模型内存压力**：5.9B Q4_K 权重约 5.5GB，加上 KV Cache 与计算缓冲区可能触发系统 LMK（后台应用会被清理）。可调低 GPU 层数或上下文上限
- **对话记录（多轮持久化）尚未实现**

## 致谢与许可

- [llama.cpp / ggml](https://github.com/ggml-org/llama.cpp) — MIT
- [Vulkan-Headers](https://github.com/KhronosGroup/Vulkan-Headers) — Apache-2.0
- [SPIRV-Headers](https://github.com/KhronosGroup/SPIRV-Headers) — MIT

本仓库**不包含**高通 Hexagon SDK 的任何文件（其许可禁止再分发），也不包含模型文件。
模型下载功能指向的第三方模型遵循各自的许可协议。
软件由ai辅助制作
