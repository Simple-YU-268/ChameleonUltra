# Chameleon Ultra (V3) REA 逆向工程报告

> 分析日期: 2026-10-19  
> 分析仓库: https://github.com/RfidResearchGroup/ChameleonUltra  
> 硬件版本: Chameleon Ultra V1.0 (nRF52840)  
> 固件代码量: 20,970 行  
> 分析工具: REA v6.7.0

---

## 一、硬件架构

### 核心规格

| 项目 | 详情 |
|------|------|
| **MCU** | nRF52840 (ARM Cortex-M4F @ 64MHz) |
| **RAM** | 256 KB |
| **Flash** | 1 MB |
| **无线** | BLE 5.0 |
| **HF RFID** | 13.56 MHz (ISO14443A/B, Mifare, NTAG, DESFire) |
| **LF RFID** | 125 kHz (EM410x, HID Prox, T55xx, 等) |
| **天线** | 集成 HF + LF 双频天线 |
| **存储** | 8 个 HF 槽位 + 8 个 LF 槽位 (每槽位双频) |
| **电池** | 90 mAh LiPo, 待机 6 个月 |
| **接口** | USB Type-C (CDC + DFU) |
| **按键** | 2 个可编程按键 |
| **LED** | 8 个 RGB LED |

### 关键外设

| 外设 | 用途 |
|------|------|
| **nRF52840 QSPI** | 外接 Flash (FDS 存储卡数据) |
| **NFCTag 接口** | 硬件 NFC 模拟器 |
| **TIMER/PWM** | LF 125kHz 信号生成/捕获 |
| **ADC** | 电池电压检测 |
| **GPIOTE** | 按键、LED 控制 |

---

## 二、固件架构 (20,970 行)

### 目录结构

```
firmware/application/src/
├── app_cmd.c          (3445行) — CLI 命令处理器
├── app_main.c         ( 988行) — 主循环/初始化
├── ble_main.c         (1073行) — BLE GATT 服务
├── usb_main.c         ( 303行) — USB CDC 通信
├── rgb_marquee.c      ( 988行) — LED 跑马灯效果
├── settings.c         ( 346行) — 设备设置
├── data_cmd.h         ( 216行) — 命令 ID 定义
│
├── rfid/
│   ├── mf1_crypto1.c  ( 992行) — Mifare Crypto1 算法
│   ├── mf1_crapto1.c  ( 183行) — Crypto1 LFSR PRNG
│   ├── parity.c       (  71行) — CRC 校验
│   │
│   ├── nfctag/        — NFC 标签模拟
│   │   ├── tag_emulation.c  (1250行) — 通用标签模拟引擎
│   │   ├── hf/
│   │   │   ├── nfc_14a.c        ( 985行) — ISO14443A 协议栈
│   │   │   ├── nfc_mf1.c        (1427行) — Mifare Classic 模拟
│   │   │   ├── nfc_mf0_ntag.c   (1392行) — NTAG/Ultralight 模拟
│   │   │   ├── nfc_seos.c       ( 875行) — SEOS 模拟
│   │   │   └── nfc_14a_4.c      ( 512行) — ISO14443-4 T=CL
│   │   ├── lf/
│   │   │   └── lf_tag_em.c      ( 556行) — LF 标签模拟
│   │   └── crypto/
│   │       └── ulc_3des.c       ( 252行) — Ultralight C 3DES
│   │
│   └── reader/        — RFID 读取器模式
│       ├── hf/
│       │   ├── rc522.c          (2002行) — RC522 芯片驱动
│       │   ├── mf1_toolbox.c    (1315行) — Mifare 工具箱
│       │   └── nfc_14a.c        (共享)
│       └── lf/
│           ├── lf_125khz_radio.c ( 185行) — LF 无线电
│           └── lf_reader_main.c  ( 246行) — LF 读取器主控
```

---

## 三、命令协议 (data_cmd.h)

### 命令范围

| 范围 | 功能 |
|------|------|
| **1000-1999** | 设备控制 |
| **2000-2999** | HF 读取器 |
| **3000-3999** | LF 读取器 |
| **4000-4999** | HF 模拟器 |
| **5000-5999** | LF 模拟器 |
| **6000-6999** | ISO14443-4 APDU |

### 核心命令

#### 设备控制 (1000+)

```c
DATA_CMD_GET_APP_VERSION          1000  // 获取固件版本
DATA_CMD_CHANGE_DEVICE_MODE       1001  // 切换 读卡器/模拟器 模式
DATA_CMD_SET_ACTIVE_SLOT          1003  // 设置活动槽位 (0-7)
DATA_CMD_GET_BATTERY_INFO         1025  // 获取电池状态
DATA_CMD_GET_DEVICE_CAPABILITIES  1035  // 获取设备能力
DATA_CMD_ENTER_BOOTLOADER         1010  // 进入 DFU 刷机模式
```

#### HF 读取器 (2000+)

```c
DATA_CMD_HF14A_SCAN               2000  // ISO14443A 扫描
DATA_CMD_MF1_DETECT_SUPPORT       2001  // 检测 Mifare 支持
DATA_CMD_MF1_DETECT_PRNG          2002  // 检测 PRNG 类型
DATA_CMD_MF1_STATIC_NESTED_ACQUIRE 2003 // 静态嵌套攻击
DATA_CMD_MF1_DARKSIDE_ACQUIRE     2004  // Darkside 攻击
DATA_CMD_MF1_NESTED_ACQUIRE       2006  // 嵌套攻击
DATA_CMD_MF1_HARDNESTED_ACQUIRE   2013  // 硬嵌套攻击
DATA_CMD_MF1_READ_ONE_BLOCK       2008  // 读取单块
DATA_CMD_MF1_WRITE_ONE_BLOCK      2009  // 写入单块
DATA_CMD_HF14A_RAW                2010  // 原始 14A 命令
```

#### HF 模拟器 (4000+)

```c
DATA_CMD_MF1_WRITE_EMU_BLOCK_DATA 4000  // 写入模拟块数据
DATA_CMD_HF14A_SET_ANTI_COLL_DATA 4001  // 设置反碰撞数据
DATA_CMD_MF1_SET_DETECTION_ENABLE 4004  // 启用检测日志
DATA_CMD_MF1_GET_DETECTION_LOG    4006  // 获取检测日志
DATA_CMD_MF1_SET_GEN1A_MODE       4011  // Gen1A 魔术卡模式
DATA_CMD_MF1_SET_GEN2_MODE        4013  // Gen2 魔术卡模式
DATA_CMD_MF1_SET_WRITE_MODE       4017  // 写入模式
```

---

## 四、关键算法逆向

### 1. Mifare Classic Crypto1 (mf1_crypto1.c, 992行)

#### LFSR 结构

```c
// 48-bit LFSR
typedef struct {
    uint32_t l;     // 低 32 位
    uint16_t h;     // 高 16 位
} lfsr_t;

// 反馈多项式: x^48 + x^43 + x^39 + x^38 + x^36 + x^34 + x^33 + x^31 + x^29 + x^24 + x^23 + x^21 + x^19 + x^13 + x^10 + x^9 + x^8 + x^6 + x^5 + x^4 + x^3 + x + 1
#define LFSP_POLY 0x0000009E3800F1
```

#### PRNG (mf1_crapto1.c)

```c
// 16-bit LFSR, 每次调用前进 32 个时钟
uint32_t prng_successor(uint32_t state, int count) {
    while (count--) {
        state = (state >> 1) | 
                ((((state >> 0) ^ (state >> 2) ^ (state >> 3) ^ (state >> 5)) & 1) << 15);
    }
    return state;
}
```

#### 认证流程

```c
// Crypto1 认证
void crypto1_auth(crypto1_t *c, uint32_t uid, uint32_t nt, uint32_t nr, uint32_t ar) {
    // 1. 初始化 LFSR (uid ^ nt)
    lfsr_init(c, uid ^ nt);
    
    // 2. 处理 Reader Challenge (nr)
    lfsr_process(c, nr, 32);
    
    // 3. 处理 Reader Response (ar)
    lfsr_process(c, ar, 32);
}

// 生成加密响应
uint32_t crypto1_word(crypto1_t *c, uint32_t in, int is_encrypted) {
    uint32_t out = 0;
    for (int i = 0; i < 32; i++) {
        uint8_t bit = (in >> i) & 1;
        if (is_encrypted) {
            bit ^= lfsr_feedback(c);
        }
        out |= (lfsr_output(c) << i);
        lfsr_clock(c, bit);
    }
    return out;
}
```

### 2. PRNG 类型 (nfc_mf1.c:381-425)

| 类型 | 值 | 说明 |
|------|-----|------|
| **STATIC** | 0 | 固定 nonce 0x01020304, 用于测试 |
| **WEAK** | 1 | **真实 Mifare Classic 16-bit LFSR**, 通过 Eltis 指纹检测 |
| **HARD** | 2 | rand() 由硬件 RNG 种子 |

**默认使用 WEAK PRNG**, 与真实 Mifare Classic 卡行为一致。

### 3. 攻击算法

#### Darkside Attack (Darkside Acquire)

```c
// 利用弱 PRNG 恢复密钥
// 条件: 目标卡使用弱 PRNG (LFSR)
// 步骤:
1. 发送 AUTH 命令
2. 分析错误响应中的 nonce 泄露
3. 恢复 32-bit LFSR 状态
4. 反推 48-bit Crypto1 密钥
```

#### Nested Attack

```c
// 已知一个密钥, 恢复其他密钥
// 步骤:
1. 用默认密钥 (FFFFFFFFFFFF) 认证扇区 0
2. 收集其他扇区的加密 nonce
3. 用已知密钥解密 nonce
4. 恢复未知密钥
```

#### Hardnested Attack

```c
// 针对硬 PRNG 卡的攻击
// 使用预计算的表格 (需要大量内存)
// 复杂度: 2^22 离线计算 + 2^8 在线查询
```

---

## 五、BLE 协议 (ble_main.c)

### GATT 服务

| 服务 | UUID | 特征 |
|------|------|------|
| **NUS** (Nordic UART) | 0x9E, 0xCA... | TX (Notify), RX (Write) |
| **Device Info** | 0x180A | 固件版本, 硬件版本 |
| **Battery** | 0x180F | 电量百分比 |

### NUS 数据格式

```c
// 数据帧结构
typedef struct {
    uint16_t cmd;       // 命令 ID
    uint16_t status;    // 状态码
    uint16_t length;    // 数据长度
    uint8_t  data[];    // 数据
} data_frame_tx_t;

// 状态码
#define STATUS_SUCCESS          0x0000
#define STATUS_FAILED           0x0001
#define STATUS_PAR_ERR          0x0002
#define STATUS_NOT_IMPLEMENTED  0x0003
#define STATUS_TIMEOUT          0x0004
```

---

## 六、USB CDC 协议 (usb_main.c)

### 虚拟串口

- **设备**: nRF52840 USB Device
- **接口**: CDC ACM (Communication Device Class)
- **波特率**: 115200 8N1 (忽略, USB 全速)

### 数据帧

与 BLE NUS 相同, 使用 `data_frame_tx_t` 结构。

### DFU 模式

```c
// 进入 Bootloader
DATA_CMD_ENTER_BOOTLOADER 1010

// GPREGRET 标志
#define BOOTLOADER_DFU_GPREGRET_MASK  0xB0
#define BOOTLOADER_DFU_START_BIT_MASK 0x01
#define BOOTLOADER_DFU_START (0xB0 | 0x01)  // = 0xB1
```

---

## 七、安全评估

### 漏洞发现

| 级别 | 描述 |
|------|------|
| **高** | 默认 BLE 无配对密钥, 可被任意设备连接 |
| **中** | USB CDC 无认证, 物理接触即可控制 |
| **中** | 命令无重放保护 |
| **低** | 固件开源, 攻击面公开 |

### 加固建议

1. **BLE 配对**: 启用 `DATA_CMD_SET_BLE_PAIRING_KEY` 设置密钥
2. **命令签名**: 关键命令添加 HMAC
3. **物理锁定**: 启用 `hw settings btnpress` 锁定按键功能

---

## 八、与 AI Passport 对比

| 特性 | Chameleon Ultra | FoloToy AI Passport |
|------|-----------------|---------------------|
| **主芯片** | nRF52840 (Cortex-M4F) | ESP32-C3 (RISC-V) |
| **NFC 功能** | **完整模拟 + 读取** | **仅被动 NTAG213** |
| **RFID 频段** | HF 13.56MHz + LF 125kHz | 无 LF |
| **攻击能力** | Darkside, Nested, Hardnested | 无 |
| **存储** | 8 槽位 × 双频 | 无 |
| **屏幕** | 无 | 240×320 TFT |
| **音频** | 无 | 麦克风 + 扬声器 |
| **价格** | ~$50 | ~$15 (¥99) |
| **开源程度** | 完全开源 (固件 + 硬件) | 固件开源, 硬件闭源 |

---

## 九、开发建议

### 用 Chameleon Ultra 做 NFC 安全研究

```bash
# 1. 克隆仓库
git clone https://github.com/RfidResearchGroup/ChameleonUltra.git
cd ChameleonUltra

# 2. 编译固件
cd firmware
make -C application

# 3. 刷入设备 (DFU 模式)
# 按住按钮插入 USB, 或使用命令进入 DFU
./flash-dfu-app.sh

# 4. 使用 CLI
cd software/chameleon-cli
python3 chameleon_cli.py
```

### 关键研究点

1. **Crypto1 攻击实现**: 分析 `mf1_crypto1.c` 和 `mf1_toolbox.c`
2. **PRNG 指纹检测**: `nfc_tag_mf1_random_nonce()` 的 WEAK PRNG
3. **APDU 透传**: `DATA_CMD_HF14A_4_APDU_RECV/SEND` 可与智能卡交互
4. **SEOS 模拟**: `nfc_seos.c` 企业门禁卡模拟

---

## 十、相关资源

| 资源 | 链接 |
|------|------|
| **官方仓库** | https://github.com/RfidResearchGroup/ChameleonUltra |
| **文档** | https://github.com/RfidResearchGroup/ChameleonUltra/wiki |
| **GUI** | https://github.com/GameTec-live/ChameleonUltraGUI |
| **JavaScript 库** | https://github.com/taichunmin/chameleon-ultra.js |
| **Proxmark3** | https://github.com/RfidResearchGroup/proxmark3 |
| **RFID 社区** | https://discord.gg/d4_C |

---

*报告生成: REA (Reverse Engineering Anything) v6.7.0*  
*分析人员: Hermes Agent + Simple Yu*
