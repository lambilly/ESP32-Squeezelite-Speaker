# ESP32-Squeezelite-Speaker
本项目用于解决Daphile数播系统输出到吸顶音箱的硬件制作教程，主打简单方便，外壳好买，焊接简单，只订制一定PCB转接板

# ESP32-WROVER-E + MAX98357A 刷 Squeezelite-ESP32 完整教程

适用：斐讯 R1 拆机喇叭 / 8Ω 6W 吸顶喇叭 / 无源小音箱
目标：接入 Daphile（LMS）网络音频播放，解决播放中断问题

---

## 一、硬件清单

- **主控模块**：ESP32-WROVER-E N16R8（必须带 PSRAM，16MB Flash + 8MB PSRAM）
- **功放模块**：MAX98357A（I2S 数字功放，内置 DAC，单声道）
- **喇叭**：8Ω 6W 吸顶喇叭（或 4Ω 3W 全频喇叭，4Ω 音量更大）
- **电源**：5V / 2A 优质电源适配器（同时供 ESP32 和功放）
- **滤波电容**：0.1μF 陶瓷电容 ×1、10μF 电解电容 ×1（功放电源滤波）
- **连接线**：杜邦线若干

---

## 二、硬件接线

### 1. ESP32-WROVER-E 与 MAX98357A 的连接

- MAX98357A **VIN** → ESP32 **5V**
- MAX98357A **GND** → ESP32 **GND**（必须共地）
- MAX98357A **LRC** → ESP32 **GPIO 25**
- MAX98357A **BCLK** → ESP32 **GPIO 33**
- MAX98357A **DIN** → ESP32 **GPIO 32**

MAX98357A 的 **SD、GAIN 引脚悬空**，默认使能、默认增益。

### 2. MAX98357A 与喇叭的连接

- 喇叭正极 → **SPK+**
- 喇叭负极 → **SPK-**

⚠️ **严禁将 SPK+ 或 SPK- 接到 GND**。MAX98357A 是 BTL 桥接输出，直接接喇叭两端即可。

### 3. 电源滤波（强烈建议）

在 MAX98357A 的 **VIN 和 GND** 之间并联：

- **0.1μF 陶瓷电容**
- **10μF 电解电容**（正极接 VIN）

这能显著降低底噪、减少断音。

---

## 三、刷机：Squeezelite-ESP32

### 1. 刷机工具

访问官方网页刷机工具（用 Chrome/Edge 浏览器）：

```
https://sle118.github.io/squeezelite-esp32-installer/
```

### 2. 刷机步骤

1. USB 数据线连接 ESP32 与电脑。
2. 点击 **Connect to device**，选择 ESP32 对应的串口。
3. 选择 **Generic/I2S** 平台，固件版本选 **I2S-4MFlash-16**（16 位版本兼容性最好）。
4. 点击安装，等待完成。

### 3. 卡在 Preparing installation 的解决方法

- 按住开发板 **BOOT** 键不放，按一下 **EN/RST** 键后松开，再重新点击连接。
- 更换 USB 数据线和 USB 端口。
- 确保 **CP2102 / CH340** 驱动已安装。

---

## 四、配网与 NVS 配置

### 1. 首次配网

刷机完成后，ESP32 会创建热点：

- SSID：**squeezelite**
- 密码：**squeezelite**

手机/电脑连接后，浏览器访问 **192.168.4.1**，填入你家 Wi-Fi 的 SSID 和密码，保存重启。

### 2. 进入 NVS 编辑器

1. 浏览器访问 ESP32 的 IP（或 `http://squeezelite-xxxx.local`）。
2. 点击 **Credits** 标签页，打开 **shows nvs editor** 开关。
3. 进入 **NVS Editor**。

### 3. 核心参数配置

- **dac_config** → `model=I2S,bck=33,ws=25,do=32`（对应 BCLK、LRC、DIN 引脚）
- **autoexec1** → `squeezelite -o I2S -b 800:2000`（输出方式 + 缓冲区）
- **wifi_ps** → `0`（**关闭 Wi-Fi 省电模式，关键！**）
- **host_name** → `ESP32-TTS`（自定义播放器名称）

修改后点击 **Commit** → 重启设备。

---

## 五、Daphile（LMS）端设置

### 1. 发现播放器

重启 ESP32 后，打开 Daphile 界面（`http://daphile:9000`），右下角播放器选择列表中应能看到 **ESP32-TTS**。

### 2. 降低码率（解决断音的关键手段）

1. 点击 **Settings** → **Player** 选项卡。
2. 下拉列表中选择你的 **ESP32 播放器**。
3. 找到 **Audio** 部分 → **Bitrate Limiting**。
4. 选择 **192 kbps** 或 **128 kbps**。
5. 点击 **Apply** 保存。

### 3. 备选：通过 Squeezelite 参数限制

若 LMS 端无码率限制选项，可在 NVS 的 `autoexec1` 中追加 `-Z` 参数：

```
squeezelite -o I2S -b 800:2000 -Z 192000
```

`-Z 192000` 表示向服务器宣称最大支持 192kbps，强制服务器转码。

---

## 六、断音问题排查清单

按优先级依次排查：

**第 1 步：关闭 Wi-Fi 省电**
NVS 设置 `wifi_ps=0`。

**第 2 步：电源滤波电容**
VIN-GND 间加 0.1μF + 10μF。

**第 3 步：Wi-Fi 信号强度**
Web 界面查看 RSSI，需 > -70dBm。

**第 4 步：更换 Wi-Fi 信道**
路由器设置为 1、6 或 11。

**第 5 步：降低码率**
LMS Bitrate Limiting 设为 192kbps。

**第 6 步：调整缓冲区**
`-b 800:2000`（N16R8 安全上限）。

**第 7 步：限制解码器**
`-c mp3`（仅播放 MP3 时）。

**第 8 步：远离干扰源**
远离微波炉、无绳电话。

**第 9 步：固件更新**
关注 Squeezelite-ESP32 GitHub Releases。

---

## 七、N16R8 内存限制说明

- ESP32 最多直接寻址约 **4MB PSRAM**。
- `-b 800:2000` = 2800KB，安全稳定。
- `-b 1000:3000` = 4000KB，**超出实际可用内存**，会导致 Squeezelite 无法启动，Daphile 找不到设备。
- **建议缓冲区上限：800:2000**，不要再增大。

---

## 八、常见问题速查

**Q1：Daphile 找不到 ESP32？**

- 确认 ESP32 与 Daphile 在同一局域网。
- 检查 `dac_config` 引脚是否与实际接线一致。
- 尝试在 `autoexec1` 中手动指定服务器：`-s 192.168.1.100`（替换为达菲 IP）。

**Q2：只有电流声、没有音乐？**

- 确认 `dac_config` 中 `model=I2S`（不是 `internal`）。
- 检查 LRC / BCLK / DIN 三根线是否接对。
- 确认 GND 共地。

**Q3：播放 3 分钟断一次？**

- 优先执行：`wifi_ps=0` + 加装电源滤波电容 + 降低码率至 192kbps。

**Q4：音量太小？**

- 换用 **4Ω** 喇叭，MAX98357A 在 5V 下驱动 4Ω 可输出 3.2W。
- 确认供电为 5V（不是 3.3V）。

---

## 九、参考链接

- Squeezelite-ESP32 项目：`https://github.com/sle118/squeezelite-esp32`
- 网页刷机工具：`https://sle118.github.io/squeezelite-esp32-installer/`
- MAX98357A 数据手册：Adafruit / Maxim 官网

---

**教程完**
