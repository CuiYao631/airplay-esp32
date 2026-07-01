# 显示组件

通过统一的 `display_init()` API 支持两类显示屏：

| 驱动 | 显示屏 | 技术栈 |
|--------|---------|-------|
| OLED (u8g2) | SSD1306、SH1106、SSD1309、SH1107 — 128×64 或 128×32 | u8g2 + I2C/SPI |
| ST7789 TFT | 1.9" IPS 320×170 横屏 | esp_lcd + LVGL 9 + esp_lvgl_port |

当显示屏禁用时（`CONFIG_DISPLAY_ENABLED=n`），仅编译 `display_stub.c` — 零运行时开销。

---

## 硬件要求

### OLED (u8g2)

兼容所有支持的目标平台 — ESP32、ESP32-S3、WROOM、Wrover、SqueezeAMP、Esparagus Audio Brick。无特殊内存要求。

### ST7789 TFT (LVGL 9)

**需要带 PSRAM 的 ESP32-S3。** 此驱动不适用于 ESP32（原版），原因如下：

| 约束条件 | ESP32 (WROOM/Wrover) | ESP32-S3 N16R8 |
|---|---|---|
| 内部 SRAM | 520 KB（与 WiFi + 音频共享） | 512 KB + 8 MB PSRAM |
| PSRAM | 无 (WROOM) / 4 MB (Wrover) | 8 MB |
| Flash | 4 MB | 16 MB |

LVGL 9 + AirPlay 音频管线 + WiFi 三者合计超出了 ESP32 的内部 SRAM 预算。即使有 PSRAM，Wrover 的 4 MB Flash 在计入音频栈、SPIFFS 分区和 LVGL 资源后也过于紧张。

`idf_component.yml` 强制执行此限制 — LVGL 和 `esp_lvgl_port` 仅在 `target == esp32s3` 时声明为依赖。非 S3 构建完全不受影响：托管组件不会被下载、不会被编译，OLED 驱动继续正常工作。

**测试平台：** ESP32-S3 N16R8（16 MB Flash，8 MB PSRAM），IDF 5.5.3。

---

## OLED 显示屏 (u8g2)

显示曲目标题、艺术家、专辑、进度条和播放时间。长文本自动滚动。128×32 面板采用紧凑的双行布局。

### 启用方法

```bash
idf.py menuconfig
# AirPlay Receiver → Display Configuration → Enable display
# 选择驱动：SSD1306 / SH1106 / SSD1309
# 选择总线：I2C 或 SPI
```

### 默认接线 (I2C)

| OLED 引脚 | ESP32 GPIO |
|----------|------------|
| SDA      | 21         |
| SCL      | 22         |
| VCC      | 3.3V       |
| GND      | GND        |

默认 I2C 地址：`0x3C`。如果显示屏使用 `0x3D`，请在 menuconfig 中修改。

---

## ST7789 TFT 显示屏

全彩显示屏，在位图背景上显示曲目元数据，带进度条和已播放/剩余时间。需要 ESP32-S3。

### 启用方法

```bash
idf.py menuconfig
# AirPlay Receiver → Display Configuration → Enable display
# 选择驱动：ST7789 TFT (320×170 landscape)
```

或在 `sdkconfig.defaults.esp32s3` 中添加：

```
CONFIG_DISPLAY_ENABLED=y
CONFIG_DISPLAY_DRIVER_ST7789=y
CONFIG_DISPLAY_SPI_CLK=18
CONFIG_DISPLAY_SPI_MOSI=17
CONFIG_DISPLAY_SPI_CS=15
CONFIG_DISPLAY_SPI_DC=16
CONFIG_DISPLAY_SPI_RST=21
CONFIG_DISPLAY_BL_GPIO=38
```

### 接线 (ESP32-S3)

| 显示屏引脚 | ESP32-S3 GPIO | 功能              |
|-------------|---------------|-----------------------|
| SCL / CLK   | 18            | SPI 时钟             |
| SDA / MOSI  | 17            | SPI 数据              |
| CS          | 15            | 片选           |
| DC / RS     | 16            | 数据/命令选择 |
| RES / RST   | 21            | 复位                 |
| BLK / BL    | 38            | 背光             |
| VCC         | 3.3V          | 电源                 |
| GND         | GND           | 接地                |

---

## 背景图片

ST7789 驱动在启动时从 SPIFFS 加载全屏背景图片
（`/spiffs/bg/background.bin`）并渲染到 PSRAM 中。所有 UI 控件（文字、
进度条）绘制在其上方。如果没有文件，屏幕默认为纯黑色 — 所有控件仍能正常渲染。

### 更换背景

1. 设计图片并导出为 PNG（任意尺寸 — 会自动缩放）
2. 从项目根目录运行转换脚本：
   ```bash
   python3 components/display/make_background.py <source.png> [brightness]
   ```
   亮度范围：`0.4`–`0.6`。从 `0.5` 开始 — ST7789 背光渲染
   比显示器亮得多。根据实际效果调整。
3. 脚本会生成 `data/bg/background.bin`。使用以下两种方法之一烧录到设备。

### 烧录背景

**方案 A — 完整 SPIFFS 烧录（串口，首次烧录或固件更新后）：**
```bash
idf.py flash   # 一步完成固件 + SPIFFS 镜像烧录
```

**方案 B — OTA 上传（WiFi，首次烧录后无需 USB）：**
```bash
curl -X POST "http://<device-ip>/api/fs/upload?path=/spiffs/bg/background.bin" \
     --data-binary @data/bg/background.bin
```
然后重启设备 — 新背景在下次启动时加载。

> **注意：** 没有用于背景上传的 Web UI。设备的 Web
> 界面仅用于 WiFi 设置和设备配置。背景
> 更新通过上述 `curl` 命令完成。

### 色深限制

ST7789 运行在 **RGB565**（16 位色彩）模式：

| 通道 | 源 | 显示 |
|---------|--------|---------|
| 红     | 8 位（256 级） | 5 位（32 级） |
| 绿   | 8 位（256 级） | 6 位（64 级） |
| 蓝    | 8 位（256 级） | 5 位（32 级） |

这会导致**暗部渐变出现可见色带** — 这是硬件层面的根本限制，没有完全的软件解决方案。设计背景时请注意：
- 避免细微的暗色到暗色渐变
- 高对比度和分明的色彩区域渲染效果好
- 转换脚本应用了 Floyd-Steinberg 抖动算法，略有改善

### 为什么不使用 LVGL 在线转换器？

LVGL v9 在线转换器（lvgl.io/tools/imageconverter）没有暴露
RGB565 的字节交换选项。`make_background.py` 提供完全控制，并生成
已在此面板上验证正确的 little-endian 输出。

---

## 实现说明 (ST7789 + esp_lvgl_port)

以下是通过测试发现的非显而易见的集成要求。它们
适用于任何使用此显示屏配合 ESP-IDF + LVGL 9 + `esp_lvgl_port` 的项目。

### 1. 旋转必须在 `lvgl_port_add_disp()` 之后应用

`esp_lvgl_port` 在 `lvgl_port_add_disp()` 内部会重置 ST7789 的 MADCTL 寄存器，
清除之前设置的任何旋转。务必在**该调用之后**应用
`swap_xy`、`mirror` 和 `set_gap`。

```c
s_lvgl_disp = lvgl_port_add_disp(&disp_cfg);

// 旋转在之后 — 而非之前
ESP_ERROR_CHECK(esp_lcd_panel_swap_xy(panel_handle, true));
ESP_ERROR_CHECK(esp_lcd_panel_mirror(panel_handle, true, false));
ESP_ERROR_CHECK(esp_lcd_panel_set_gap(panel_handle, 0, 35));
```

此规则适用于所有 ESP32 变体。

### 2. LVGL port 任务必须固定到 Core 0（双核构建）

默认的 `task_affinity = -1` 允许 LVGL 任务迁移到 Core 1。
在双核构建中，音频运行在 Core 1（优先级 7），这会导致渐进式
音频缓冲区背压 — 延迟从约 1800ms 攀升至 10000ms 且无法恢复，
最终导致流错位。

**不要**使用 `ESP_LVGL_PORT_INIT_CONFIG()` — 它默认为 `-1`。显式设置：

```c
const lvgl_port_cfg_t lvgl_cfg = {
    .task_priority     = 4,
    .task_stack        = 6144,
    .task_affinity     = 0,   // Core 0 — 保持 Core 1 空闲给音频
    .task_max_sleep_ms = 500,
    .timer_period_ms   = 5,
};
```

规则：所有显示任务在 Core 0，所有音频任务在 Core 1。

### 3. 绘制缓冲区必须在支持 DMA 的内部 SRAM 中

在此版本的 `esp_lvgl_port` 中，flush 回调将绘制缓冲区指针
**直接**传递给 `esp_lcd_panel_draw_bitmap` — `trans_size` 无效。
如果缓冲区在 PSRAM 中（`buff_spiram=true`），SPI 主控制器会在运行时
从内部堆分配私有 DMA 缓冲区。在内存压力下这会失败，
导致 flush 死锁和看门狗复位。

**症状：** 监视器中显示 `Failed to allocate priv TX buffer`，随后
taskLVGL 看门狗触发。

```c
.flags = {
    .buff_dma    = true,   // 内部支持 DMA 的 SRAM
    .buff_spiram = false,  // 非 PSRAM
    .swap_bytes  = true,
},
```

10 条绘制线（双缓冲）的成本：**12,800 字节**内部 SRAM。
设置 `trans_size = 0` — 此代码路径中不使用它。

---

## 诊断：色条测试

如果背景图片颜色看起来不对（褪色、通道交换
或颜色怪异），在调试其他问题之前先运行此测试确认字节顺序：

```python
from PIL import Image, ImageDraw

bars = [
    (255,255,255), (255,255,0), (0,255,255), (0,255,0),
    (255,0,255),   (255,0,0),   (0,0,255),
]
img = Image.new('RGB', (320, 170))
draw = ImageDraw.Draw(img)
bar_w = 320 // len(bars)
for i, color in enumerate(bars):
    draw.rectangle([i*bar_w, 0, i*bar_w+bar_w, 170], fill=color)
# 使用 make_background.py 或相同的字节转换循环进行转换
```

屏幕上预期显示（从左到右）：
**白 → 黄 → 青 → 绿 → 品红 → 红 → 蓝**

如果渲染正确，字节顺序已确认。如果通道看起来不对，
问题出在转换环节，而非显示驱动。

---

## macOS AirPlay 发现

macOS 会缓存 AirPlay 设备状态。如果 ESP32 在连接时反复崩溃
（例如上述 DMA 看门狗），macOS 看到设备反复消失后会变得
保守 — 提供或连接设备的速度变慢。

如果重新烧录后 AirPlay 发现似乎变慢或停滞，首先刷新 macOS
的 mDNS 缓存作为诊断第一步：

```bash
sudo dscacheutil -flushcache
```

这可以排除 macOS 持有过期状态的可能性，然后再假设问题出在
固件端。固件稳定后，连接几乎是即时的。

---

## 使用 Claude 构建

此组件是与 [Claude](https://claude.ai)（Anthropic）协作开发的。Claude 贡献了驱动实现、LVGL 9 迁移、上述非显而易见集成问题的调试、背景图片工具以及本文档。

所有由 Claude 直接提交的 commit 均包含以下共同作者标签：
```
Co-authored-by: Claude (Anthropic) <claude@anthropic.com>
```
