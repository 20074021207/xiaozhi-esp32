# xiaozhi-esp32 项目架构与技术内幕

> 给后端/全栈开发者的嵌入式 AI 语音助手入门指南

---

## 第 0 章：写在前面 —— 给后端开发者的阅读指南

### 你已经会的 vs 你即将学的

如果你熟悉 HTTP、微服务、事件循环、依赖注入这些后端概念，那么理解本项目并不困难。
嵌入式开发的很多思想和后端是相通的，只是术语不同。

### 概念映射表

| 后端/全栈概念 | 本项目对应 | 说明 |
|---|---|---|
| Web 服务器 | ESP32 设备（客户端） | ESP32 是连接云端的客户端，不是服务器 |
| 数据库/ORM | NVS（Non-Volatile Storage） | Flash 上的 key-value 持久化存储 |
| 事件总线/消息队列 | FreeRTOS EventGroup | `xEventGroupWaitBits` 等价于事件循环 |
| 依赖注入 | Board 单例 + 虚方法重写 | 通过继承 Board 基类注入不同硬件实现 |
| 中间件管线 | AudioProcessor 链 | AEC -> 降噪 -> VAD，类似 Express 中间件 |
| 插件系统 | MCP Tools | `self.XXX` 工具注册，类似 VSCode 插件 |
| 线程池 | FreeRTOS Tasks | 多个优先级不同的任务并发运行 |
| Docker 镜像 | Firmware 固件 | 编译产物，烧录到 Flash 分区 |
| CI/CD | OTA 更新 | 通过 HTTP 下载新固件并自动刷入 |
| 环境变量 | Kconfig + sdkconfig.defaults | `idf.py menuconfig` 配置编译选项 |
| RPC/API 调用 | MCP JSON-RPC 2.0 | 服务端调用设备工具函数 |
| WebSocket 实时通信 | WebSocket 协议 | 设备与服务端的双向实时通信 |
| MQTT 消息队列 | MQTT+UDP 协议 | 控制信令走 MQTT，音频数据走 UDP |

### ESP-IDF C++ 代码阅读提示

- **命名风格**：本项目使用 Google C++ 风格，类名大驼峰（`AudioService`），函数名大驼峰（`Initialize`），成员变量带下划线后缀（`audio_service_`），常量全大写（`MAIN_EVENT_SCHEDULE`）
- **ESP_ERROR_CHECK**：ESP-IDF 的错误检查宏，如果返回值不是 `ESP_OK` 就会打印错误并 abort
- **ESP_LOGI / ESP_LOGW / ESP_LOGE**：分级日志宏，参数是 `TAG, "format", ...`
- **extern "C"**：`app_main()` 必须用 `extern "C"` 声明，因为 ESP-IDF 框架是 C 写的

---

## 第 1 章：系统全景架构

### 项目简介

xiaozhi-esp32 是一个运行在 ESP32 系列微控制器上的 **AI 语音聊天助手**固件。它集成了大语言模型（如通义千问、DeepSeek），支持离线语音唤醒、实时流式 ASR + LLM + TTS 架构，可以控制 70+ 种不同的硬件开发板。

### 五层架构

```
+================================================================+
|                      应用层 (Application)                       |
|  +-------------------+  +--------------------+                 |
|  | Application       |  | DeviceStateMachine  |                |
|  | (单例 + 事件循环)  |->| (有限状态机)        |                 |
|  +-------------------+  +--------------------+                 |
+================================================================+
|                      协议层 (Protocol)                          |
|  +------------------+   +-------------------+                  |
|  | WebsocketProtocol|   |  MqttProtocol     |                  |
|  | (单连接)          |   | (MQTT控制+UDP音频) |                  |
|  +------------------+   +-------------------+                  |
|       都实现 Protocol 抽象接口                                   |
+================================================================+
|                    音频管线层 (Audio Pipeline)                  |
|  +-------------+ +------------+ +----------+ +----------+      |
|  | AudioCodec  | | Processor  | | Audio    | | WakeWord |      |
|  | (I2S HAL)   | | (AEC/VAD)  | | Service  | | Engine   |      |
|  +-------------+ +------------+ +----------+ +----------+      |
+================================================================+
|                    外设层 (Peripherals)                         |
|  +----------+ +--------+ +--------+ +----------+               |
|  | Display  | |  LED   | | Button | | Backlight|               |
|  | OLED/LCD | | 单/环  | |        | |          |               |
|  | (LVGL)   | |        | |        | |          |               |
|  +----------+ +--------+ +--------+ +----------+               |
+================================================================+
|                  硬件抽象层 (Board HAL)                         |
|  +----------+ +-----------+ +----------+ +--------+            |
|  |WifiBoard | |Ml307Board | |Nt26Board | | 70+    |            |
|  |(WiFi)    | |(4G LTE)   | |(N28/N26) | | boards |            |
|  +----------+ +-----------+ +----------+ +--------+            |
+================================================================+
         |              |              |
         v              v              v
    WiFi / BLE      ML307 AT       NT26 AT
    网络协议栈       4G 模组         4G 模组
```

### 一句话数据流

```
用户说话 -> 麦克风 -> 音频处理 -> Opus编码 -> 网络 -> 云端ASR -> LLM -> TTS -> 网络 -> Opus解码 -> 扬声器
```

### 入口文件

整个固件的入口极其简洁，只有 29 行代码（`main/main.cc`）：

```cpp
extern "C" void app_main(void)
{
    // 初始化 NVS（Flash 上的 key-value 存储）
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        nvs_flash_erase();
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // 创建应用单例，初始化并运行（永不返回）
    auto& app = Application::GetInstance();
    app.Initialize();
    app.Run();  // 进入主事件循环，永不返回
}
```

> 类比后端：`app_main()` 就像 `main()` 函数，`Application::GetInstance()` 是 Spring 的
> `ApplicationContext`，`Run()` 是 Node.js 的 `event loop`。

---

## 第 2 章：源码阅读路线图

### 推荐阅读顺序

按照以下顺序阅读源码，可以在约 45 分钟内建立对项目的完整理解：

| 顺序 | 文件 | 预计用时 | 理解要点 |
|------|------|----------|----------|
| 1 | `main/main.cc` | 2 分钟 | 入口函数，非常简单 |
| 2 | `main/device_state.h` | 1 分钟 | 所有 11 种设备状态 |
| 3 | `main/device_state_machine.cc` | 5 分钟 | 状态转换规则（核心逻辑） |
| 4 | `main/application.h` | 5 分钟 | "上帝类"接口，所有事件和回调 |
| 5 | `main/application.cc` | 15 分钟 | 核心控制器，连接所有模块 |
| 6 | `main/protocols/protocol.h` | 3 分钟 | 协议抽象接口 |
| 7 | `main/audio/audio_service.h` | 5 分钟 | 音频管线，队列模型 |
| 8 | `main/boards/common/board.h` | 3 分钟 | 硬件抽象基类 |
| 9 | 任选一个板卡目录，如 `main/boards/esp-box-3/` | 5 分钟 | 具体硬件实现 |

### 构建系统概览

```
menuconfig 选择 BOARD_TYPE
        |
        v
Kconfig.projbuild 定义可选板卡
        |
        v
CMakeLists.txt 根据 CONFIG_BOARD_TYPE_XXX 映射到板卡目录
        |
        v
板卡目录中的 .cc 文件使用 DECLARE_BOARD(XXXBoard) 宏
        |
        v
Board::GetInstance() 调用 create_board() 工厂函数创建实例
```

---

## 第 3 章：核心控制器 —— Application 单例与事件循环

### 单例模式

`Application` 是整个系统的"大脑"，采用线程安全的单例模式：

```cpp
class Application {
public:
    static Application& GetInstance() {
        static Application instance;  // C++11 保证线程安全
        return instance;
    }
    Application(const Application&) = delete;
    Application& operator=(const Application&) = delete;
};
```

### 事件循环（类比 Node.js）

Application 使用 FreeRTOS 的 `EventGroup` 作为事件循环，类似 Node.js 的事件驱动模型：

```
app_main()
  |
  v
Application::Initialize()     // 初始化各子系统
  |-- Board::GetInstance()    // 创建板卡实例（工厂模式）
  |-- display->SetupUI()      // 设置显示界面
  |-- audio_service_.Initialize()
  |-- audio_service_.Start()
  |-- SetCallbacks(...)       // 注册音频回调
  |-- McpServer::AddCommonTools()
  |-- McpServer::AddUserOnlyTools()
  |-- board.StartNetwork()    // 异步启动网络
  |
  v
Application::Run()            // 主事件循环（永不返回）
  |
  +---> xEventGroupWaitBits(ALL_EVENTS, portMAX_DELAY)
        |
        |-- NETWORK_CONNECTED?    --> HandleNetworkConnectedEvent()
        |                              |-- 创建 ActivationTask
        |                                  |-- CheckAssetsVersion()
        |                                  |-- CheckNewVersion()
        |                                  |-- InitializeProtocol()
        |
        |-- NETWORK_DISCONNECTED? --> HandleNetworkDisconnectedEvent()
        |-- ACTIVATION_DONE?      --> HandleActivationDoneEvent()
        |-- STATE_CHANGED?        --> HandleStateChangedEvent()
        |-- TOGGLE_CHAT?          --> HandleToggleChatEvent()
        |-- START_LISTENING?      --> HandleStartListeningEvent()
        |-- STOP_LISTENING?       --> HandleStopListeningEvent()
        |-- SEND_AUDIO?           --> PopPacket -> protocol->SendAudio()
        |-- WAKE_WORD_DETECTED?   --> HandleWakeWordDetectedEvent()
        |-- VAD_CHANGE?           --> 更新 LED 状态
        |-- SCHEDULE?             --> 执行延迟回调
        |-- CLOCK_TICK?           --> 更新状态栏
        |
        +---- loop forever ----+
```

### Schedule() —— 跨任务安全调度

`Schedule()` 是线程间通信的核心方法，类似 JavaScript 的 `setTimeout(fn, 0)`：

```cpp
void Application::Schedule(std::function<void()>&& callback) {
    {
        std::lock_guard<std::mutex> lock(mutex_);
        main_tasks_.push_back(std::move(callback));  // 加入任务队列
    }
    xEventGroupSetBits(event_group_, MAIN_EVENT_SCHEDULE);  // 唤醒主循环
}
```

**为什么需要 Schedule？** 因为 FreeRTOS 的多个任务（音频任务、网络任务、MCP 工具调用）
都在不同线程中运行，直接操作 UI 或状态是不安全的。通过 `Schedule()` 将回调投递到主任务
执行，保证了线程安全。

> 类比后端：这就像 JavaScript 中的 `Promise.resolve().then(fn)` —— 将操作投递到下一个
> 事件循环周期执行，避免并发问题。

---

## 第 4 章：状态机 —— 设备状态与转换规则

### 11 种设备状态

| 状态枚举 | 状态名 | 含义 |
|----------|--------|------|
| `kDeviceStateUnknown` | unknown | 初始状态 |
| `kDeviceStateStarting` | starting | 系统启动中 |
| `kDeviceStateWifiConfiguring` | wifi_configuring | WiFi 配置模式 |
| `kDeviceStateIdle` | idle | 待机，等待唤醒 |
| `kDeviceStateConnecting` | connecting | 正在连接服务器 |
| `kDeviceStateListening` | listening | 正在听用户说话 |
| `kDeviceStateSpeaking` | speaking | AI 正在说话（播放TTS） |
| `kDeviceStateUpgrading` | upgrading | 固件/资源升级中 |
| `kDeviceStateActivating` | activating | 设备激活中 |
| `kDeviceStateAudioTesting` | audio_testing | 音频测试模式 |
| `kDeviceStateFatalError` | fatal_error | 致命错误，不可恢复 |

### 状态转换图

```
                      +----------+
                      | Unknown  |
                      +----+-----+
                           |
                           v
                      +----------+
                +---->| Starting |
                |     +----+-----+
                |          |
                |          v
                |   +----------------+     +-----------+
                |   |WifiConfiguring |<--->|AudioTesting|
                |   +-------+--------+     +-----------+
                |           |
                |           v
                |   +-----------+             +-----------+
                +---| Activating |----------->| Upgrading |
                    +-----+------+            +-----+-----+
                          |                         |
                          v                         v
                    +---------+              +-----------+
                    |  Idle   |<-------------+ Activating|
                    +----+----+              +-----------+
                         |
               +---------+---------+
               |                   |
               v                   v
         +-----------+       +-----------+
         |Connecting |       | Listening |
         +-----+-----+       +-----+-----+
               |                   |
               +------+   +--------+
                      |   |
                      v   v
                   +-----------+        +-----------+
                   | Listening |------->| Speaking  |
                   +-----+-----+        +-----+-----+
                         |                    |
                         +-------+  +---------+
                                 |  |
                                 v  v
                              +---------+
                              |  Idle   |
                              +---------+
```

**核心流转**：`Idle` 是所有交互的起点和终点。一次完整的语音交互是：

```
Idle -> Connecting -> Listening -> Speaking -> Listening/Idle
```

### 状态转换验证

`DeviceStateMachine` 严格验证所有转换，非法转换会被拒绝并打印警告：

```cpp
bool DeviceStateMachine::IsValidTransition(DeviceState from, DeviceState to) const {
    if (from == to) return true;  // 允许原地不动
    switch (from) {
        case kDeviceStateUnknown:
            return to == kDeviceStateStarting;          // 只能进入启动
        case kDeviceStateIdle:
            return to == kDeviceStateConnecting ||       // 开始连接
                   to == kDeviceStateListening ||        // 手动模式直接听
                   to == kDeviceStateSpeaking ||         // 服务端推送TTS
                   to == kDeviceStateUpgrading ||        // OTA升级
                   to == kDeviceStateWifiConfiguring ||  // 重新配网
                   to == kDeviceStateActivating;         // 重新激活
        case kDeviceStateListening:
            return to == kDeviceStateSpeaking || to == kDeviceStateIdle;
        case kDeviceStateSpeaking:
            return to == kDeviceStateListening || to == kDeviceStateIdle;
        case kDeviceStateFatalError:
            return false;  // 致命错误不可恢复
        // ... 其他状态
    }
}
```

### 观察者模式

状态变更通过观察者模式通知所有订阅者：

```cpp
// 注册监听器
state_machine_.AddStateChangeListener(
    [this](DeviceState old_state, DeviceState new_state) {
        xEventGroupSetBits(event_group_, MAIN_EVENT_STATE_CHANGED);
    }
);

// 状态变更时，HandleStateChangedEvent 统一分发
void Application::HandleStateChangedEvent() {
    DeviceState new_state = state_machine_.GetState();
    auto display = board.GetDisplay();
    auto led = board.GetLed();
    led->OnStateChanged();  // LED 响应状态变化

    switch (new_state) {
        case kDeviceStateIdle:
            display->SetStatus("待机");
            audio_service_.EnableWakeWordDetection(true);  // 开启唤醒词
            break;
        case kDeviceStateListening:
            display->SetStatus("聆听中");
            audio_service_.EnableVoiceProcessing(true);    // 开启音频处理
            break;
        case kDeviceStateSpeaking:
            display->SetStatus("说话中");
            audio_service_.ResetDecoder();                  // 重置解码器
            break;
    }
}
```

---

## 第 5 章：通信协议层

### Protocol 抽象接口

所有协议都继承自 `Protocol` 基类，定义了统一的生命周期和回调接口：

```cpp
class Protocol {
public:
    virtual ~Protocol() = default;

    // 生命周期
    virtual bool Start() = 0;
    virtual bool OpenAudioChannel() = 0;
    virtual void CloseAudioChannel(bool send_goodbye = true) = 0;
    virtual bool IsAudioChannelOpened() const = 0;

    // 数据传输
    virtual bool SendAudio(std::unique_ptr<AudioStreamPacket> packet) = 0;
    virtual void SendWakeWordDetected(const std::string& wake_word);
    virtual void SendStartListening(ListeningMode mode);
    virtual void SendStopListening();
    virtual void SendAbortSpeaking(AbortReason reason);
    virtual void SendMcpMessage(const std::string& message);

    // 回调注册（观察者模式）
    void OnIncomingAudio(callback);      // 收到服务端音频
    void OnIncomingJson(callback);       // 收到 JSON 消息
    void OnAudioChannelOpened(callback); // 音频通道打开
    void OnAudioChannelClosed(callback); // 音频通道关闭
    void OnNetworkError(callback);       // 网络错误
};
```

> 类比后端：`Protocol` 就像一个 HTTP 客户端抽象接口，你可以用 `fetch`、`axios`、
> `WebSocket` 等不同实现，但调用方式统一。

### 两种协议实现对比

```
WebSocket 协议（简单模式）:
+--------+     TCP/WebSocket      +--------+
|  ESP32 | <===================> | Server |
|        |  JSON (文本帧)         |        |
|        |  Opus (二进制帧)       |        |
+--------+                       +--------+

MQTT + UDP 协议（高性能模式）:
+--------+    MQTT (控制信令)    +--------+
|  ESP32 | <===================> | Server |
|        |  JSON on topics       |        |
|        |                       |        |
|        | <=== UDP (音频数据) ==> |        |
|        |  AES 加密的 Opus 数据  |        |
+--------+  带序列号防重放        +--------+
```

| 特性 | WebSocket | MQTT + UDP |
|------|-----------|------------|
| 连接数 | 1 个 TCP | 1 个 MQTT + 1 个 UDP |
| 延迟 | 较高（TCP 重传） | 较低（UDP 无重传） |
| 安全性 | TLS | AES-CTR 加密 |
| 防火墙穿透 | 好（端口 80/443） | 需开放 UDP 端口 |
| 适用场景 | 简单部署、内网 | 高性能、低延迟 |

### 二进制协议版本

WebSocket 的音频帧使用二进制格式，支持 3 个版本：

```cpp
// 版本 2：带时间戳，用于服务端 AEC
struct BinaryProtocol2 {
    uint16_t version;       // 协议版本号
    uint16_t type;          // 消息类型 (0: Opus, 1: JSON)
    uint32_t reserved;      // 保留字段
    uint32_t timestamp;     // 毫秒时间戳
    uint32_t payload_size;  // 负载大小
    uint8_t payload[];      // 实际数据
} __attribute__((packed));

// 版本 3：精简头部
struct BinaryProtocol3 {
    uint8_t type;           // 消息类型
    uint8_t reserved;
    uint16_t payload_size;
    uint8_t payload[];
} __attribute__((packed));
```

### 服务端 JSON 消息类型

```cpp
// InitializeProtocol 中注册的 JSON 消息处理
protocol_->OnIncomingJson([this, display](const cJSON* root) {
    auto type = cJSON_GetObjectItem(root, "type");

    if (strcmp(type->valuestring, "tts") == 0) {
        // TTS 语音合成：start / stop / sentence_start
    } else if (strcmp(type->valuestring, "stt") == 0) {
        // 语音识别结果（用户说了什么）
    } else if (strcmp(type->valuestring, "llm") == 0) {
        // 大模型回复，含 emotion 表情
    } else if (strcmp(type->valuestring, "mcp") == 0) {
        // MCP 工具调用
        McpServer::GetInstance().ParseMessage(payload);
    } else if (strcmp(type->valuestring, "system") == 0) {
        // 系统命令（如 reboot）
    } else if (strcmp(type->valuestring, "alert") == 0) {
        // 告警消息
    }
});
```

### 协议选择逻辑

协议类型由 OTA 服务器在激活时下发配置决定：

```cpp
void Application::InitializeProtocol() {
    if (ota_->HasMqttConfig()) {
        protocol_ = std::make_unique<MqttProtocol>();
    } else if (ota_->HasWebsocketConfig()) {
        protocol_ = std::make_unique<WebsocketProtocol>();
    } else {
        protocol_ = std::make_unique<MqttProtocol>();  // 默认 MQTT
    }
    protocol_->Start();
}
```

---

## 第 6 章：音频处理管线 —— 从麦克风到云端再回来

### 架构概览

音频子系统是本项目最复杂的部分，采用**多任务 + 队列**的流水线架构：

```cpp
/*
 * 两条音频数据流:
 * 1. (MIC) -> [Processors] -> {Encode Queue} -> [Opus Encoder] -> {Send Queue} -> (Server)
 * 2. (Server) -> {Decode Queue} -> [Opus Decoder] -> {Playback Queue} -> (Speaker)
 *
 * 使用一个任务处理 MIC/Speaker/Processors，一个任务处理 Opus 编解码
 */
```

### 上行数据流（MIC -> 云端）

```
                          上行：MIC -> 云端
                          ==================

  +-------+    I2S     +-----------+   Feed()   +-------------+
  |  MIC  | ---------> |AudioCodec | ---------> | WakeWord    |---> (唤醒词检测)
  +-------+  Read()    |  Read()   |            | Engine      |
                        +-----------+            +-------------+
                              |                        |
                              | 原始 PCM               | (非唤醒词)
                              v                        v
                        +-------------+
                        | Audio       |
                        | Processor   |
                        | (AEC/VAD)   |
                        +------+------+
                               | 干净的 PCM
                               v
                   +----------------------+
                   | audio_encode_queue_  |  (PCM 数据队列)
                   +----------+-----------+
                              |
                   +----------v-----------+
                   |  OpusCodecTask       |
                   |  [Opus Encoder]      |
                   |  16kHz, 60ms 帧      |
                   +----------+-----------+
                              | Opus 压缩包
                              v
                   +----------------------+
                   | audio_send_queue_    |  (Opus 包队列)
                   +----------+-----------+
                              | PopPacketFromSendQueue()
                              v
                   Application::Run() --> protocol_->SendAudio()
```

### 下行数据流（云端 -> Speaker）

```
                          下行：云端 -> Speaker
                          ====================

  Server --> protocol_->OnIncomingAudio()
                       |
                       v
            +----------------------+
            | audio_decode_queue_  |  (Opus 包队列)
```

```
                          下行：云端 -> Speaker
                          ====================

  Server --> protocol_->OnIncomingAudio()
                    |
                    v
         +----------------------+
         | audio_decode_queue_  |  (Opus 包队列)
         +----------+-----------+
                    |
         +----------v-----------+
         |  OpusCodecTask       |
         |  [Opus Decoder]      |
         |  24kHz 输出           |
         +----------+-----------+
                    | PCM 数据
                    v
         +----------------------+
         |audio_playback_queue_ |  (PCM 数据队列)
         +----------+-----------+
                    |
         +----------v-----------+
         |  AudioOutputTask     |
         |  AudioCodec::Write() |
         +----------+-----------+
                    | I2S
                    v
               +---------+
               | Speaker |
               +---------+
```

### Opus 编码参数

```cpp
// Opus 编码器配置 (main/audio/audio_service.h)
{
    .sample_rate      = 16000,        // 输入采样率 16kHz
    .channel          = MONO,         // 单声道
    .bits_per_sample  = 16,           // 16-bit
    .bitrate          = AUTO,         // 自适应码率
    .frame_duration   = 60ms,         // 每帧 60ms
    .application_mode = AUDIO,        // 音频模式（非语音）
    .complexity       = 0,            // 最低复杂度（省CPU）
    .enable_fec       = false,        // 不启用前向纠错
    .enable_dtx       = true,         // 启用不连续传输（静音时不发数据）
    .enable_vbr       = true,         // 启用可变码率
}
```

### AudioServiceCallbacks 回调机制

AudioService 通过回调将事件通知给 Application：

```cpp
struct AudioServiceCallbacks {
    std::function<void(void)> on_send_queue_available;        // 发送队列有数据
    std::function<void(const std::string&)> on_wake_word_detected; // 唤醒词检测
    std::function<void(bool)> on_vad_change;                  // 语音活动变化
};
```

这些回调在 Application::Initialize() 中设置，通过 EventGroup 投递到主循环处理。

### 电源管理

音频子系统有自动省电机制：如果 15 秒内没有音频输入输出，自动关闭 ADC/DAC 以省电：

```cpp
#define AUDIO_POWER_TIMEOUT_MS 15000         // 15秒无活动后省电
#define AUDIO_POWER_CHECK_INTERVAL_MS 1000   // 每秒检查一次
```

---

## 第 7 章：硬件抽象层 —— Board 模式与 70+ 开发板支持

### Board 类继承体系

```
                        Board (抽象基类)
                        + GetAudioCodec()   = 0 (纯虚)
                        + GetDisplay()      (默认返回 NoDisplay)
                        + GetLed()          (默认返回 NoLed)
                        + GetBacklight()    (默认返回 nullptr)
                        + GetCamera()       (默认返回 nullptr)
                        + GetNetwork()      = 0 (纯虚)
                        + StartNetwork()    = 0 (纯虚)
                        + SetPowerSaveLevel() = 0
                             |
              +--------------+--------------+
              |              |              |
          WifiBoard     Ml307Board     DualNetworkBoard
          (WiFi STA)    (ML307 4G)     (WiFi + 4G)
              |              |
        +-----+-----+  +----+----+
        |     |     |  |         |
      EspBox ...  ... ZhCh    ...
        |
   EspBox3Board (具体实现)
        + GetAudioCodec() -> BoxAudioCodec
        + GetDisplay()    -> SpiLcdDisplay
        + GetBacklight()  -> PwmBacklight
```

### 工厂模式 + DECLARE_BOARD 宏

Board 实例通过编译期工厂创建：

```cpp
// board.h 中定义的工厂函数和宏
void* create_board();  // 由具体板卡实现

class Board {
public:
    static Board& GetInstance() {
        static Board* instance = static_cast<Board*>(create_board());
        return *instance;
    }
};

// 每个板卡目录中用此宏注册自己
#define DECLARE_BOARD(BOARD_CLASS_NAME) \
void* create_board() { \
    return new BOARD_CLASS_NAME(); \
}
```

> 类比后端：`DECLARE_BOARD` 就像 Spring 的 `@Component` 注解 —— 编译时注册，运行时由工厂创建。
> `Board::GetInstance()` 就是 `ApplicationContext.getBean(Board.class)`。

### 板卡目录结构

每个板卡都有自己的目录，包含配置和实现：

```
main/boards/esp-box-3/
    config.h          -- GPIO 引脚定义、音频/显示参数
    config.json       -- 构建目标配置（芯片型号、Flash 大小等）
    esp_box3_board.cc -- 板卡类实现
```

### config.h 示例（ESP-BOX-3）

```cpp
// main/boards/esp-box-3/config.h
#define AUDIO_INPUT_SAMPLE_RATE  24000    // 麦克风采样率
#define AUDIO_OUTPUT_SAMPLE_RATE 24000    // 扬声器采样率
#define AUDIO_INPUT_REFERENCE    true     // 启用参考信号（AEC需要）

// I2S 音频总线引脚
#define AUDIO_I2S_GPIO_MCLK GPIO_NUM_2
#define AUDIO_I2S_GPIO_WS   GPIO_NUM_45
#define AUDIO_I2S_GPIO_BCLK GPIO_NUM_17
#define AUDIO_I2S_GPIO_DIN  GPIO_NUM_16   // 麦克风数据
#define AUDIO_I2S_GPIO_DOUT GPIO_NUM_15   // 扬声器数据

// 显示屏参数
#define DISPLAY_WIDTH   320
#define DISPLAY_HEIGHT  240
#define DISPLAY_BACKLIGHT_PIN GPIO_NUM_47

// 按钮引脚
#define BOOT_BUTTON_GPIO GPIO_NUM_0
```

### 具体板卡实现示例（ESP-BOX-3）

```cpp
// main/boards/esp-box-3/esp_box3_board.cc (简化)
class EspBox3Board : public WifiBoard {
private:
    i2c_master_bus_handle_t i2c_bus_;
    Button boot_button_;
    Display* display_;

    // 初始化硬件总线
    void InitializeI2c() { /* I2C 总线配置 */ }
    void InitializeSpi() { /* SPI 总线配置（用于显示屏） */ }

    // 初始化按钮
    void InitializeButtons() {
        boot_button_.OnClick([this]() {
            auto& app = Application::GetInstance();
            if (app.GetDeviceState() == kDeviceStateStarting) {
                EnterWifiConfigMode();  // 启动时按键进入配网模式
                return;
            }
            app.ToggleChatState();      // 运行时按键切换聊天
        });
    }

    // 初始化显示屏
    void InitializeIli9341Display() { /* LCD 驱动初始化 */ }

public:
    EspBox3Board() : boot_button_(BOOT_BUTTON_GPIO) {
        InitializeI2c();            // 1. I2C 总线
        InitializeSpi();            // 2. SPI 总线
        InitializeIli9341Display(); // 3. 显示屏
        InitializeButtons();        // 4. 按钮
        GetBacklight()->RestoreBrightness(); // 5. 恢复亮度
    }

    // 返回音频编解码器（使用 static 局部变量实现单例）
    virtual AudioCodec* GetAudioCodec() override {
        static BoxAudioCodec audio_codec(
            i2c_bus_,
            AUDIO_INPUT_SAMPLE_RATE, AUDIO_OUTPUT_SAMPLE_RATE,
            AUDIO_I2S_GPIO_MCLK, AUDIO_I2S_GPIO_BCLK,
            AUDIO_I2S_GPIO_WS, AUDIO_I2S_GPIO_DOUT, AUDIO_I2S_GPIO_DIN,
            AUDIO_CODEC_PA_PIN, AUDIO_CODEC_ES8311_ADDR,
            AUDIO_CODEC_ES7210_ADDR, AUDIO_INPUT_REFERENCE
        );
        return &audio_codec;
    }

    virtual Display* GetDisplay() override { return display_; }

    virtual Backlight* GetBacklight() override {
        static PwmBacklight backlight(DISPLAY_BACKLIGHT_PIN, DISPLAY_BACKLIGHT_OUTPUT_INVERT);
        return &backlight;
    }
};

// 注册板卡
DECLARE_BOARD(EspBox3Board);
```

### Null Object 模式

对于没有某种硬件的板卡，基类提供空实现：

```cpp
// 没有屏幕的板卡：GetDisplay() 返回 NoDisplay
class NoDisplay : public Display {
    virtual bool Lock(int timeout_ms = 0) override { return true; }
    virtual void Unlock() override {}
};

// 没有 LED 的板卡：GetLed() 返回 NoLed
class NoLed : public Led {
    virtual void OnStateChanged() override {}
};
```

> 类比后端：这就是 Null Object 模式 —— 类似返回空集合而不是 null，调用方无需判空。

---

## 第 8 章：显示子系统

### Display 类继承体系

```
Display (抽象基类)
  + SetStatus()          设置状态文字
  + ShowNotification()   显示通知
  + SetEmotion()         设置表情
  + SetChatMessage()     显示聊天消息
  + SetTheme()           切换主题
  + UpdateStatusBar()    更新状态栏
  + Lock()/Unlock()      线程安全锁（纯虚）
  |
  |-- NoDisplay (空实现，用于无屏幕板卡)
  |
  +-- LvglDisplay (基于 LVGL 图形库)
        + 状态栏（网络、电量、音量图标）
        + 聊天消息（用户/助手/系统）
        + 表情显示
        |
        +-- OledDisplay (单色 OLED，128x64 或 128x32)
        |
        +-- LcdDisplay (彩色 LCD)
              + SpiLcdDisplay  (SPI 接口)
              + RgbLcdDisplay  (RGB 并行接口)
              + MipiLcdDisplay (MIPI DSI 接口)

  emote::EmoteDisplay (独立分支，基于动画的表情显示)
```

### DisplayLockGuard —— RAII 线程安全

显示操作必须在持锁状态下进行，`DisplayLockGuard` 用 RAII 模式自动管理锁：

```cpp
// display.h
class DisplayLockGuard {
public:
    DisplayLockGuard(Display *display) : display_(display) {
        if (!display_->Lock(30000)) {  // 最多等 30 秒
            ESP_LOGE("Display", "Failed to lock display");
        }
    }
    ~DisplayLockGuard() {
        display_->Unlock();
    }
private:
    Display *display_;
};

// 使用方式（类似 std::lock_guard）
void SomeFunction() {
    DisplayLockGuard lock(display);  // 构造时加锁
    display->SetChatMessage("assistant", "你好！");
    // 函数结束时自动解锁（析构函数）
}
```

> 类比后端：和 C++ 的 `std::lock_guard<std::mutex>` 完全一样的 RAII 模式。

### 主题系统

支持亮色/暗色主题切换，通过 MCP 工具 `self.screen.set_theme` 可以远程切换：

```cpp
// 主题由 LvglThemeManager 管理
auto& theme_manager = LvglThemeManager::GetInstance();
auto theme = theme_manager.GetTheme("dark");  // 或 "light"
display->SetTheme(theme);
```

---

## 第 9 章：LED 指示系统

### Led 接口

LED 系统的设计极其简洁，只有一个方法：

```cpp
// main/led/led.h
class Led {
public:
    virtual ~Led() = default;
    virtual void OnStateChanged() = 0;  // 唯一的方法
};
```

当设备状态变化时，`HandleStateChangedEvent()` 会调用 `led->OnStateChanged()`，
LED 实现自行决定如何显示当前状态。

### 三种 LED 实现

| 实现类 | 硬件类型 | 特点 |
|--------|----------|------|
| `SingleLed` | 单个 WS2812 RGB LED | 最常见，支持闪烁、呼吸动画 |
| `CircularStrip` | WS2812 LED 环形灯带 | 多灯效果：旋转、彩虹、跑马灯 |
| `GpioLed` | 普通 GPIO LED | 只能开/关，无颜色 |

### 状态-颜色映射

以 `SingleLed` 为例，不同状态对应不同颜色和动画：

| 设备状态 | LED 颜色 | 动画 |
|----------|----------|------|
| Idle（待机） | 绿色 | 常亮 |
| Connecting（连接中） | 黄色 | 缓慢闪烁 |
| Listening（聆听中） | 蓝色 | 闪烁（VAD 检测到声音时加速） |
| Speaking（说话中） | 青色 | 缓慢呼吸 |
| Upgrading（升级中） | 紫色 | 快速闪烁 |
| Error（错误） | 红色 | 快速闪烁 |

### Null Object 模式

没有 LED 的板卡使用 `NoLed`，无需判空：

```cpp
class NoLed : public Led {
public:
    virtual void OnStateChanged() override {}
};
```

---

## 第 10 章：MCP 设备控制 —— 让 AI 操控硬件

### 什么是 MCP

MCP（Model Context Protocol）是一个开放协议，允许大语言模型（LLM）通过标准化的接口
调用外部工具。在本项目中，ESP32 设备作为 **MCP Server**，暴露工具函数供云端 AI 调用。

> 类比后端：MCP 就像 JSON-RPC 风格的 API。云端的 LLM 是客户端，ESP32 设备是服务端。
> LLM 调用 `self.audio_speaker.set_volume` 就像前端调用后端的 `/api/volume` 接口。

### MCP 消息流

```
   云端 LLM                        ESP32 设备
   +--------+                     +----------------------------------+
   |        |   JSON-RPC req      |  Application::OnIncomingJson()   |
   |  LLM   | ==================> |    type: "mcp"                   |
   |  调用  |                     |         |                        |
   |  工具  |                     |         v                        |
   |        |                     |  McpServer::ParseMessage()       |
   |        |                     |    +-- tools/list: 返回工具列表   |
   |        |                     |    +-- tools/call: 执行工具       |
   |        |                     |         |                        |
   |        |   JSON-RPC resp     |         v                        |
   |        | <================== |  tool callback -> ReplyResult()  |
   +--------+                     +----------------------------------+
```

### McpTool 类型系统

每个 MCP 工具由名称、描述、参数列表和回调函数组成：

```cpp
// 工具参数支持三种类型
enum PropertyType {
    kPropertyTypeBoolean,   // 布尔值
    kPropertyTypeInteger,   // 整数（支持范围校验）
    kPropertyTypeString     // 字符串
};

// 参数定义支持默认值和范围限制
Property("volume", kPropertyTypeInteger, 0, 100)         // 整数，范围 0-100
Property("theme", kPropertyTypeString)                    // 字符串，必填
Property("quality", kPropertyTypeInteger, 80, 1, 100)    // 整数，默认80，范围1-100

// 返回值支持多种类型
using ReturnValue = std::variant<bool, int, std::string, cJSON*, ImageContent*>;
```

### 内置工具列表

**通用工具（AI 可见）**：

| 工具名 | 功能 | 参数 |
|--------|------|------|
| `self.get_device_status` | 获取设备状态（音量、电量等） | 无 |
| `self.audio_speaker.set_volume` | 设置音量 | volume: int(0-100) |
| `self.screen.set_brightness` | 设置屏幕亮度 | brightness: int(0-100) |
| `self.screen.set_theme` | 切换主题 | theme: string("light"/"dark") |
| `self.camera.take_photo` | 拍照并解释 | question: string |

**用户工具（仅用户可见，AI 不可调用）**：

| 工具名 | 功能 |
|--------|------|
| `self.get_system_info` | 获取系统信息 |
| `self.reboot` | 重启设备 |
| `self.upgrade_firmware` | OTA 固件升级 |
| `self.screen.snapshot` | 屏幕截图上传 |
| `self.screen.preview_image` | 在屏幕预览图片 |

### 工具注册示例

```cpp
// 注册音量控制工具 (main/mcp_server.cc)
AddTool("self.audio_speaker.set_volume",
    "Set the volume of the audio speaker...",
    PropertyList({
        Property("volume", kPropertyTypeInteger, 0, 100)  // 0-100 的整数
    }),
    [&board](const PropertyList& properties) -> ReturnValue {
        auto codec = board.GetAudioCodec();
        codec->SetOutputVolume(properties["volume"].value<int>());
        return true;  // 返回成功
    }
);
```

### 如何添加自定义工具

在板卡的 `InitializeTools()` 方法中添加：

```cpp
class MyBoard : public WifiBoard {
public:
    void InitializeTools() override {
        auto& mcp = McpServer::GetInstance();

        // 添加一个自定义工具
        mcp.AddTool("self.my_device.led_control",
            "控制板载 LED 颜色",
            PropertyList({
                Property("color", kPropertyTypeString)  // 颜色名称
            }),
            [this](const PropertyList& props) -> ReturnValue {
                auto color = props["color"].value<std::string>();
                // 执行硬件操作...
                return std::string("LED 设置为 " + color);
            });
    }
};
```

### MCP JSON-RPC 消息格式

```json
// 工具调用请求（云端 -> 设备）
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "self.audio_speaker.set_volume",
        "arguments": { "volume": 80 }
    }
}

// 工具调用响应（设备 -> 云端）
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [{ "type": "text", "text": "true" }],
        "isError": false
    }
}
```

---

## 第 11 章：OTA 与设备激活

### 激活流程

设备首次启动或网络连接后，会执行 `ActivationTask()` 完成激活：

```cpp
void Application::ActivationTask() {
    ota_ = std::make_unique<Ota>();
    CheckAssetsVersion();   // 1. 检查资源版本（字体、表情等）
    CheckNewVersion();      // 2. 检查固件版本
    InitializeProtocol();   // 3. 初始化通信协议
    xEventGroupSetBits(event_group_, MAIN_EVENT_ACTIVATION_DONE);
}
```

激活任务在独立的 FreeRTOS 任务中运行（优先级 2），不阻塞主循环。

### 完整激活流程图

```
网络连接成功
    |
    v
+--------------------+
| CheckAssetsVersion |  检查是否有新资源需要下载
+--------+-----------+
         |
         v
+--------------------+
| CheckNewVersion    |  向 OTA 服务器查询新版本
+--------+-----------+
         |
         +-- 有新版本？--> UpgradeFirmware() --> 重启
         |
         +-- 需要激活？--> 显示激活码 --> 轮询激活状态
         |
         +-- 无需操作 --> InitializeProtocol()
         |
         v
   发送 ACTIVATION_DONE 事件
         |
         v
   进入 Idle 状态，播放成功提示音
```

### 固件升级

OTA 升级采用 A/B 分区方案，新固件写入备用分区后重启：

```cpp
bool Application::UpgradeFirmware(const std::string& url, const std::string& version) {
    // 1. 关闭音频通道
    if (protocol_ && protocol_->IsAudioChannelOpened()) {
        protocol_->CloseAudioChannel();
    }
    // 2. 设置高性能模式
    board.SetPowerSaveLevel(PowerSaveLevel::PERFORMANCE);
    // 3. 停止音频服务
    audio_service_.Stop();
    // 4. 下载并刷入新固件
    bool success = Ota::Upgrade(url, [this, display](int progress, size_t speed) {
        // 进度回调（通过 Schedule 在主线程更新 UI）
        Schedule([display, progress, speed]() {
            // 显示 "进度% 速度KB/s"
        });
    });
    // 5. 成功则重启，失败则恢复音频服务
    if (success) {
        Reboot();
    } else {
        audio_service_.Start();
    }
    return success;
}
```

---

## 第 12 章：设置与系统信息

### Settings —— NVS 持久化存储

`Settings` 是对 ESP-IDF NVS（Non-Volatile Storage）的薄封装，提供命名空间的 key-value 存储：

```cpp
// main/settings.h
class Settings {
public:
    Settings(const std::string& ns, bool read_write = false);
    ~Settings();

    std::string GetString(const std::string& key, const std::string& default_value = "");
    void SetString(const std::string& key, const std::string& value);
    int32_t GetInt(const std::string& key, int32_t default_value = 0);
    void SetInt(const std::string& key, int32_t value);
    bool GetBool(const std::string& key, bool default_value = false);
    void SetBool(const std::string& key, bool value);
    void EraseKey(const std::string& key);
    void EraseAll();
};
```

使用方式：

```cpp
// 读取设置（只读模式）
Settings settings("board");
std::string uuid = settings.GetString("uuid", "");

// 写入设置（读写模式）
Settings settings("assets", true);
settings.SetString("download_url", "http://example.com/assets.bin");

// RAII：析构时自动提交变更到 Flash
```

> 类比后端：NVS 就像一个简单的 Redis —— key-value 持久化存储，支持命名空间隔离。

### SystemInfo —— 设备信息查询

```cpp
class SystemInfo {
public:
    static size_t GetFlashSize();              // Flash 大小
    static size_t GetFreeHeapSize();           // 可用堆内存
    static std::string GetMacAddress();        // MAC 地址
    static std::string GetChipModelName();     // 芯片型号
    static std::string GetUserAgent();         // 浏览器式 User-Agent
    static void PrintHeapStats();              // 打印内存统计
    static void PrintTaskList();               // 打印任务列表
};
```

---

## 第 13 章：线程模型与并发安全

### 任务优先级分配

| 任务 | 优先级 | 职责 |
|------|--------|------|
| Main Task（主循环） | 10 | Application::Run()，处理所有事件 |
| Audio Input Task | 由 AudioService 管理 | MIC 数据读取、唤醒词检测 |
| Audio Output Task | 由 AudioService 管理 | 扬声器数据播放 |
| Opus Codec Task | 由 AudioService 管理 | Opus 编解码 |
| Activation Task | 2 | OTA 检查、版本升级、协议初始化 |
| LVGL Task | 由 LVGL 管理 | 显示渲染 |
| Network Tasks | 由 ESP-IDF 管理 | WiFi/MQTT/TLS 协议栈 |

### 线程安全机制

本项目使用了多种并发安全手段：

**1. mutex + lock_guard（互斥锁）**

```cpp
// Schedule() 使用 mutex 保护任务队列
void Application::Schedule(std::function<void()>&& callback) {
    {
        std::lock_guard<std::mutex> lock(mutex_);
        main_tasks_.push_back(std::move(callback));
    }
    xEventGroupSetBits(event_group_, MAIN_EVENT_SCHEDULE);
}
```

**2. atomic（原子变量）**

```cpp
// 状态机使用 atomic 保证状态读写的原子性
class DeviceStateMachine {
    std::atomic<DeviceState> current_state_{kDeviceStateUnknown};
};
```

**3. EventGroup（事件组）**

```cpp
// 跨任务事件通知（类似信号量）
xEventGroupSetBits(event_group_, MAIN_EVENT_WAKE_WORD_DETECTED);  // 发送
auto bits = xEventGroupWaitBits(event_group_, ALL_EVENTS, ...);   // 等待
```

**4. DisplayLockGuard（RAII 锁）**

```cpp
// 任何线程操作显示前必须持锁
DisplayLockGuard lock(display);
display->SetChatMessage("assistant", "你好");
```

**5. Schedule()（投递到主线程）**

MCP 工具回调通过 `Schedule()` 确保在主线程执行：

```cpp
void McpServer::DoToolCall(...) {
    auto& app = Application::GetInstance();
    app.Schedule([this, id, tool_iter, arguments]() {
        // 安全地在主线程执行工具
        ReplyResult(id, (*tool_iter)->Call(arguments));
    });
}
```

**6. TaskPriorityReset（RAII 优先级管理）**

临时降低任务优先级以执行低优先级操作：

```cpp
class TaskPriorityReset {
public:
    TaskPriorityReset(BaseType_t priority) {
        original_priority_ = uxTaskPriorityGet(NULL);
        vTaskPrioritySet(NULL, priority);  // 临时降低优先级
    }
    ~TaskPriorityReset() {
        vTaskPrioritySet(NULL, original_priority_);  // 恢复
    }
};

// 用法：拍照时降低优先级避免影响音频
TaskPriorityReset priority_reset(1);
camera->Capture();
```

---

## 第 14 章：设计模式总结

| 模式 | 使用位置 | 后端类比 |
|------|----------|----------|
| **单例 (Singleton)** | Application、Board、McpServer、Assets | Spring 的单例 Bean |
| **工厂方法 (Factory)** | `create_board()` + `DECLARE_BOARD()` 宏 | 依赖注入容器 |
| **抽象接口 (Interface)** | Protocol、AudioCodec、Display、Led | Java Interface / Go Interface |
| **观察者 (Observer)** | DeviceStateMachine 状态变更通知 | EventEmitter / EventBus |
| **策略 (Strategy)** | Assets 的 LvglStrategy / EmoteStrategy | 策略模式，运行时切换算法 |
| **Null Object** | NoDisplay、NoLed、NoAudioCodec | 返回空集合而非 null |
| **RAII** | DisplayLockGuard、TaskPriorityReset、Settings | C++ 的 lock_guard |
| **回调注册** | Protocol 的 OnXxx() 方法 | Express 中间件注册 |
| **事件驱动 (Reactor)** | Application::Run() 事件循环 | Node.js 事件循环 |
| **状态机 (FSM)** | DeviceStateMachine | 订单状态机（待支付->已支付->已发货） |
| **建造者 (Builder)** | AudioServiceCallbacks | Builder 模式配置对象 |
| **模板方法** | Board 基类定义流程，子类重写具体步骤 | Spring 的 AbstractTemplate |

---

## 第 15 章：扩展指南

### 如何添加新板卡

**步骤 1：创建板卡目录**

```
main/boards/my-board/
    config.h           -- GPIO 引脚和参数定义
    config.json         -- 构建配置
    my_board.cc         -- 板卡类实现
```

**步骤 2：编写 config.h**

```cpp
#ifndef _BOARD_CONFIG_H_
#define _BOARD_CONFIG_H_
#include <driver/gpio.h>

// 音频参数
#define AUDIO_INPUT_SAMPLE_RATE  16000
#define AUDIO_OUTPUT_SAMPLE_RATE 24000
#define AUDIO_INPUT_REFERENCE    false

// I2S 引脚
#define AUDIO_I2S_GPIO_MCLK GPIO_NUM_2
#define AUDIO_I2S_GPIO_BCLK GPIO_NUM_17
#define AUDIO_I2S_GPIO_WS   GPIO_NUM_45
#define AUDIO_I2S_GPIO_DIN  GPIO_NUM_16
#define AUDIO_I2S_GPIO_DOUT GPIO_NUM_15

// 显示屏
#define DISPLAY_WIDTH   320
#define DISPLAY_HEIGHT  240
#define DISPLAY_BACKLIGHT_PIN GPIO_NUM_47

// 按钮
#define BOOT_BUTTON_GPIO GPIO_NUM_0
#endif
```

**步骤 3：实现板卡类**

```cpp
// main/boards/my-board/my_board.cc
#include "wifi_board.h"
#include "codecs/es8311_audio_codec.h"
#include "display/no_display.h"  // 如果没有屏幕
#include "config.h"

class MyBoard : public WifiBoard {
public:
    MyBoard() {
        InitializeI2c();
        // 其他硬件初始化...
    }

    virtual AudioCodec* GetAudioCodec() override {
        static Es8311AudioCodec codec(i2c_bus_, ...);
        return &codec;
    }

    // 没有屏幕的板卡可以不重写，基类返回 NoDisplay
    // 没有背光可以不重写，基类返回 nullptr
};

DECLARE_BOARD(MyBoard);
```

**步骤 4：注册到构建系统**

在 `main/Kconfig.projbuild` 中添加板卡选项，在 `main/CMakeLists.txt` 中添加映射关系。

### 如何添加 MCP 工具

在板卡构造函数或 `InitializeTools()` 中：

```cpp
auto& mcp = McpServer::GetInstance();
mcp.AddTool("self.my_sensor.read",
    "读取温湿度传感器",
    PropertyList(),
    [this](const PropertyList& props) -> ReturnValue {
        float temp;
        ReadTemperature(temp);
        return std::string("温度: " + std::to_string(temp) + "°C");
    });
```

### 如何添加新的音频编解码芯片

1. 在 `main/audio/codecs/` 下创建新文件
2. 继承 `AudioCodec` 基类
3. 实现 `Read()`（麦克风数据）、`Write()`（扬声器数据）等虚方法

### 如何添加新的通信协议

1. 在 `main/protocols/` 下创建新文件
2. 继承 `Protocol` 基类
3. 实现所有纯虚方法（`Start`、`OpenAudioChannel`、`SendAudio` 等）
4. 在 `InitializeProtocol()` 中添加协议选择逻辑

---

## 第 16 章：端到端语音交互追踪

下面完整追踪一次"你好小智" -> AI 回答的完整数据流：

```
步骤 1: 用户说 "你好小智"
         |
         v
步骤 2: AudioInputTask 从 MIC 读取 PCM 数据
         |  (main/audio/audio_service.cc)
         v
步骤 3: WakeWord Engine 检测到唤醒词
         |  (main/audio/wake_words/)
         v
步骤 4: 触发 on_wake_word_detected 回调
         |  -> xEventGroupSetBits(MAIN_EVENT_WAKE_WORD_DETECTED)
         v
步骤 5: Application::HandleWakeWordDetectedEvent()
         |  (main/application.cc:776)
         |  状态: Idle -> Connecting
         v
步骤 6: EncodeWakeWord() 编码唤醒词前的音频缓冲
         |  (保存唤醒词上下文)
         v
步骤 7: Schedule() -> ContinueWakeWordInvoke()
         |  等待状态变更事件处理完再继续
         v
步骤 8: protocol_->OpenAudioChannel()
         |  建立 WebSocket 或 MQTT+UDP 连接
         |  发送 hello 消息（音频参数、MCP 能力等）
         v
步骤 9: protocol_->SendWakeWordDetected("ni hao xiao zhi")
         |  发送唤醒词数据和音频
         v
步骤 10: SetListeningMode() -> 状态: Connecting -> Listening
         |
         v
步骤 11: audio_service_.EnableVoiceProcessing(true)
         |  启动 AEC（回声消除）和 VAD（语音活动检测）
         |  MIC -> Processor -> Encode Queue -> Opus Encoder -> Send Queue
         v
步骤 12: 主循环中 PopPacketFromSendQueue() -> protocol_->SendAudio()
         |  持续发送 Opus 音频包到云端
         |  服务端进行 ASR（语音识别）
         v
步骤 13: VAD 检测到用户停止说话 -> 自动结束（或用户按按钮手动结束）
         |  protocol_->SendStopListening()
         v
步骤 14: 云端处理: ASR -> LLM -> TTS 流式输出
         |  服务端开始发送 JSON 和音频数据
         v
步骤 15: 收到 {type: "stt", text: "你好"}
         |  -> display->SetChatMessage("user", "你好")
         |  收到 {type: "tts", state: "start"}
         |  -> 状态: Listening -> Speaking
         v
步骤 16: 收到 Opus 音频包
         |  -> audio_decode_queue_ -> Opus Decoder -> playback_queue_ -> Speaker
         |  收到 {type: "tts", state: "sentence_start", text: "你好呀！"}
         |  -> display->SetChatMessage("assistant", "你好呀！")
         v
步骤 17: 收到 {type: "tts", state: "stop"}
         |  -> 状态: Speaking -> Listening（自动模式）或 Idle（手动模式）
         |  -> audio_service_.EnableWakeWordDetection(true)
         |  回到步骤 1，等待下一次唤醒
```

---

## 附录 A：关键数据结构参考

### AudioStreamPacket（音频数据包）

```cpp
struct AudioStreamPacket {
    int sample_rate = 0;        // 采样率
    int frame_duration = 0;     // 帧时长（毫秒）
    uint32_t timestamp = 0;     // 时间戳
    std::vector<uint8_t> payload; // 实际音频数据
};
```

### BinaryProtocol2 / BinaryProtocol3（二进制协议帧头）

```cpp
struct BinaryProtocol2 {
    uint16_t version;       // 协议版本
    uint16_t type;          // 0: Opus, 1: JSON
    uint32_t reserved;
    uint32_t timestamp;     // 用于服务端 AEC
    uint32_t payload_size;
    uint8_t payload[];
};

struct BinaryProtocol3 {
    uint8_t type;
    uint8_t reserved;
    uint16_t payload_size;
    uint8_t payload[];
};
```

### DeviceState（设备状态枚举）

```cpp
enum DeviceState {
    kDeviceStateUnknown = 0,        // 初始
    kDeviceStateStarting,           // 启动中
    kDeviceStateWifiConfiguring,    // WiFi 配置
    kDeviceStateIdle,               // 待机
    kDeviceStateConnecting,         // 连接中
    kDeviceStateListening,          // 聆听中
    kDeviceStateSpeaking,           // 说话中
    kDeviceStateUpgrading,          // 升级中
    kDeviceStateActivating,         // 激活中
    kDeviceStateAudioTesting,       // 音频测试
    kDeviceStateFatalError          // 致命错误
};
```

### 主事件位定义

```cpp
#define MAIN_EVENT_SCHEDULE             (1 << 0)   // 延迟任务
#define MAIN_EVENT_SEND_AUDIO           (1 << 1)   // 发送音频
#define MAIN_EVENT_WAKE_WORD_DETECTED   (1 << 2)   // 唤醒词检测
#define MAIN_EVENT_VAD_CHANGE           (1 << 3)   // VAD 变化
#define MAIN_EVENT_ERROR                (1 << 4)   // 错误
#define MAIN_EVENT_ACTIVATION_DONE      (1 << 5)   // 激活完成
#define MAIN_EVENT_CLOCK_TICK           (1 << 6)   // 时钟滴答
#define MAIN_EVENT_NETWORK_CONNECTED    (1 << 7)   // 网络已连接
#define MAIN_EVENT_NETWORK_DISCONNECTED (1 << 8)   // 网络断开
#define MAIN_EVENT_TOGGLE_CHAT          (1 << 9)   // 切换聊天
#define MAIN_EVENT_START_LISTENING      (1 << 10)  // 开始聆听
#define MAIN_EVENT_STOP_LISTENING       (1 << 11)  // 停止聆听
#define MAIN_EVENT_STATE_CHANGED        (1 << 12)  // 状态已变更
```

### AudioServiceCallbacks（音频服务回调）

```cpp
struct AudioServiceCallbacks {
    std::function<void(void)> on_send_queue_available;           // 发送队列有数据
    std::function<void(const std::string&)> on_wake_word_detected; // 唤醒词检测
    std::function<void(bool)> on_vad_change;                     // VAD 状态变化
    std::function<void(void)> on_audio_testing_queue_full;       // 音频测试队列满
};
```

---

## 附录 B：构建与烧录速查

### 环境搭建

```bash
# 1. 安装 ESP-IDF（v5.3+）
git clone https://github.com/espressif/esp-idf.git
cd esp-idf && ./install.sh && . ./export.sh

# 2. 克隆项目
git clone https://github.com/78/xiaozhi-esp32.git
cd xiaozhi-esp32
```

### 编译与烧录

```bash
# 选择目标芯片
idf.py set-target esp32s3

# 配置板卡类型（menuconfig -> Board Type）
idf.py menuconfig

# 编译
idf.py build

# 烧录
idf.py flash

# 串口监视器（Ctrl+] 退出）
idf.py monitor

# 一键编译+烧录+监视
idf.py flash monitor
```

### 常用 menuconfig 配置

- **Board Type**：选择对应的开发板
- **Wake Word**：配置唤醒词
- **AEC Mode**：回声消除模式（关闭/设备端/服务端）
- **Language**：界面语言

---

## 附录 C：推荐阅读与资源

| 资源 | 说明 |
|------|------|
| ESP-IDF 编程指南 | ESP32 开发框架官方文档 |
| LVGL 文档 | 嵌入式图形库文档 |
| MCP 规范 (modelcontextprotocol.io) | Model Context Protocol 规范 |
| ESP-ADF | ESP 音频开发框架 |
| Opus 编解码器 (opus-codec.org) | Opus 音频编解码器 |
| ESP-SR (github.com/espressif/esp-sr) | ESP 语音识别框架（唤醒词） |
| FreeRTOS (freertos.org) | 实时操作系统文档 |
