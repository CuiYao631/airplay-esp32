# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

## 项目概述

ESP32 AirPlay 2 接收器 — 将 ESP32/ESP32-S3/ESP32-P4 开发板变为 AirPlay 2 音箱的固件。支持 ALAC 和 AAC 解码、蓝牙 A2DP（仅 ESP32）、W5500 以太网（Esparagus Audio Brick）、OLED/TFT 显示屏、硬件按键和 OTA 更新。

## 构建与烧录

**PlatformIO**（推荐）：
```bash
pio run -e <env> -t build          # 构建固件
pio run -e <env> -t upload          # 构建 + 通过 USB 烧录
pio run -e <env> -t monitor         # 串口监视器（115200 波特率）
pio run -e <env> -t uploadfs        # 从 data/ 烧录 SPIFFS
pio run -e <env> -t menuconfig      # Kconfig 配置
```

**ESP-IDF**（原生）：
```bash
source /path/to/esp-idf/export.sh
idf.py set-target esp32s3           # 或 esp32、esp32p4
idf.py build
idf.py -p /dev/ttyUSB0 flash
idf.py -p /dev/ttyUSB0 monitor
```

## 构建环境

| 环境 | 开发板 | 备注 |
|---|---|---|
| `esp32s3` | ESP32-S3 + 外部 DAC（如 PCM5102A） | 默认 |
| `esp32s3-jtag` | 带 JTAG 的 ESP32-S3 | 继承自 esp32s3 |
| `squeezeamp` | ESP32 + TAS5756 DAC/功放 | 8MB flash |
| `squeezeamp-bt` | 同上 + 蓝牙 A2DP | |
| `squeezeamp-4m` | 4MB flash 版 SqueezeAMP | |
| `esparagus-audio-brick` | ESP32 + TAS5825M DAC/功放 | |
| `esparagus-audio-brick-bt` | 同上 + 蓝牙 + 以太网 | |
| `esparagus-audio-brick-s3` | ESP32-S3 + TAS5825M | |
| `esparagus-louder` | ESP32 + TAS5825M + 额外增益 | |
| `esparagus-louder-bt` | Louder + 蓝牙 | |
| `esparagus-louder-s3` | ESP32-S3 + Louder | |

Sdkconfig 默认值通过 `cmake_extra_args` 分层叠加（从左到右覆盖）。自定义板级配置：创建 `sdkconfig.user.<name>` + `user_platformio.ini` 即可扩展任意环境，无需修改主配置。

## 架构

```
main/
├── main.c                  # 入口点 — 初始化 NVS、WiFi，启动 AirPlay 服务
├── settings.c              # NVS 持久化存储：设备名称、WiFi 凭据、音量
├── audio/                  # 音频管线
│   ├── audio_receiver.c    # RTSP 会话管理器 — 编排流（缓冲/非缓冲）
│   ├── audio_stream.c      # 基础流抽象
│   ├── audio_stream_buffered.c   # AirPlay 2 AAC（深度抖动缓冲）
│   ├── audio_stream_realtime.c     # AirPlay 1 ALAC（低延迟 UDP）
│   ├── audio_decoder.c     # ALAC 和 AAC 解码器
│   ├── audio_buffer.c      # 接收器与输出之间的帧缓冲
│   ├── audio_timing.c      # 基于 PTP 的时序 — 提前/滞后帧处理
│   ├── audio_resample.c    # 采样率转换（44.1→48kHz）
│   ├── audio_output.c      # I2S 输出
│   ├── audio_output_spdif.c # S/PDIF 输出
│   ├── audio_output_usb.c  # USB 音频输出
│   ├── audio_crypto.c      # AirPlay 加密
│   ├── a2dp_sink.c         # 蓝牙 A2DP 接收器
│   └── eq_events.c         # EQ 参数变更（TAS58xx）
├── rtsp/                   # RTSP 协议服务器
│   ├── rtsp_server.c       # RTSP 连接处理器
│   ├── rtsp_conn.c         # 连接管理
│   ├── rtsp_handlers.c     # RTSP 方法处理器（OPTIONS、SETUP、PLAY 等）
│   ├── rtsp_events.c       # RTSP 事件处理（含蓝牙透传）
│   ├── rtsp_crypto.c       # RTSP 层加密
│   ├── rtsp_fairplay.c     # Apple FairPlay 集成
│   └── rtsp_rsa.c          # RSA 加密
├── hap/                    # HomeKit 配件协议
│   ├── hap.c               # 核心 HAP
│   ├── hap_pair_setup.c    # 配对设置（SRP 握手）
│   ├── hap_pair_verify.c   # 配对验证（Ed25519）
│   ├── hap_crypto.c        # HAP 加密
│   └── srp.c               # SRP-6a 密钥交换
├── plist/                  # Apple Property List 解析
├── network/                # 网络栈
│   ├── wifi.c              # WiFi AP+STA、强制门户、自动重连
│   ├── ethernet.c          # W5500 SPI 以太网驱动
│   ├── mdns_airplay.c      # mDNS AirPlay 服务公告
│   ├── ptp_clock.c         # 精确时间协议时钟
│   ├── ntp_clock.c         # NTP 时间同步回退
│   ├── web_server.c        # HTTP 配置/控制服务器
│   ├── ota.c               # OTA 固件更新
│   ├── dns_server.c        # 强制门户 DNS
│   └── log_stream.c        # 远程日志流
├── dacp_client.c           # DACP（数字音频控制协议）— 按键/遥控命令
├── playback_control.c      # 统一播放控制抽象
├── buttons.c               # 硬件按键输入，带去抖 + 自动重复
└── led.c                   # LED 状态指示

components/
├── dac/                    # 抽象 DAC API（Kconfig 选择实现）
│   └── dac.c               # 分发层 → TAS57xx 或 TAS58xx 驱动
├── dac_tas57xx/            # TI TAS57xx（TAS5756/5754/5751）DAC 驱动，带混合流 DSP
├── dac_tas58xx/            # TI TAS58xx（TAS5825M）DAC 驱动，带片上 DSP + 15 段 EQ
├── display/                # 显示驱动
│   ├── display.c           # 通用显示 API
│   ├── display_st7789.c    # ST7789 TFT，LVGL 9 渲染（ESP32-S3）
│   └── display_stub.c      # 显示禁用时的空操作桩
├── boards/                 # 板级支持（HAL）
│   ├── board_common.c      # 共享板级工具
│   ├── esp32-generic/      # ESP32 通用板初始化
│   ├── esp32s3-generic/    # ESP32-S3 通用板初始化
│   ├── waveshare-esp32p4/  # Waveshare ESP32-P4 板初始化
│   ├── squeezeamp/         # SqueezeAMP（ESP32 + TAS5756）
│   └── esparagus-audio-brick/ # Esparagus Audio Brick（ESP32 + TAS5825M + W5500）
├── spiffs_storage/         # SPIFFS 文件系统挂载（存储网页 + DSP 配置）
├── audio-resampler/        # 基于 sinc 的音频重采样器（44.1→48kHz）
└── board_utils/            # 板级工具
```

## 关键约定

- **CMake/Kconfig**：通过 `CONFIG_` Kconfig 选项选择开发板。DAC 驱动自动选择（`CONFIG_DAC_TAS57XX` 或 `CONFIG_DAC_TAS58XX`）。显示、按键、蓝牙、以太网均由 Kconfig 控制。
- **组件结构**：每个组件有自己的 `CMakeLists.txt`，使用 `idf_component_register()`。
- **Git 子模块**：`u8g2`（OLED 图形库）和 `u8g2-hal-esp-idf`（u8g2 的 ESP-IDF HAL）是子模块 — 克隆时始终使用 `--recursive`。
- **SPIFFS**：`data/` 目录内容烧录到 SPIFFS。`data/www/` = Web UI，`data/hf/` = TAS57xx 的混合流 DSP 二进制文件。
- **音频管线**：AudioReceiver (rtsp) → decoder → AudioBuffer → AudioOutput (I2S/SPDIF/USB)。缓冲流（AAC）使用深度抖动缓冲；实时流（ALAC）使用低延迟 UDP，带提前/滞后时序阈值。
- **AirPlay/蓝牙共存**：运行时互斥。蓝牙连接暂停 AirPlay；断开后恢复。
- **以太网/WiFi 故障切换**：启动时优先以太网；无网线时回退到 WiFi。运行时支持热切换。

## 代码质量

**要求**：ESP-IDF >= 5.5（已针对 v5.5.2 测试）。旧版本可能需要变通方案。

**格式化**：LLVM 风格，2 空格缩进，80 字符列宽限制。参见 `.clang-format`。

**代码检查**：clang-tidy，包含 bugprone、performance、portability 和 readability 检查。参见 `.clang-tidy`。

**Pre-commit 钩子**：自动用 clang-format 格式化已暂存的 C/H 文件，运行 clang-tidy（需要 `build/compile_commands.json`）。通过 `git config core.hooksPath .githooks` 安装。

**CI**（`.github/workflows/ci-release.yml`）：推送到 `main` 或发起 PR 时：
- `format-check`：对所有 C/H 文件进行 clang-format 干运行（排除 `components/u8g2`）
- `lint-check`：对构建输出运行 clang-tidy（需要 ESP-IDF v5.5 工具链）
- `build`：编译 4 个目标配置（esp32s3、squeezeamp-bt、squeezeamp-4m、esparagus-audio-brick-bt）
- `release`：自动创建 GitHub Release，包含合并后的固件 bin 文件（仅推送时触发）

**本地工具**（在 `scripts/` 中）：
```bash
scripts/format.sh          # 格式化所有 C/H 文件（排除 u8g2 子模块）
scripts/lint.sh            # 对所有 C/H 文件运行 clang-tidy
scripts/lint.sh --fix      # 尝试自动修复 clang-tidy 问题
```

**无单元测试**：这是嵌入式固件 — 没有测试框架。需要在硬件上进行手动测试。
