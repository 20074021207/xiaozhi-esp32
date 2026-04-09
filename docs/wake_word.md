# 唤醒词定制指南

本文档介绍 xiaozhi-esp32 项目中唤醒词的工作原理与定制方法。

## 唤醒词系统架构

xiaozhi-esp32 的唤醒词检测运行在独立的 FreeRTOS 任务中，通过 `WakeWord` 抽象接口统一管理：
AudioService
└── WakeWord（抽象基类）
├── EspWakeWord      — Wakenet 单唤醒词检测
├── AfeWakeWord      — ESP-AFE 语音前端（Wakenet + AEC + VAD）
└── CustomWakeWord  — Multinet 多命令词识别（支持自定义唤醒词）


当检测到唤醒词后，触发 `wake_word_detected_callback_`，进而唤醒应用进入语音交互流程（参见 [状态转换图](./learning-guide.md#状态转换图)）。

## 三种唤醒词方案对比

| 维度 | EspWakeWord | AfeWakeWord | CustomWakeWord |
|------|-------------|-------------|----------------|
| 底层引擎 | Wakenet | ESP-AFE | Multinet |
| AEC 回声消除 | ❌ 不支持 | ✅ 支持 | ❌ 不支持 |
| 自定义唤醒词 | ❌ 受限（只能选内置模型） | ❌ 受限（只能选内置模型） | ✅ 完全可自定义 |
| 灵敏度调节 | ❌ 固定（DET_MODE_95） | ❌ 固定 | ✅ 阈值 1-99 可配置 |
| 适用芯片 | ESP32 C3/C5/C6 + PSRAM ESP32 | **仅 ESP32-S3/P4 + PSRAM** | **仅 ESP32-S3/P4 + PSRAM** |

### 1. EspWakeWord — 轻量基础方案

直接使用 Wakenet 单模型检测唤醒词，底层调用 `esp_wn_iface_t` 接口：

```cpp
// main/audio/wake_words/esp_wake_word.cc
wakenet_iface_->create(model_name, DET_MODE_95);  // 固定 95% 置信度门限
```

适用于 ESP32 C3/C5/C6 等资源有限的芯片。

### 2. AfeWakeWord — 完整语音前端

集成 Wakenet + 回声消除（AEC）+ 语音活动检测（VAD），共享 AFE 音频前端管道：

```cpp
// main/audio/wake_words/afe_wake_word.cc
afe_config->aec_init = codec_->input_reference();   // 启用 AEC
afe_config->aec_mode = AEC_MODE_SR_HIGH_PERF;
```

AEC 利用 codec 的 reference 信号（扬声器输出的参考）消除设备自身播放的声音，适合语音助手场景。

### 3. CustomWakeWord — 自定义唤醒词

基于 Multinet 引擎，支持用户自定义拼音唤醒词，无需重新训练模型：

```cpp
// main/audio/wake_words/custom_wake_word.cc
multinet_->create(mn_name_, duration_);
// 置信度 > 阈值 时触发回调
```

## Kconfig 配置

通过 `idf.py menuconfig` 选择唤醒词方案：
ESP Speech Recognition
→ Wake Word Implementation Type
→ Disabled                    # 禁用唤醒词
→ Wakenet model without AFE  # EspWakeWord
→ Wakenet model with AFE     # AfeWakeWord（需 ESP32-S3/P4 + PSRAM）
→ Multinet model             # CustomWakeWord（需 ESP32-S3/P4 + PSRAM）


### 内置唤醒词模型（Wakenet）

对于 EspWakeWord 和 AfeWakeWord，可通过以下 Kconfig 选择内置的预训练唤醒词模型：

| Kconfig 符号 | 唤醒词 | 说明 |
|-------------|--------|------|
| `CONFIG_SR_WN_WN9_NIHAOXIAOZHI_TTS` | 你好小智 | WakeNet9 TTS 优化版（推荐） |
| `CONFIG_SR_WN_WN9S_NIHAOXIAOZHI` | 你好小智 | WakeNet9 短版本（C3/C5/C6） |
| `CONFIG_SR_WN_WN9_ALEXA` | Alexa | 英文唤醒词 |
| `CONFIG_SR_WN_WN9_HIESP` | Hi ESP | 英文唤醒词 |
| `CONFIG_SR_WN_WN9_HIWFIVE` | Hi Five | 英文唤醒词 |
| ... | ... | 更多内置模型见 menuconfig |

这些模型来自 **乐鑫官方 ESP-SR 框架**的预训练模型库，xiaozhi-esp32 项目仅通过 Kconfig 选择加载哪个模型。

### 自定义唤醒词配置（CustomWakeWord）

启用 CustomWakeWord 后，可配置以下 Kconfig 项：

| Kconfig 符号 | 默认值 | 说明 |
|-------------|--------|------|
| `CONFIG_USE_CUSTOM_WAKE_WORD` | N | 启用 Multinet 自定义唤醒词 |
| `CONFIG_CUSTOM_WAKE_WORD` | "xiao tu dou" | 拼音唤醒词（空格分隔） |
| `CONFIG_CUSTOM_WAKE_WORD_DISPLAY` | "小土豆" | 唤醒后显示的文字 |
| `CONFIG_CUSTOM_WAKE_WORD_THRESHOLD` | 20 | 灵敏度 1-99，越小越灵敏 |
| `CONFIG_SEND_WAKE_WORD_DATA` | y | 是否发送唤醒词音频数据到服务器 |

示例配置：
CONFIG_USE_CUSTOM_WAKE_WORD=y
CONFIG_CUSTOM_WAKE_WORD="xiao zhi ni hao"
CONFIG_CUSTOM_WAKE_WORD_DISPLAY="小智你好"
CONFIG_CUSTOM_WAKE_WORD_THRESHOLD=20


**注意**：自定义唤醒词使用拼音格式，中文词之间用空格分隔，声调不敏感。

### 灵敏度阈值说明

CustomWakeWord 的 `CONFIG_CUSTOM_WAKE_WORD_THRESHOLD` 控制检测灵敏度：

| 阈值 | 效果 |
|------|------|
| 1-10 | 极灵敏，可能误触发 |
| 10-30 | 较灵敏，正常推荐范围 |
| 30-50 | 较保守，减少误触发 |
| 50-99 | 极保守，可能漏检 |

## 唤醒词检测流程
用户说"你好小智"
↓
麦克风采集音频（16kHz PCM）
↓
WakeWord 任务实时检测（Feed 数据流）
↓
检测到唤醒词（置信度 > 阈值）
↓
触发 wake_word_detected_callback_
↓
AudioService 编码唤醒词音频（Opus）
↓
Application 收到回调，进入 Connecting/Listening 状态


对于 AfeWakeWord，唤醒词检测还可以在 Speaking 状态继续运行（需开启 `CONFIG_WAKE_WORD_DETECTION_IN_LISTENING`），实现打断（barge-in）功能。

## 固件打包与校验

构建固件资源时，[scripts/build_default_assets.py](../scripts/build_default_assets.py) 会校验唤醒词配置：

```python
# 如果启用自定义唤醒词但未选中 Multinet 模型，报错
if 'USE_CUSTOM_WAKE_WORD=y' in config and not multinet_selected:
    print("Error: USE_CUSTOM_WAKE_WORD is enabled but no multinet models are selected")
    sys.exit(1)
```

## 总结

| 需求 | 推荐方案 |
|------|---------|
| 使用内置"你好小智" | EspWakeWord 或 AfeWakeWord |
| 自定义任意唤醒词 | CustomWakeWord |
| 需要 AEC 回声消除 | AfeWakeWord |
| ESP32 C3/C5/C6 芯片 | EspWakeWord（唯一选择） |
| 自定义 + AEC 兼得 | ❌ 目前不支持 |

## 相关文档

- [learning-guide.md](./learning-guide.md) — 项目架构与状态机
- [custom-board.md](./custom-board.md) — 自定义开发板指南
- 乐鑫 ESP-SR 框架：https://github.com/espressif/esp-sr

## 其他参考内容
- [ESP-SR 官方文档](https://docs.espressif.com/projects/esp-sr/zh_CN/latest/esp32/speech_command_recognition/README.html)
- [Multinet 引擎](https://github.com/espressif/multinet)
- [CustomWakeWord 示例](https://github.com/36dian5hao/esp-sr-multinet)
