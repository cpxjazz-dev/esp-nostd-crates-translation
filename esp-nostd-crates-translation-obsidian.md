---
title: ESP32-C3 Rust no_std 常用 crate 文档翻译
tags: [esp32, rust, no_std, esp-hal, esp32c3]
created: 2026-11-05
updated: 2026-11-05
version: v2
source: crates.io / docs.rs / Rust on ESP Book
---

# ESP32-C3 Rust `no_std` 常用 crate 文档翻译（v2 · C3 专属查阅版）

> 整理时间：2026 年 11 月 5 日（v2 扩充版）
> 面向读者：已经会写 Rust、正在学 ESP32-C3 + Rust `no_std` 的开发者
> 本文不是官方翻译。原文以 crates.io / docs.rs / 《Rust on ESP Book》为准，
> 版本号变化很快，落到自己项目前请先到 crates.io 核对最新版再固定版本。

## v2 改了什么（相对 v1）

v1 只讲了 8 个 crate 的“是什么”，三个缺口这次补上：

1. **API 表**：每个 C3 外设 / 每个 crate 给出模块路径、主类型、关键方法名，
   能直接按表查，不用再翻英文原文找入口。
2. **示例索引**：只列已核实文件名的官方示例，每个一句话说明它演示什么，
   没核实文件名的不编造，教你用 `//% CHIPS:` 行和 `Cargo.toml` 自己判定。
3. **异步专章**：`esp-rtos` 启动、`#[esp_rtos::main]`、Embassy 任务、
   `Timer` / `Channel` / `Mutex` / `select`、Wi-Fi + `embassy-net` 的
   完整异步链路，不再只有一句“配合 Embassy 使用”。

同时 v2 **只留 C3 有关的部分**：删掉了 v1 里按 C6 写的示例参数
（例如 `esp-storage` 示例里的 `esp32c6`），删掉了 C3 上不存在的外设
（DAC、Ethernet、USB-OTG、PSRAM 等），并纠正了 v1 遗留的两处错误：
`esp-nvs` 的真实 API 不是 `NvsPartition::new` / `get_i32` / `commit`
（那是 ESP-IDF C API 和 `esp-idf-svc` 的写法），默认 NVS 分区也不是
示例里的 `0x390000 / 0x32000`（见第 7 节）。

## 版本警告：先对版本，再抄代码

2026 年 11 月核查时看到的版本线：`esp-hal` 1.2.2、`esp-radio` 1.0.0-beta.1、
`esp-rtos` 0.3.0、`esp-nvs` 0.5.0、`esp-println` 0.18.0、`esp-alloc` 0.9.0、
`esp-storage` 0.8.x。**`esp-radio` 0.17 → 0.18 改了 Wi-Fi API，
`esp-hal` 1.x 各小版本的外设稳定性也在变**，不同版本同一个东西可能
叫两个名字，例如：

- Wi-Fi 接口：旧版 `interfaces.sta` / `WifiDevice`，新版 `interfaces.station` / `Interface`
- Wi-Fi 模式：旧版 `ModeConfig::Client` / `ModeConfig::AccessPoint`，
  新版 `Config::Station` / `Config::AccessPoint`
- `esp_rtos::start`：旧版单参数 `start(timg0.timer0)`，当前 C3 实例一律双参数
  `start(timg0.timer0, sw_int.software_interrupt0)`

**规则：成套升级，不要只升一个 crate；抄代码前先确认你锁定的版本文档。
本文凡是写“版本不同，两个名字”的地方，都是这个原因。**

## 翻译原则：哪些翻，哪些不翻

| 内容 | 处理方式 | 原因 |
|---|---|---|
| 说明性文字、用途、注意事项、步骤 | 翻译成中文 | 这是本文的主要价值 |
| crate 名、函数名、类型名、宏名、Cargo feature 名 | 保持原文，例如 `esp_hal::init`、`heap_allocator!`、`esp32c3` | 改了就编不过，查文档也对不上 |
| `Cargo.toml`、代码片段、命令行 | 保持原文，只在代码外用中文注释 | 直接可用 |
| 错误信息、日志输出、专有名词 | 保持原文，例如 `Out of capacity`、`backtrace`、`PSRAM` | 搜索和排错时用原文最有效 |
| 每个 API 的完整参数表 | 只翻译“怎么选、怎么用”，给关键方法表；完整签名以 docs.rs 为准 | docs.rs 的 API 文档是按芯片生成的，量太大且随版本变 |

一句话总结：**讲清楚用中文讲，动手用原文动手。**

---

## C3 硬件快照（查 API 之前先对上这张表）

| 项目 | C3 的值 |
|---|---|
| CPU | 单核 RISC-V，最高 160 MHz，目标 `riscv32imc-unknown-none-elf`，稳定 Rust 可用 |
| 内存 | SRAM 400 KB，ROM 384 KB；Flash 外置，常见 4 MB，最高 16 MB |
| PSRAM | **没有**。`psram_allocator!`、`psram` feature 都不要用在 C3 上 |
| 无线 | Wi-Fi 4（802.11 b/g/n）+ BLE 5.0 LE，**无经典蓝牙，无 IEEE 802.15.4**（Zigbee / Thread 要换 C6 / H2） |
| USB | 只有 USB-Serial-JTAG（固定 CDC-ACM + JTAG，占 GPIO18/19），**无 USB-OTG** |
| 没有的外设 | DAC、Ethernet、触摸 Touch、摄像头、MCPWM、PCNT、ULP、SDMMC host、I3C、LP UART/I2C/SPI、SPI3、I2C1、UART2 |
| 只有 1 个的 | SPI2（SPI0/SPI1 接 Flash）、I2C0、I2S 1 个、TWAI 1 个 |
| 只有 2 个的 | UART0 + UART1、TIMG0 + TIMG1 |

### C3 引脚坑（写 Blink / 接线之前先看）

- **Flash 占 GPIO11–GPIO17，永远别用。**
- **Strapping 脚 GPIO2 / GPIO8 / GPIO9**：复位时采样定启动模式。
  GPIO9 拉低进下载模式（多为 BOOT 键），GPIO8 选 Flash 电压。
  复位后可当普通 GPIO，但外接电路别在复位时把它们拉错电平。
- **USB-Serial-JTAG 占 GPIO18（D−）/ GPIO19（D+）**：原生 USB-C 烧录 /
  打印 / 调试，不用 CH340；烧录或看日志时别把这两脚当普通 GPIO。
- **UART0（GPIO20 RX / GPIO21 TX）是启动日志口**，别另作他用。
- **JTAG 脚 GPIO4–GPIO7**：不用外接 JTAG 时可当普通 GPIO。
- **ADC2（GPIO5）与 Wi-Fi 冲突**：开 Wi-Fi 后 ADC2 读数不可靠；
  做 Wi-Fi 传感器优先用 ADC1（GPIO0–GPIO4），其中 GPIO2 是 strapping 脚，
  首选 GPIO0/1/3/4。
- **LED 引脚因板而异，没有统一答案**：DevKitC-02 是 GPIO8 的可寻址 RGB LED；
  SuperMini 常见 GPIO8 普通蓝色 LED（低电平亮）；XIAO 等板又不同。
  GPIO8 当普通 `Output` 只对普通 LED 板有效，WS2812 板必须用 RMT +
  `smart-leds` 类驱动，先看板子丝印 / 原理图再写 Blink。
- **GPIO 全是 3.3 V，不耐 5 V**：5 V 信号要分压 / 电平转换。
- I2C / SPI / UART 多数可经 GPIO 矩阵任意改脚，但 ADC、USB、JTAG、
  Flash-SPI 是固定脚。

---

## 生态总览（先建立全图，再看单个 crate）

原文（《Rust on ESP Book · Ancillary Crates Overview》）：

> The core crate that ties all work with Espressif chips in Rust is the `esp-hal` crate.
> Through it, you will be able to perform basic initialization of the chip,
> as well as access drivers for the peripherals available on the chip.

中文：把所有 Espressif 芯片上的 Rust 工作串起来的核心 crate 是 `esp-hal`。
通过它可以完成芯片的基本初始化，并访问芯片上各个外设的驱动。

原文：

> Furthermore, you may want to use more advanced functionality of the chip.
> For example, network and connectivity. This part of the ecosystem is the
> responsibility of `esp-radio`.

中文：如果你还需要更高级的功能，例如网络和连接，这部分由 `esp-radio` 负责。
它把各芯片上可用的通信协议驱动合在一起：Wi-Fi、BLE、ESP-NOW，
以及底层的 IEEE 802.15.4（**C3 没有 802.15.4 射频，这一项与 C3 无关**）。

原文：

> For more advanced work with chip memory and to use collections from the `alloc`
> crate in `no_std` that require heap allocation, you are welcome to use `esp-alloc`.

中文：如果你要更深入地使用芯片内存，或者在 `no_std` 下使用 `alloc` 里
需要堆分配的集合（例如 `Vec`、`Box`、`String`），请使用 `esp-alloc`。

分工速查：

| 层 | crate | 一句话 |
|---|---|---|
| 硬件抽象 | `esp-hal` | 裸机 HAL，外设驱动的入口 |
| 堆内存 | `esp-alloc` | `no_std` 下的全局堆分配器 |
| 出错时怎么办 | `esp-backtrace` | panic / 异常时打印调用栈 |
| 打印和日志 | `esp-println` | `println!`、`log` 的串口后端 |
| 掉电不丢的数据 | `esp-storage` | Flash 的底层读写，配合 NVS / 文件系统用 |
| 掉电不丢的键值 | `esp-nvs` | ESP-IDF 兼容的 NVS 键值存储（v1 漏掉，本版补全） |
| 无线 | `esp-radio` | Wi-Fi、BLE、ESP-NOW（原 `esp-wifi` 的后继；C3 无 802.15.4） |
| 调度 | `esp-rtos` | 给 `esp-radio` 和 Embassy 提供调度器 |
| TCP/IP | `embassy-net` | Embassy 的 TCP/IP 栈，Wi-Fi 联网必备（v1 漏掉，本版补全） |
| 异步执行器 | `embassy-executor` / `embassy-time` / `embassy-sync` | 任务、定时、通道、锁（本版补全） |
| 启动描述 | `esp-bootloader-esp-idf` | 写入 ESP-IDF 二级 bootloader 需要的应用描述，含 OTA 支持 |

注意：`esp-hal` 本身标注为 Stable（但个别外设驱动仍可能标 `unstable`），
其余多数辅助 crate 在官方表里仍标注为 Unstable，升级时要按 SemVer 之外
的变更预期来对待，即小版本也可能有破坏性变化。**`esp-hal` 1.0 的稳定集
很小：只有 GPIO、UART、SPI master、I2C master 是稳定 API，其余几乎都在
`unstable` feature 后面。**

---

## 最小骨架：一个 ESP32-C3 工程长什么样

### 阻塞版骨架

```toml
# Cargo.toml（节选，版本号请以 crates.io 最新版为准）
[dependencies]
esp-hal = { version = "1", features = ["esp32c3", "unstable"] }
esp-alloc = { version = "0.9", features = ["esp32c3"] }
esp-backtrace = { version = "0.18", features = ["esp32c3", "panic-handler", "println"] }
esp-println = { version = "0.18", features = ["esp32c3", "log-04"] }
esp-bootloader-esp-idf = { version = "0.6", features = ["esp32c3"] }
```

```rust
// src/main.rs（节选）
#![no_std]
#![no_main]

use esp_backtrace as _;
use esp_hal::{
    clock::CpuClock,
    gpio::{Level, Output, OutputConfig},
    main,
    time::{Duration, Instant},
};
use esp_println::println;

esp_bootloader_esp_idf::esp_app_desc!();

#[main]
fn main() -> ! {
    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    // GPIO8 接普通 LED 的板子才这样写；WS2812 板见上节“LED 引脚因板而异”
    let mut led = Output::new(peripherals.GPIO8, Level::Low, OutputConfig::default());

    loop {
        led.toggle();
        println!("blink");
    }
}
```

### 异步版骨架（Embassy，见第 2 章详解）

```toml
# 在阻塞版基础上加（版本号以 crates.io 为准）
esp-rtos = { version = "0.3", features = ["embassy", "esp32c3"] }
embassy-executor = "0.10"
embassy-time = "0.5"
```

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use esp_hal::timer::timg::TimerGroup;

#[esp_rtos::main]
async fn main(spawner: Spawner) -> ! {
    let peripherals = esp_hal::init(esp_hal::Config::default());
    let timg0 = TimerGroup::new(peripherals.TIMG0);
    let sw_int = esp_hal::interrupt::software::SoftwareInterruptControl::new(
        peripherals.SW_INTERRUPT,
    );
    esp_rtos::start(timg0.timer0, sw_int.software_interrupt0);

    spawner.spawn(blink_task()).unwrap();

    loop {
        Timer::after(Duration::from_secs(60)).await;
    }
}

#[embassy_executor::task]
async fn blink_task() {
    loop {
        Timer::after(Duration::from_millis(500)).await;
    }
}
```

注意：`#[esp_rtos::main]` 只包一层 executor，**不会**替你启动运行时，
必须手动调 `esp_rtos::start(...)`。`esp_rtos::start_second_core(...)`
是双核芯片专用的，**C3 单核绝对不要用**。

---

## 1. `esp-hal` —— 裸机 HAL，所有外设的入口

原文一句话：

> Bare-metal (`no_std`) HAL for all Espressif ESP32 devices.

中文：面向所有 Espressif ESP32 设备的裸机（`no_std`）硬件抽象层。

### 翻译详解

`esp-hal` 是 Espressif ESP32 系列单片机的硬件抽象层，
提供安全、符合 Rust 习惯的 API 来控制硬件外设，让你不用操作系统、
不用 C 的 ESP-IDF，也能直接在裸机上写固件。

它同时提供两种风格的驱动：

- **Blocking（阻塞）**：调用后等外设做完再返回，逻辑最直观，适合 Blink、
  传感器轮询、简单控制；
- **Async（异步）**：配合 Embassy 使用，等待时不占 CPU，适合 Wi-Fi、
  多任务、需要同时等多个事件的工程。异步细节见第 2 章。

在适用的地方，驱动还会实现 `embedded-hal` / `embedded-hal-async` 的 trait。
这意味着很多通用驱动（例如 OLED、传感器驱动）不用为 ESP32 单独重写，
只要它面向 `embedded-hal` 写，就能直接接在 `esp-hal` 的驱动上。
同理，随机数外设还实现了 `rand_core` 的 trait，串口类驱动还实现了
`embedded-io`（`no_std` 版的 `std::io` 对应物）。

### 最核心的两个概念

**1. 芯片 feature 必须选对**

`esp-hal` 以及几乎所有辅助 crate 都要开一个芯片 feature，
例如 `esp32`、`esp32c2`、`esp32c3`、`esp32c5`、`esp32c6`、`esp32h2`、
`esp32s2`、`esp32s3`。同一个工程里所有 esp 系 crate 的芯片 feature
必须一致，否则编译会报错或行为异常。

注意：ESP8266 不在 `esp-hal` 的支持范围内。

**2. 外设单例（Peripheral Singleton）**

每个外设驱动都需要一个“外设单例”来告诉它用的是哪块硬件。
这些单例由 HAL 初始化创建，并作为 `Peripherals` 结构体的字段
从 `esp_hal::init` 返回：

```rust
let peripherals = esp_hal::init(esp_hal::Config::default());
let mut i2c = I2c::new(peripherals.I2C0, /* ... */);
```

单例默认代表该外设的整个生命周期。如果你还想在别处复用同一个外设，
可以用 `reborrow()` 方法借出，而不是用 `core::mem::forget` 绕过去——
官方文档明确提醒：不要对 HAL 定义的类型使用 `core::mem::forget`。

### 库作者特别注意（原文保留）

如果你写的是一个依赖 `esp-hal` 的库，而不是最终应用，官方建议这样引：

```toml
esp-hal = { version = "1", default-features = false }
```

这样可以保证不提前启用 `rt` feature 和芯片 feature，
由最终应用来选芯片、启用 `rt`，保证全工程只有一个入口调用 `init`。
如果你的库依赖 `unstable` 功能，还必须启用 `requires-unstable` feature。

---

### 1.1 C3 外设全表（按表查，✔️=稳定 API，⚒️=`unstable` 后可用）

| 外设 | 模块 | 主类型 / 关键方法 | C3 数量 / 状态 |
|---|---|---|---|
| GPIO | `esp_hal::gpio` | `Input`、`Output`、`OutputOpenDrain`、`Flex`、`Level`、`Event`、`InputConfig`、`OutputConfig`；`Output::new(peripherals.GPIOx, Level::High, OutputConfig::default())`；`set_high()` / `set_low()` / `toggle()`；`is_high()` / `is_low()`；`listen(Event)` | ✔️，GPIO0–GPIO21 共 22 个 |
| SPI master | `esp_hal::spi::master` | `Spi`、`Config`、`Mode`；`Spi::new(peripherals.SPI2, Config::default().with_frequency(Rate::from_khz(100)).with_mode(Mode::_0))`，再 `with_sck` / `with_mosi` / `with_miso` / `with_cs`；读写走 `embedded-hal` 的 `SpiBus`（`read` / `write` / `transfer` / `transfer_in_place`）；DMA 版 `SpiDma` / `SpiDmaBus`（`with_dma(dma_channel).with_buffers(rx, tx)`，单次上限 32736 字节） | ✔️，**只有 SPI2 能给用户用**，SPI0/SPI1 接 Flash，C3 无 SPI3 |
| I2C master | `esp_hal::i2c::master` | `I2c`、`Config`；`I2c::new(peripherals.I2C0, config)?`；`write(addr, &buf)` / `read(addr, &mut buf)` / `write_read(addr, &w, &mut r)` / `transaction(addr, &mut [Operation::…])` | ✔️，**只有 I2C0 一个**；I2C 从机是 ⚒️ |
| UART | `esp_hal::uart` | `Uart`、`UartRx`、`UartTx`、`Config`；`Uart::new(peripherals.UART0, Config::default())?`；`write(b"…")?` / `read(&mut buf)?` / `flush()?` | ✔️，UART0 + UART1 共 2 个 |
| ADC | `esp_hal::analog::adc` | `Adc`、`AdcConfig`、`AdcPin`、`Attenuation`；`AdcConfig::new()` → `enable_pin(pin, Attenuation::_11dB)` → `Adc::new(peripherals.ADC1, config)` → `nb::block!(adc.read_oneshot(&mut pin))?`；另有 `AdcCalBasic` / `AdcCalLine`（C3 可用） | ⚒️，2 个 12-bit SAR ADC，共 6 通道：ADC1 = GPIO0–GPIO4，ADC2 = GPIO5（仅 1 通道） |
| LEDC（PWM） | `esp_hal::ledc` | `Ledc`、`timer::{…}`、`channel::{…}`；`Ledc::new(peripherals.LEDC)` → `ledc.timer::<LowSpeed>(timer::Number::Timer0).configure(timer::config::Config { duty: Duty::Duty5Bit, clock_source: LSClockSource::APBClk, frequency: Rate::from_khz(24) })` → `ledc.channel(channel::Number::Channel0, led).configure(channel::config::Config { timer, duty_pct: 10, pin_config: PinConfig::PushPull })`；`set_duty(duty_pct: u8)`、`start_duty_fade(0, 100, 1000)`、`is_duty_fade_running()` | ⚒️，6 通道，**C3 只有 LowSpeed**；HighSpeed 仅 ESP32 |
| RMT | `esp_hal::rmt` | 模块存在，`Rmt` 等；具体 Tx/Rx 通道类型名本次未核实，查 docs.rs 为准 | ⚒️，4 通道（2 TX + 2 RX），常用于 WS2812 |
| I2S | `esp_hal::i2s` | 模块存在；具体类型 / 方法未核实，查 docs.rs 为准 | ⚒️，1 个 |
| TWAI（CAN） | `esp_hal::twai` | 模块存在；具体类型 / 方法未核实，查 docs.rs 为准 | ⚒️，1 个，兼容 CAN 2.0，**需外接 transceiver，不能直连 CAN 总线** |
| Timer / SYSTIMER | `esp_hal::timer::{OneShotTimer, PeriodicTimer, timg::TimerGroup, systimer::SystemTimer}` | `TimerGroup::new(peripherals.TIMG0)` → `timg0.timer0` / `timer1`；`OneShotTimer::new(timg0.timer0)`，`delay_millis` / `delay_micros` / `delay_nanos`；`PeriodicTimer` 用 `nb::block!(periodic.wait())`；`SystemTimer::new(peripherals.SYSTIMER)`（`alarm0…`，旧版作 Embassy 时间驱动） | ⚒️，TIMG0 + TIMG1（各 54-bit），SYSTIMER 3 个 alarm |
| DMA | `esp_hal::dma` | `DmaRxBuf` / `DmaTxBuf`，`dma_buffers!` / `dma_tx_buffer!` / `dma_rx_stream_buffer!` 等宏；`Mem2Mem`（`into_async()`） | ⚒️，GDMA 3 TX + 3 RX 通道；带 DMA 的外设：SPI2、UHCI0、I2S、AES、SHA、ADC |
| RNG | `esp_hal::rng` | 模块存在；构造函数 / 确切类型名未核实，查 docs.rs 为准 | 矩阵 API ✔️、C3 实现 ⚒️ |
| AES / SHA / RSA / HMAC | `esp_hal::aes` / `sha` / `rsa` / `hmac` | 模块均存在；加密 / 摘要的具体方法名未核实，查 docs.rs 为准；AES / SHA 支持 DMA | ⚒️；注意：**ECC 在 C3 栏为 —，RSA / ECDSA 数字签名在 C3 栏为 ❌，不要在 C3 上用** |
| USB-Serial-JTAG | `esp_hal::usb_serial_jtag` | `UsbSerialJtag`、`UsbSerialJtagRx`、`UsbSerialJtagTx`；`UsbSerialJtag::new(peripherals.USB_DEVICE)`，`into_async().split()` 后实现 `embedded-io(-async)` 的 `Read` / `Write` | ⚒️，固定 CDC-ACM + JTAG，64 字节 payload，占 GPIO18/19 |
| 中断 | `esp_hal::interrupt`（+ `interrupt::software`） | `SoftwareInterruptControl::new(peripherals.SW_INTERRUPT)` → `software_interrupt0`（供 `esp-rtos` 用，见第 2 章）；`#[handler]` 等中断注册宏的确切名字未核实，查 docs.rs 为准 | ⚒️ |
| 时钟 | `esp_hal::clock` | `CpuClock`；`esp_hal::Config::default().with_cpu_clock(CpuClock::max())` | 稳定，最高 160 MHz |
| 温度传感器 | `esp_hal::tsens` | 模块存在；具体 API 未核实 | ⚒️，片内 1 个 |
| eFuse / RTC / 系统 | `esp_hal::efuse` / `rtc_cntl` / `system` / `rom` / `assist_debug` | 均有模块；具体方法未核实 | ⚒️ |
| Wi-Fi / BLE | **不在 esp-hal，在 `esp-radio`** | `esp_radio::wifi` 等，需 `esp-alloc` 堆 + `esp-rtos` 调度器，TCP/IP 走 `embassy-net` | C3：Wi-Fi 4 + BLE 5 LE，无经典蓝牙 |

### 1.2 常用外设示例（C3，阻塞版）

**GPIO 输出（Blink）**

```rust
use esp_hal::gpio::{Level, Output, OutputConfig};
let mut led = Output::new(peripherals.GPIO8, Level::Low, OutputConfig::default());
led.toggle();
```

**GPIO 输入 + 中断等待（异步见第 2 章）**

```rust
use esp_hal::gpio::{Event, Input, InputConfig};
let mut button = Input::new(peripherals.GPIO9, InputConfig::default());
// 阻塞版轮询：button.is_high() / button.is_low()
// 异步版：button.wait_for_rising_edge().await
```

**I2C 读传感器**

```rust
use esp_hal::i2c::master::{Config, I2c};
let mut i2c = I2c::new(peripherals.I2C0, Config::default())?;
let mut buf = [0u8; 2];
i2c.write_read(0x48, &[0x00], &mut buf)?;
```

**SPI 读写**

```rust
use esp_hal::spi::master::{Config, Mode, Spi};
use esp_hal::time::Rate;
let spi = Spi::new(
    peripherals.SPI2,
    Config::default()
        .with_frequency(Rate::from_khz(100))
        .with_mode(Mode::_0),
)?
.with_sck(peripherals.GPIO6)
.with_mosi(peripherals.GPIO7)
.with_miso(peripherals.GPIO2);
// 再用 embedded-hal 的 SpiBus：spi.write(&[0x01]) / spi.transfer(&mut buf)
```

**UART 收发**

```rust
use esp_hal::uart::{Config, Uart};
let mut uart = Uart::new(peripherals.UART1, Config::default())?;
uart.write(b"hello")?;
let mut buf = [0u8; 16];
let _n = uart.read(&mut buf)?;
```

**ADC 单次采样**

```rust
use esp_hal::analog::adc::{Adc, AdcConfig, Attenuation};
let mut adc_config = AdcConfig::new();
let mut pin = adc_config.enable_pin(peripherals.GPIO0, Attenuation::_11dB);
let mut adc = Adc::new(peripherals.ADC1, adc_config);
let value: u16 = nb::block!(adc.read_oneshot(&mut pin))?;
```

**LEDC 调光**

```rust
use esp_hal::ledc::{Ledc, timer::{self, TimerIFace}, channel::{self, ChannelIFace}};
use esp_hal::time::Rate;
let mut ledc = Ledc::new(peripherals.LEDC);
let mut timer = ledc.timer::<timer::LowSpeed>(timer::Number::Timer0);
timer.configure(timer::config::Config {
    duty: timer::config::Duty::Duty5Bit,
    clock_source: timer::LSClockSource::APBClk,
    frequency: Rate::from_khz(24),
})?;
let mut channel = ledc.channel(channel::Number::Channel0, peripherals.GPIO8);
channel.configure(channel::config::Config {
    timer: &timer,
    duty_pct: 10,
    pin_config: channel::config::PinConfig::PushPull,
})?;
channel.set_duty(50)?;
```

---

## 2. 异步专章：Embassy + `esp-rtos`（C3 专用）

v1 只有一句话“异步配合 Embassy 使用”，这一章是 v2 补的核心。

### 2.1 总结构：谁负责什么

| 层 | 负责什么 |
|---|---|
| `esp-hal` | 外设驱动（Blocking + Async 两种形态） |
| `esp-rtos` | 调度器 + Embassy 时间驱动 + Embassy 执行器（1.x 线的官方接法） |
| `embassy-executor` | 任务宏 `#[embassy_executor::task]`、执行器类型 |
| `embassy-time` | `Timer`、`Duration`、`Instant`、`Ticker`、超时 |
| `embassy-sync` | 任务间 `Channel` / `Signal` / `Mutex` / `PubSubChannel` |
| `embassy-futures` | `select` / `join` 等 future 组合子 |
| `embassy-net` | TCP/IP 栈，见第 11 章 |
| `esp-radio` | Wi-Fi / BLE 的后台协议栈，见第 8–9 章 |

**不要再用 `esp-hal-embassy`**：它是更早的接法，1.x 的新工程请走
`esp-rtos` 路线，不要把两套调度器混在一个工程里。

### 2.2 启动顺序（错一步就醒不过来）

```rust
#[esp_rtos::main]
async fn main(spawner: Spawner) -> ! {
    let peripherals = esp_hal::init(esp_hal::Config::default());
    let timg0 = TimerGroup::new(peripherals.TIMG0);
    let sw_int = SoftwareInterruptControl::new(peripherals.SW_INTERRUPT);
    esp_rtos::start(timg0.timer0, sw_int.software_interrupt0);
    // 堆要在 esp-radio 之前建好（见第 3 章）
    // esp_alloc::heap_allocator!(size: 72 * 1024);
    // 再 esp_radio::init() / wifi::new() …
    spawner.spawn(my_task()).unwrap();
    loop { Timer::after(Duration::from_secs(60)).await; }
}
```

顺序：`esp_hal::init` → `esp_rtos::start` → 堆 → `esp_radio::init` →
`wifi::new`。`esp-rtos` 的 feature：`embassy`（Embassy 集成：时间驱动 +
执行器）、`esp-radio`、`esp-alloc`、`log-04` / `defmt`，芯片选 `esp32c3`。
**不要给 `embassy-executor` 开任何 `arch-*` feature**，这是 `esp-rtos`
文档明确说的。

### 2.3 哪些驱动有异步版

| 驱动 | 开异步方式 | 关键异步方法 |
|---|---|---|
| I2C master | `i2c.into_async()` | `write_async(addr, &buf).await`、`read_async(addr, &mut buf).await`、`write_read_async(…).await`、`transaction_async(addr, &mut [Operation::Write(..), Operation::Read(..)]).await`；阻塞方法在 async 模式下仍可用 |
| UART | `uart.into_async()`（`Uart` / `UartTx` 同理） | `write_async(&msg).await`、`flush_async().await`、`read_async(&mut buf).await` |
| SPI master | `Spi` 可阻塞 / 异步两用 + DMA 版 `into_async()` | 固有 `*_async` 方法名未核实；应走 `embedded-hal-async` 的 `SpiBus` trait（`read` / `write` / `transfer` / `transfer_in_place` 的 async 版），DMA 版走 `SpiDmaBus` |
| USB-Serial-JTAG | `UsbSerialJtag::new(…).into_async().split()` → `(rx, tx)` | `embedded-io-async` 的 `Read` / `Write` |
| GPIO Input / Flex | 本身即支持 await（需 `unstable`） | `wait_for(Event)`、`wait_for_high()`、`wait_for_low()`、`wait_for_rising_edge()`、`wait_for_falling_edge()`、`wait_for_any_edge()`；**`Flex::wait_for` 注明不是 cancellation-safe**，会覆盖之前的 `listen` |
| OneShotTimer / PeriodicTimer | `OneShotTimer::new(…).into_async()` | `delay_millis_async` / `delay_micros_async` / `delay_nanos_async`；Periodic 的 async `wait` 未核实 |
| ADC | **未找到异步版** | 只有阻塞 `read_oneshot`；在 async 任务里直接调会占住 executor，短调用可接受，要么另起任务 / 用阻塞任务包装——不要写成“ADC 有 async” |
| Wi-Fi（esp-radio） | `esp_radio::wifi` + `embassy-net` | `Runner::run().await` 等，走 `embassy-net` 的 async socket（见第 8、11 章） |

### 2.4 `Delay` vs `embassy_time::Timer`（最容易混的两处）

| | `esp_hal::delay::Delay` | `embassy_time::Timer` |
|---|---|---|
| 本质 | **阻塞忙等**：`Delay::new()`，内部用 SYSTIMER 计时，`delay_millis` / `delay_micros` / `delay_nanos`（另实现 `embedded-hal` 的 `DelayNs::delay_ns`），“至少等这么久”，精度受中断影响 | **异步让出**：`Timer::after(Duration::from_millis(500)).await` / `Timer::after_millis(500).await`，到时由时间驱动唤醒，期间 executor 可跑别的任务 |
| 前提 | 不用 executor，`#[esp_hal::main]` 的阻塞程序直接用 | 必须已 `esp_rtos::start(…)` 启动时间驱动，否则不会醒 |
| 用法 | 短时精确等待、传感器触发脉冲、初始化时延 | 任务间节拍、Blink、周期采样；另有 `embassy_time::Delay` 可作 async 版 `DelayNs` 传给驱动 |

### 2.5 任务、定时、超时、二选一

```rust
use embassy_executor::Spawner;
use embassy_time::{Duration, Instant, Timer, Ticker, with_timeout, TimeoutError};
use embassy_futures::select::{select, Either};

// 生成任务：参数必须 'static、非泛型、具体类型
spawner.spawn(blink_task()).unwrap();

#[embassy_executor::task]
async fn blink_task() {
    loop {
        Timer::after(Duration::from_millis(500)).await;
    }
}

// 周期任务：Ticker 比每次 Timer::after 更准（不累计漂移）
let mut ticker = Ticker::every(Duration::from_secs(1));
loop {
    ticker.next().await;
    // 采样一次
}

// 超时：超时会 drop 掉那个 future
match with_timeout(Duration::from_secs(5), some_future).await {
    Ok(v) => { /* 按时完成 */ }
    Err(TimeoutError) => { /* 超时 */ }
}

// 二选一：先完成者胜
match select(socket.read(&mut buf), Timer::after(Duration::from_secs(10))).await {
    Either::First(n) => { /* 读到了 */ }
    Either::Second(()) => { /* 超时 */ }
}
```

`embassy_futures::join::join` 是等全部完成（BLE host + 广播循环常用），
与 `select` 的“先完成者胜”正好相反。

### 2.6 任务间通信：`Channel` / `Signal` / `Mutex`

```rust
use embassy_sync::{
    blocking_mutex::raw::ThreadModeRawMutex,
    channel::Channel,
    mutex::Mutex,
    signal::Signal,
};

// MPMC 通道：每条消息只被一个消费者收到
static LED_CH: Channel<ThreadModeRawMutex, bool, 4> = Channel::new();
// 只给一个消费者发最新值
static STATE: Signal<ThreadModeRawMutex, u8> = Signal::new();
// 共享可变状态
static SHARED: Mutex<ThreadModeRawMutex, u32> = Mutex::new(0);

LED_CH.send(true).await;
let v = LED_CH.receive().await;
STATE.signal(42);
let v = STATE.wait().await;
let mut guard = SHARED.lock().await;
*guard += 1;
```

`RawMutex` 常用 `ThreadModeRawMutex` 或 `CriticalSectionRawMutex`。
`PubSubChannel` 是广播（一发多收），与 `Channel` 的一发一收不同。

### 2.7 单核影响（C3 只有 1 个 RISC-V 核，这条必须记住）

Embassy 是**协作式调度，没有抢占**——任务只在 `.await` 处让出 CPU；
一个从不 `.await` 的长计算会饿死 Wi-Fi、`net_task`、BLE，
表现为断线 / 丢包 / 超时。`esp-rtos` 的调度器是给 radio 后台用的，
不是让你并行跑两个核。任务间传数据用 `Channel` / `Signal` / `Mutex`，
不要共享可变裸状态；`'static` 约束意味着共享资源一般放 `static` 里
（`StaticCell` / `static Mutex`）。

异步驱动不是 `Send`（中断绑在当前核），跨核先传 Blocking 版、
到目标核再 `into_async`；**C3 单核可忽略这条**。

---

## 3. `esp-alloc` —— `no_std` 下的堆

原文一句话：

> A `no_std` heap allocator for RISC-V and Xtensa processors from Espressif.
> Supports all currently available ESP32 devices.

中文：面向 Espressif RISC-V 和 Xtensa 处理器的 `no_std` 堆分配器，
支持目前所有 ESP32 设备。

### 翻译详解

在 `no_std` 环境下，标准库不存在，`Vec`、`Box`、`String`、
`HashMap` 这些需要堆的东西默认是不能用的。`esp-alloc` 就是补上这一块：
它提供一个全局堆分配器，让 `alloc` crate 能在裸机上工作。

最常用的写法只有一行宏，在 `main` 开头、用到堆之前调用：

```rust
esp_alloc::heap_allocator!(size: 72 * 1024);
```

意思是：划 72 KiB 出来当全局堆。堆多大没有标准答案，要按你的程序算：
Wi-Fi、网络协议栈、显示缓冲、JSON 解析都会吃堆，太小会在运行时
分配失败，太大又会挤占栈和其他静态内存。先给一个保守值，
跑起来后再用 `esp_alloc::HEAP.stats()` 看实际占用去调。

进阶用法（保持原文名）：

- `heap_allocator!`：最常用，指定字节数建一个内部 RAM 堆；
  只接受 `size:` + 属性，没有其它变体；
- `heap_allocator!(#[ram(reclaimed)] size: 64 * 1024)`：把二级 bootloader
  用过、主程序阶段已空闲的内存也回收进堆。这段内存启动阶段不能当栈用，
  但进主程序后当堆是安全的；
- 手动建堆：`static mut HEAP: MaybeUninit<[u8; HEAP_SIZE]>;` +
  `HEAP.add_region(HeapRegion::new(ptr, size, MemoryCapability::Internal.into()))`；
- **`psram_allocator!` 在 C3 上不适用（C3 无 PSRAM），不要写进 C3 文档。**

### `HEAP.stats()`：怎么看堆还剩多少

```rust
println!("{}", esp_alloc::HEAP.stats());
```

`esp_alloc::HEAP.stats() -> HeapStats`，`HeapStats` 实现 `Display` /
`defmt::Format`，可直接打印。公开字段只有
`region_stats: [Option<RegionStats>; 3]`、`size`、`current_usage`；
示例输出里的 `Max usage` / `Total freed` / `Total allocated`
需开 `internal-heap-stats`（每次 alloc/dealloc 多算一次），
结构体页未逐项列出，写代码时以 docs.rs 为准。

### C3 堆多大够用（经验法则，非官方数字）

C3 总 SRAM 400 KB，没有官方推荐值，只能给实测参照：

- 不联网：32–64 KB 够跑；
- Wi-Fi 或 BLE：至少留 128 KB 以上再加应用余量；
- 实测参照：某 C3 项目配 `#[ram(reclaimed)]` 66,320 字节 + 主堆 174,080
  字节 ≈ 236 KB；Wi-Fi STA 实测堆占用 47–57 KB、开放 AP 53–63 KB；
  另一 C3 显示项目用 150 KB 堆。

**最终以 `HEAP.stats()` 实测为准**，不要把上面的参照当成官方数字。

### 必须知道的几条限制

1. **全局分配器只能有一个。** 你可以给这一个分配器配多个内存区
   （内部 RAM、PSRAM、多个不连续块），但不能同时注册两个全局分配器。
   需要多个独立分配器时，要走 nightly 的 `allocator_api` 或
   `allocator_api2`，`esp-alloc` 对后者有实现。
   nightly 可直接用 `alloc` 的 `Box` / `Vec::new_in`；stable 可用
   `allocator-api2 = { version = "0.3", features = ["alloc"] }` +
   `esp_alloc::{InternalMemory, ExternalMemory, AnyMemory}`，
   如 `Vec::new_in(esp_alloc::InternalMemory)`；另有 `EspHeap::empty()` +
   `add_region()` 自建堆（最多 3 个 region）。
2. **不是所有 RAM 都能拼成一块平坦的堆。** 有些芯片的内存映射不连续，
   有的区域是 ROM 代码保留的，不能覆盖；上一点提到的 `reclaimed` 内存
   就是处理这种不连续的办法之一。
3. **Xtensa 芯片 + PSRAM 有个原子操作坑。** 在 Xtensa 芯片上，
   PSRAM 里的原子操作工作不正常，可能造成数据竞争、让原子操作的意义失效，
   因此分配器不能用来分配 `Atomic*` 类型。RISC-V 芯片（C3/C6/H2）
   不受这条影响，PSRAM 里的原子操作是正常的。**C3 本来就没有 PSRAM，
   这条对 C3 只是背景知识。**
4. **作全局分配器用时，Rust 版本要求。** 官方文档注明需要 Rust 1.68
   或更高版本，或者 nightly 通道。现在的稳定版 Rust 早已满足，
   这条更多是历史说明，不用特意切 nightly。

### 没有堆会坏什么

`alloc` 的 `Vec` / `String` / `Box` 全不可用；**Wi-Fi / BLE 必须有
动态分配器 + 抢占式调度器（`esp-rtos`）**（`esp-radio` 文档原话）；
`embassy-net`（底层 smoltcp）同样要堆。**顺序必须是：先建堆 →
再 `esp_radio::init()` / `esp_rtos::start()`**，否则 radio 初始化分配即失败。

一句话：没有堆需求（纯 Blink、纯 GPIO）可以不用它；
一旦要用 `Vec`、字符串、Wi-Fi、网络、显示库，它就是必选项，
而且要在用堆之前、越早初始化越好。

---

## 4. `esp-backtrace` —— panic 时告诉你死在哪

原文一句话：

> Bare-metal backtrace support for Espressif devices.

中文：面向 Espressif 设备的裸机调用栈（backtrace）支持。

### 翻译详解

`no_std` 的二进制程序没有默认的 panic 处理器，不自己提供一个，
程序根本编不过。`esp-backtrace` 就是官方推荐的那个处理器：
panic 或 CPU 异常发生时，它会把出错原因和调用栈打印到串口，
然后停住（或按配置 halt 所有核），而不是悄无声息地死机。

用法上它几乎是“零代码”的，只要在依赖里打开 feature，
并在 `main.rs` 顶部导入一次：

```toml
esp-backtrace = { version = "0.18", features = ["esp32c3", "panic-handler", "println"] }
```

```rust
use esp_backtrace as _;
```

`as _` 的意思是：我不需要直接调用它里面的任何东西，
只要让它把 panic 处理器注册进最终产物。

### 关键 feature（保持原文）

| feature | 中文含义 |
|---|---|
| `panic-handler` | 接管 Rust 的 `panic!`，会引入 `esp-println` 做输出 |
| `exception-handler` | 接管 CPU 异常（非法指令、访存错误等），同样会引入 `esp-println`；**0.19 版文档只提 panic handler，新版是否保留未核实** |
| `println` | 用 `esp-println` 打印消息 |
| `defmt` | 改用 `defmt` 日志格式打印 panic / 异常消息 |
| `colors` | 打印的消息带颜色（只对 panic / 异常处理器有效） |
| `halt-cores` | panic / 异常时停住 ESP32 / ESP32-S3 的两个核，而不是只在一个核上空转；**C3 单核可忽略** |
| `semihosting` | panic 时调用 `semihosting::process::abort()` |
| `custom-halt` / `custom-pre-backtrace` | 调用外部 `custom_halt()` / `custom_pre_backtrace()` |

### C3 专属：必须加 `force-frame-pointers`

C3 是 RISC-V，**必须在 `.cargo/config.toml` 加 `"-C", "force-frame-pointers"`**，
否则 panic 处理只打印栈转储、需工具解码：

```toml
# .cargo/config.toml
[build]
rustflags = ["-C", "force-frame-pointers"]
```

### 自取回溯与解码

- 自取回溯：`arch::backtrace()`，默认最多 10 帧
  （`ESP_BACKTRACE_CONFIG_BACKTRACE_FRAMES`，默认 10；
  栈转储上限 `ESP_BACKTRACE_CONFIG_STACK_DUMP_MAX_SIZE`，默认 8K，
  可选 4K / 8K / 16K / 32K）。
- 串口上看到的是一串地址，想看到函数名和行号，要用 `espflash` 等工具
  结合 ELF 文件解析。新版 espflash 支持按 `.debug_frame` 解码 RISC-V
  回溯（另有 `.eh_frame`），`monitor` 有 `--unwind-tables auto|debug-frame|eh-frame`、
  `--all-addresses`、`--no-addresses` 等选项。

### 容易踩的坑

- **`esp-backtrace` 和 `esp-println` 的输出通道必须一致。**
  例如一个走 UART、一个走 USB-Serial-JTAG，那 panic 信息就可能
  发到你没在看的那条通道上，看起来像“没打印”。
- **它和 `panic-probe` 只能二选一。** 一个最终产物里只能有一个
  panic 处理器，两个都引进来会链接冲突。
- **调用栈地址需要配合符号表解析。** 串口上看到的是一串地址，
  想看到函数名和行号，要用 `espflash` 等工具结合 ELF 文件解析，
  不是串口直接给你中文函数名。

---

## 5. `esp-println` —— 串口打印和日志

原文一句话：

> Provides `print!` and `println!` implementations various Espressif devices.

中文：为各类 Espressif 设备提供 `print!`、`println!` 的实现。

### 翻译详解

`esp-println` 解决的是 `no_std` 下最基础的问题：没有标准库，
`println!` 默认是没有地方可去的。这个 crate 把 `print!`、`println!`、
`dbg!` 接到芯片的串口输出上，并在此基础上提供日志能力，
支持所有 ESP32 系列设备。

基本用法：

```toml
esp-println = { version = "0.18", features = ["esp32c3", "log-04"] }
```

```rust
use esp_println::println;
println!("Hello world");
```

注意：一定要开芯片 feature，不开或开错芯片，输出通道就可能不对。

### 输出通道（二选一，必须且只能开一个）

| feature | 中文含义 |
|---|---|
| `auto`（默认） | 运行时检测：检测到 USB SOF 包 → 走 USB-Serial-JTAG，否则走 UART；源码按 `USB_DEVICE_INT_RAW` 判断，C3 地址 `0x60043008` |
| `jtag-serial` | 走 USB-Serial-JTAG |
| `uart` | 走 UART（C3 走 UART0） |
| `no-op` | 不输出 |

- **USB-Serial-JTAG**：只在 ESP32-C3、C5、C6、C61、H2、S3 等
  自带 USB-Serial-JTAG 的芯片上可用，C3 的开发板插上 USB 线就能看日志，
  不需要外接串口芯片；C3 上是 GPIO18/19，烧录和打印可共用一根 USB 线。
- **`defmt` 后端**：用 `defmt-espflash` feature 时，`esp-println`
  会装一个 `defmt` 全局日志器，输出和 `println!()` 走同一条数据流，
  主机端用 `espflash` 解码，日志体积更小、格式更结构化；
  `defmt` 输出带 rzCOBS 帧，**只能用 espflash 读**
  （`espflash monitor --log-format defmt`）。

与 `esp-backtrace` 的配对表（旧版文档，0.18 需复核）：
`uart` ↔ `print-uart`，`jtag-serial` ↔ `print-jtag-serial`；
**两边输出方式不一致时 panic 信息会丢。**

### 日志：`log` crate 的简单实现

打开 `log-04` feature、并依赖 `log` 0.4 后，就可以初始化一个
简单的日志器：

```rust
esp_println::logger::init_logger(log::LevelFilter::Info);
log::info!("Logger is setup");
```

也可以用环境变量的方式，让级别跟随环境走：

```rust
esp_println::logger::init_logger_from_env();
```

这时用到的环境变量是 `ESP_LOG`，写法类似 `RUST_LOG`，
例如 `warn,test::foo=info,test::foo::bar=debug`，但**不支持正则表达式**。
注意：`init_logger_from_env()` 读的是**编译期**环境变量 `ESP_LOG`
（build.rs 生成过滤表），不是运行期改的。
默认 feature 里还有 `colors`，日志输出会带颜色。
`timestamp` feature 需自己实现 `_esp_println_timestamp() -> u64`
（毫秒，可用 `esp_hal::time::Instant`）。

如果这个简单日志器不够用（例如你要写文件、发网络、做过滤），
官方文档的建议是：在 `esp-println` 之上自己实现一个 `log` 的
`Log` trait，而不是去改 `esp-println` 本身。

---

## 6. `esp-storage` —— Flash 底层读写，不是文件系统

原文一句话：

> Implementation of embedded-storage traits to access unencrypted ESP32 flash.

中文：实现 `embedded-storage` trait，用于访问未加密的 ESP32 Flash。

### 翻译详解

`esp-storage` 给你的是 Flash 的“裸读写”能力：按地址读、按扇区擦、
按页写，数据掉电不丢失。它本身**不是**文件系统，
也不是键值数据库，上面要跑什么由你选：

- 直接用它提供的 `FlashStorage` 做定长记录、配置块；
- 在它上面跑 `esp-nvs`，得到和 ESP-IDF 兼容的 NVS 键值存储（见第 7 章）；
- 在它上面跑 littlefs 等文件系统，得到文件读写。

典型用法：

```rust
let storage = esp_storage::FlashStorage::new(peripherals.FLASH);
```

注意 `peripherals.FLASH` 这个外设单例：Flash 外设也是从
`esp_hal::init` 返回的 `Peripherals` 里取的，和 GPIO、I2C 一样，
不能凭空 new 一个。**重复调用 `FlashStorage::new` 会 panic。**

### API 分层（不要混淆）

| 层 | 内容 |
|---|---|
| `embedded-storage` trait | `ReadStorage::read(offset, &mut [u8])`、`Storage::write(offset, &[u8])`（内部按扇区读-改-擦-写）、`NorFlash::{read, write, erase}`，以及 `READ_SIZE` / `WRITE_SIZE` / `ERASE_SIZE`（最小读 / 写 / 擦单位） |
| 底层（`low-level` feature，`ll.rs`，全是 `unsafe`、不做前置检查） | `spiflash_read` / `spiflash_write` / `spiflash_erase_sector` / `spiflash_erase_block` / `spiflash_unlock` / `spiflash_write_encrypted` |

常量（`esp-storage` 0.8.0 源码已核对）：`WORD_SIZE: u32 = 4`，
`SECTOR_SIZE: u32 = 4096`。C3 的扇区 / 字大小与全系列通用，
**C3 专属的只是 Flash 容量（通常 4 MB，`capacity()` 自动识别）和“单核”**——
源码里 `FlashStorageError` 有“多核运行时不可写 Flash”的变体，C3 单核用不上。
`capacity()` 返回总字节数，由 bootloader 头里的 Flash 大小字段决定。

坑（esp-hal issue #5484，已核实）：`ReadNorFlash::read` 目标切片若非
4 字节对齐，会在栈上悄悄分配 4096 字节的 `FlashSectorBuffer` 做中转；
`[u8; 4096]` 本身不对齐保证。

### 必须知道的限制

1. **未加密。** 这个 crate 访问的是未加密 Flash，敏感数据
   （Wi-Fi 密码、密钥）不要明文往里写，要加密后再存，
   或者走 ESP-IDF 的加密分区方案。
2. **Flash 有擦写寿命和扇区粒度。** 读可以无限次，写要按扇区擦除
   再写，频繁写同一个地址会加速损耗。需要频繁更新的计数、
   日志类数据，要做磨损均衡或换 NVS / 文件系统，不要直接裸写。
   NOR 规则：写前必须擦除（只能 1→0），同一 word 不可写两次，
   断电时正在写的 word 内容未定义、其余页保证不变；擦除后区间全为 1。
3. **不要写到程序和分区表所在的区域。** Flash 里有分区表、
   bootloader、应用本身，写错偏移会把固件写坏。用 NVS 或文件系统时，
   要按分区表给它划一块专用的偏移和大小，参数必须和你的分区表对上，
   不能随手填（见第 7 章的分区表）。
4. **`low-level` feature 是给 `esp-nvs` 这类上层用的。**
   普通应用直接用 `FlashStorage` 就够了。旧版 `esp-nvs` 示例说需要
   `low-level`，但 0.5.0 的文档改为“两边开同一芯片 feature 即可，
   ROM CRC32 由 `esp-nvs` 的 `Crc` 实现提供；不要 ROM CRC 可自己用
   `esp_nvs::platform::software_crc32` 实现 `Crc`”——当前版是否仍需
   `low-level` 未核实，写文档时以 docs.rs 为准。

### 分区表（ESP-IDF 通用，非 C3 专属）

分区表本身在 Flash `0x8000`，长 `0xC00`。no_std 下 espflash 是否默认
刷分区表**未核实**，写文档时必须说明“需自带 `partitions.csv` 并确认
刷入方式”，不要写成默认就有。

单 factory、无 OTA：

```csv
nvs,      data, nvs,     0x9000,  0x6000
phy_init, data, phy,     0xf000,  0x1000
factory,  app,  factory, 0x10000, 1M
```

双 OTA：

```csv
nvs,      data, nvs,     0x9000,  0x4000
otadata,  data, ota,     0xd000,  0x2000
factory,  app,  factory, 0x10000, 1M
ota_0,    app,  ota_0,   0x110000, 1M
ota_1,    app,  ota_1,   0x210000, 1M
```

结论：用默认表时，`esp-nvs` 应传 `partition_offset = 0x9000,
partition_size = 0x6000`（OTA 表则 `0x4000`），不是 v1 示例的
`0x390000 / 0x32000`——那只是某个示例的偏移，不是默认分区。

---

## 7. `esp-nvs` —— 掉电不丢的键值存储（v1 完全漏掉，本版补全）

定位：ESP-IDF 兼容的裸机 NVS 库，Flash 上的数据格式与 ESP-IDF 互通
（v4.0 起的多页 blob 格式；旧单页 blob 打开时迁移）；生产可用但作者
自述“可能还有小毛病”；内部用 `unsafe` 转 C 结构体，靠 CRC32 校验。

```toml
esp-storage = { version = "0.8", features = ["esp32c3"] }
esp-nvs     = { version = "0.5", features = ["esp32c3"] }
```

注意：另有升级记录出现 `esp-storage` 0.9.0 / 0.10.0，写文档前需以
crates.io 为准核对最新版。

### 纠正：真实 API 长这样（不要照 C API 写）

`NvsPartition::new` / `EspNvs` / `get_i32` / `set_i32` / `commit` 是
**ESP-IDF C API（`nvs_open` / `nvs_get_i32` / `nvs_set_i32` /
`nvs_commit` / `nvs_erase_key`）和 std 路线 `esp-idf-svc`
（`EspDefaultNvsPartition` / `EspNvs::new(partition, namespace, readwrite)`）**
的写法，**不是 no_std `esp-nvs`**。两者不要混写进同一份文档。

no_std `esp-nvs` 的真实 API（docs.rs 最新版源码已核对）：

```rust
let storage = esp_storage::FlashStorage::new(peripherals.FLASH);
let mut nvs = esp_nvs::Nvs::new(0x9000, 0x6000, storage).expect("failed to create nvs");
```

构造：`Nvs::new(partition_offset: usize, partition_size: usize, hal: T)
-> Result<Nvs<T>, Error>`，其中 `T: Platform`（提供 Flash 访问 + CRC32）。
打开时读遍全分区：解析命名空间、建每页哈希键缓存、清理重复条目 / 孤儿 blob；
CRC 坏的页 / 条目标记损坏，必要时擦除。

约束（源码已核对）：offset 必须是 4K 倍数，否则 `InvalidPartitionOffset`；
size 必须是非零 4K 倍数且 `offset + size` 不超出 `hal.capacity()`，
否则 `InvalidPartitionSize`；**可写至少要 2 个扇区**（一页写、一页留给
碎片整理），少于 2 个时 `set` 报 `FlashFull`。

### 命名空间与键

命名空间就是“文件夹”，防不同模块键名冲突；命名空间名和键名都
**最长 15 字符**（`Key` 是 16 字节：15 字符 + `\0`，`MAX_KEY_LENGTH = 15`），
一个分区最多 254 个命名空间；同名命名空间在不同分区算不同的。

`Key` 构造（已核实）：`Key::from_str("my_key")` /
`Key::from_array(b"my_key")` / `Key::from_slice(b"my_key")`，
建议用 `const { Key::from_str(...) }` 在编译期转换。

### 读写（泛型方法 + trait）

```rust
pub fn get<R>(&mut self, namespace: &Key, key: &Key) -> Result<R, Error>
    where Nvs<T>: Get<R>
pub fn set<R>(&mut self, namespace: &Key, key: &Key, value: R) -> Result<(), Error>
    where Nvs<T>: Set<R>
// trait 本体：Get::get(&mut self, namespace: &Key, key: &Key) -> Result<T, Error>
//             Set::set(&mut self, namespace: &Key, key: &Key, value: T) -> Result<(), Error>
```

类型：`bool`、有 / 无符号整数到 64 位、`String`、`Vec<u8>`（即 `&[u8]` blob）；
`&str` 单页、最大 4000 字节；blob 可跨多页、最大约 500 kB
（源码注：`MAX_BLOB_SIZE - 1 = 507,999` 字节）；同键重写会替换旧值和旧类型，
类型不匹配读时报错。

**`set` 就是“写即落盘”，全文检索 `Nvs` 页无 `commit`**——与 ESP-IDF C API
的“先缓存、后 `nvs_commit()`”不同，**不要照 C API 补 `commit`**。

### 删除与清理

`delete(&mut self, namespace: &Key, key: &Key)`——**忽略不存在的键 /
命名空间**（对应 C API 的 `nvs_erase_key`，但 Rust 侧无 `commit`）；
`set_purge_mode(bool)` / `purge_mode()` / `purge_all(&namespace)`
（对标 ESP-IDF `NVS_READWRITE_PURGE`，物理清零已删数据，默认关闭）；
另有 `namespaces()`、`keys()`、`typed_entries()`（产出
`(namespace, key, ItemType)`）、`statistics()`、`into_inner()`（取回底层 Flash）。

### 错误表（`esp_nvs::error::Error`，`#[non_exhaustive]`，15 个变体）

`InvalidPartitionOffset`、`InvalidPartitionSize`、`FlashError`、
`NamespaceNotFound`、`NamespaceTooLong`、`NamespaceMalformed`、
`ValueTooLong`、`KeyMalformed`、`KeyTooLong`、`KeyNotFound`、
`ItemTypeMismatch(ItemType)`、`CorruptedData`、`FlashFull`、
`TooManyNamespaces`、`PageFull`。
文档原话：调用方一般只需处理 `NamespaceNotFound` 和 `KeyNotFound`。

### 最小示例

```rust
let storage = esp_storage::FlashStorage::new(peripherals.FLASH);
let mut nvs = esp_nvs::Nvs::new(0x9000, 0x6000, storage).expect("failed to create nvs");
let ns = const { esp_nvs::Key::from_str("config") };
let key = const { esp_nvs::Key::from_str("boot_count") };
let count: i32 = nvs.get(&ns, &key).unwrap_or(0);
nvs.set(&ns, &key, count + 1).unwrap();
let name: String = nvs.get(&ns, &const { esp_nvs::Key::from_str("name") }).unwrap_or_default();
nvs.delete(&ns, &key).unwrap(); // 无 commit，不要照 C API 补 commit
```

注意：`get::<String>` / `get::<Vec<u8>>` 的具体行为（空串、缺省）未在示例中
核实，`unwrap_or_default` 只是示意；`use esp_nvs::{Get, Set}` 是否必需
未核实——方法本身是 inherent 方法，trait 导入可能不需要，写代码时不要
硬写导入行，先看编译器提示。

---

## 8. `esp-radio` —— Wi-Fi、BLE、ESP-NOW（原 `esp-wifi`）

原文一句话：

> Wi-Fi, BLE, IEEE 802.15.4, and ESP-NOW functionality for Espressif devices.

中文：面向 Espressif 设备的 Wi-Fi、BLE、IEEE 802.15.4 和 ESP-NOW 功能。
**C3 没有 802.15.4 射频，Zigbee / Thread / Matter-over-Thread 都做不了**，
软件升级也补不出来；要原生 Zigbee/Thread 请换 C6 / H2。
`esp-radio` 的 `ieee802154` feature 只在有该射频的芯片（C6/H2 等）上
有意义，且不能和 Wi-Fi 同时用。

### 翻译详解

这是 1.x 生态里无线部分的统一入口。老教程里写的 `esp-wifi`
在 1.x 里已经改名、重组为 `esp-radio`，看到老教程不要直接照抄
crate 名，先确认它对应的是哪个版本线。

它包含三类 C3 可用的能力（不同芯片支持情况不同，查文档时先对芯片）：

- **Wi-Fi**：Station（连路由器）和 Access Point（自己开热点）两种模式；
- **BLE**：低功耗蓝牙，通过 `BleConnector` 给 `trouble-host` 这类
  HCI 蓝牙主机做传输层；C3 只有 BLE 5.0，**无经典蓝牙**；
- **ESP-NOW**：无连接、低功耗的点对点通信，适合设备之间直接传小数据。

**`esp-radio` 本身不提供 TCP/IP、HTTP、TLS**：它只给 Wi-Fi / BLE /
ESP-NOW 的 L2 驱动，TCP/IP、DHCP、DNS 要靠 `embassy-net`
（底层历史上是 `smoltcp`；近期 embassy 侧有提到改用 `xarxa` 引擎，
C3 官方例子用的版本是否已切过去**未核实**，落版本时以你锁定的
`embassy-net` 文档为准）。

### 前置条件：比前面几个 crate 都重

Wi-Fi 和 BLE 要在后台持续跑协议栈（定时器、中断、状态机），
因此必须同时满足：

1. **堆**：用 `esp-alloc` 先初始化堆，协议栈要动态分配；
2. **调度器**：用 `esp-rtos`（或等价的 `esp-radio-rtos-driver` 实现）
   给协议栈提供调度，IEEE 802.15.4 不需要调度器是个例外（C3 无关）；
3. **网络协议栈（如果要 TCP/IP）**：`esp-radio` 本身只给到无线驱动层，
   TCP/IP、DHCP、DNS 要接 `embassy-net`（基于 smoltcp），
   再由 `embassy-net` 把 `esp-radio` 的 Wi-Fi 接口当作网卡用。

一句话：`esp-hal` 是手，`esp-alloc` 是内存，`esp-rtos` 是心跳，
`esp-radio` 是无线电，`embassy-net` 是 TCP/IP 脑子，缺一个都跑不起来。
radio 的二进制 blob 要求优化等级 O2/O3，debug 构建需在 `Cargo.toml`
里给 `esp-radio` 单独提优化等级。

### 8.1 初始化与 Wi-Fi 入口

已核实的两套写法（版本不同，二选一）：

```rust
// 0.16–0.18 常见写法
let radio_init = esp_radio::init().expect("radio init");
let (wifi_controller, interfaces) =
    esp_radio::wifi::new(&radio_init, peripherals.WIFI, Default::default()).unwrap();
let wifi_interface = interfaces.sta; // 0.18 用 interfaces.station / Interface 类型
```

```rust
// Rust on ESP Book 当前写法（1.x 方向）
let (mut controller, interfaces) = esp_radio::wifi::new(
    peripherals.WIFI,
    ControllerConfig::default().with_initial_config(station_config),
).unwrap();
let wifi_interface = interfaces.station;
```

要点：`esp_radio::init()` 返回 `esp_radio::Controller<'static>`，
一般用 `mk_static!` / `StaticCell` 提升为 `'static` 再交给后台任务；
`wifi::new` 返回 `(WifiController, Interfaces)`，STA 口是 `interfaces.sta`
（旧）/ `interfaces.station`（新），AP 口对应 `interfaces.ap`。
旧版里接口类型叫 `WifiDevice` / `WifiStaDevice` / `WifiApDevice`，
0.18 里叫 `esp_radio::wifi::Interface`——**不要混用这两个名字**。

### 8.2 `WifiController`：方法与阻塞 / 异步

`esp_radio::wifi::WifiController<'d>`，已在 0.16 文档核实：

| 方法 | 性质 |
|---|---|
| `set_config(&ModeConfig / &Config)` | 设模式；传 `ModeConfig::None` / `Config` 的空变体可关掉 STA+AP |
| `start()` / `start_async().await` | 非阻塞 / 等到启动完成；之后用 `is_started()` 查 |
| `stop()` / `stop_async().await` | 同上 |
| `connect()` / `connect_async().await` | `connect()` 本身**不阻塞**，要轮询 `is_connected()`；`connect_async()` 等到连上才返回 |
| `disconnect()` / `disconnect_async().await` | 同上 |
| `scan_with_config(ScanConfig)` / `scan_with_config_async(ScanConfig).await` | 阻塞扫描 / 异步扫描；`ScanConfig::default().with_max(10)` |
| `wait_for_event(WifiEvent)` | 等一个或多个事件，返回发生的事件 |
| `is_started()`、`is_connected()`、`capabilities()`、`get_rssi()`、`set_protocol(Protocol)`、`set_power_saving(...)` | 查询 / 设置；`get_rssi()` 只在 STA 或 AP-STA 模式、连上之后用，AP-only 时报 `WifiError::Unsupported`；`set_protocol` 默认 `WIFI_PROTOCOL_11B \| WIFI_PROTOCOL_11G \| WIFI_PROTOCOL_11N`，须在 `set_config` 之后调 |

注意：Book 里还出现 `controller.scan_async(&scan_config).await`、
`controller.wait_for_disconnect_async().await`、
`esp_radio::wifi::sta_state()` + `WifiStaState::Connected`，这些在 0.16 的
`WifiController` 页里**没有列出**，属于新版 / 教程写法，落版本时以你锁定的
版本文档为准，不要当成 0.16 的 API。

### 8.3 STA / AP 模式

模式枚举（版本不同，两个名字）：

```rust
// 0.16–0.17
ModeConfig::Client(ClientConfig) / ModeConfig::AccessPoint(AccessPointConfig)
// 1.0.0-beta.0（C3 预览页，已打开）
Config::Station(StationConfig)
Config::AccessPoint(AccessPointConfig)
Config::AccessPointStation(StationConfig, AccessPointConfig)
Config::EapStation(EapStationConfig) // 需 wifi-eap feature，企业 Wi-Fi
```

- **STA**：`StationConfig::default().with_ssid(SSID).with_password(PASSWORD.into())`
  （Book 写法）；旧版是 `ClientConfig::default().with_ssid(...).with_password(...)` +
  `ModeConfig::Client(...)`。连上后 `embassy-net` 再 DHCP 取 IP。
- **AP（Soft-AP）**：`AccessPointConfig` 字段（0.16 C3 页已核实）：
  `ssid`、`ssid_hidden`、`channel`、`secondary_channel`、`protocols`、
  `auth_method`、`password`、`max_connections`、`dtim_period`（1~10）、
  `beacon_timeout`（`unstable` feature）。
- **AP-STA**：同时做 AP 又连上游 AP；AP 口通常配静态 IP，STA 口走 DHCP。

`WifiEvent`（0.16 页，已核实，`#[non_exhaustive]`，共 45 个变体）：
`WifiReady`、`ScanDone`、`StaStart`、`StaStop`、`StaConnected`、
`StaDisconnected`、`ApStaConnected`、`ApStaDisconnected` 等。
典型断线重连就是 `controller.wait_for_event(WifiEvent::StaDisconnected).await`
后再 `connect_async()`。

### 8.4 断线重连任务（异步版，典型写法）

```rust
#[embassy_executor::task]
async fn connection(mut controller: WifiController<'static>) {
    loop {
        // 按版本二选一：set_config(Client/Station) 后 start_async + connect_async
        // 或按 Book：controller.connect_async().await 后
        // controller.wait_for_disconnect_async().await
        if controller.connect_async().await.is_ok() {
            controller.wait_for_event(WifiEvent::StaDisconnected).await;
        }
        Timer::after(Duration::from_secs(5)).await;
    }
}
```

---

## 9. BLE 与 ESP-NOW（C3）

### 9.1 BLE：`BleConnector` + trouble-host

`esp-radio` 的 BLE 部分**只提供 HCI 传输**，不提供 GATT 本身：

- `esp_radio::ble::BleConnector`（需 `ble` feature + `coex`）：
  实现 `bt-hci` 的 HCI transport，底层是 NimBLE Porting Layer。
- 上层用 **`trouble-host`**（纯 Rust、no_std、Embassy 原生）：
  `trouble_host::prelude::*`，`ExternalController<BleConnector, N>`
  （`bt-hci` 的 `ExternalController`，`N` 为包池大小）、
  `HostResources` / `Host` / `Runner`。
- GATT 用宏定义：

```rust
#[gatt_server]
struct Server { sensor_service: SensorService }

#[gatt_service(uuid = "a9c81b72-0f7a-4c59-b0a8-425e3bcf0a0e")]
struct SensorService {
    #[characteristic(uuid = "13c0ef83-09bd-4767-97cb-ee46224ae6db", read, notify)]
    sensor_data: u8,
    #[characteristic(uuid = "c79b2ca7-f39d-4060-8168-816fa26737b7", write, read)]
    sensor_settings: bool,
}
```

- 广播：`trouble_host::advertise::{AdStructure, Advertisement}`，如
  `AdStructure::Flags(LE_GENERAL_DISCOVERABLE | BR_EDR_NOT_SUPPORTED)`、
  `AdStructure::ServiceUuids128(&[…])`、
  `AdStructure::CompleteLocalName("…")`。
- 典型规模：`CONNECTIONS_MAX: usize = 1`、`L2CAP_CHANNELS_MAX: usize = 1`
  （C3 单连接例子）；后台要同时跑 `Runner`（host 事件循环）和广播 /
  连接处理循环，常用 `embassy_futures::join::join` 或 `select` 拼。
- **未核实**：`trouble_host::new(…)` / `Host::new(…)` / `Runner::run()`
  的确切签名在本次打开的页面里没有完整给出（只核实了 `prelude` 导出
  `ExternalController`、`Runner`、`gatt_server` / `gatt_service` 宏），
  写代码时以你锁定的 `trouble-host` 版本文档为准。L2CAP CoC 在
  `trouble-host` 里当时不稳定，不要当成已可用的 API。

### 9.2 ESP-NOW

`esp_radio::esp_now::EspNow<'d>`，需 `esp-now` feature（依赖 `wifi`），且
**整个 API 标记为 `unstable`**，需 `unstable` feature，无 SemVer 保证。

已核实（0.16 页）：

- `split()`：拆成 manager / sender / receiver 三部分。
- `add_peer(…)` / `remove_peer(…)` / `modify_peer(…)` / `get_peer(…)` /
  `fetch_peer(…)` / `is_peer_known(…)` / `peer_count()`；发送前必须先 `add_peer`。
- `send(…)`：发给指定 peer；`receive(…)`：异步收，返回的 future
  **不要在 ready 前 drop**，否则会拿到错误的发送状态。
- `set_channel(…)`：只在不用 AP/STA 时用；`set_primary_master_key(…)`、
  `get_version()`、`set_rate(…)`、wake window 设置。
- 默认已在 STA 口上加了一个广播 peer；单包最大 **250 字节**；
  收发双方须在同一 Wi-Fi 信道。

---

## 10. `esp-rtos` —— 给无线和 Embassy 的调度器

原文一句话：

> Scheduler implementation for esp-radio, embassy support for `esp-hal`.

中文：面向 `esp-radio` 的调度器实现，同时为 `esp-hal` 提供 Embassy 支持。

### 翻译详解

`esp-rtos` 是 `esp-radio` 默认搭售的调度后端：它实现了
`esp-radio-rtos-driver` 定义的调度与运行时接口，让无线协议栈
能在后台可靠地跑起来。`esp-generate` 在你勾选 Wi-Fi 等模板选项时，
会自动把 `esp-rtos` 加进来，一般不用手动再配一遍。

对 Embassy 工程来说，它同时承担了时间驱动的角色：
Embassy 的 `Timer::after(…).await`、`Timer::at(…).await`
这类异步延时，最终要落到一个能被调度的定时器上，
`esp-rtos` 就是把 `esp-hal` 的定时器和 Embassy 的时间驱动接起来的那层。

老教程里常见的 `esp-hal-embassy` 是更早的接法，
在 1.x 的新工程里请优先按 `esp-rtos` 的路线走，
不要把两套调度器混在一个工程里。

### 启动

```rust
use esp_hal::timer::timg::TimerGroup;
let timg0 = TimerGroup::new(peripherals.TIMG0);
esp_rtos::start(timg0.timer0); // 0.2.0 文档写法
// Rust on ESP Book / 新版写法：
// esp_rtos::start(timg0.timer0, sw_int.software_interrupt0);
// 其中 sw_int = SoftwareInterruptControl::new(peripherals.SW_INTERRUPT);
```

- `esp_rtos::start(…)`：启动调度器；`start_with_idle_hook(…)`：
  带自定义 idle hook；`start_second_core(…)` /
  `start_second_core_with_stack_guard_offset(…)`：
  **只给双核芯片，C3 是单核，不要调**。
- 顺序：`esp_hal::init` → `esp_rtos::start` → `esp_radio::init` → `wifi::new`。
- 配置项（0.2.0 页）：`ESP_RTOS_CONFIG_TICK_RATE_HZ`（默认 100 Hz，
  不稳定配置）等。
- Feature（0.2.0 页）：`embassy`（Embassy 集成：时间驱动 + 执行器）、
  `esp-radio`、`esp-alloc`（`esp-radio` 需要堆分配；不开它就要自己提供
  `malloc_internal` / `free_internal`）、`log-04` / `defmt`，
  芯片选 `esp32c3`。

### 和 embassy-executor 的关系

- 写 async 要开 `embassy` feature，入口用 **`#[esp_rtos::main]`**，
  它会创建一个 `esp_rtos::embassy::Executor`（主线程上的 thread-mode executor），
  并把 `main` 体作为一个 async 任务 spawn。
- 任务本身用 `embassy-executor` 的 `#[embassy_executor::task]` 宏；
  文档明确：**不要给 `embassy-executor` 开任何 `arch-*` feature**。
- 不要把 `#[esp_rtos::main]` 和旧 `esp-hal-embassy` 的
  `#[embassy_executor::main]` 混用——1.x 线用前者。

---

## 11. `embassy-net` 与 Embassy 异步链路（C3）

### 11.1 Stack / Runner / 资源

```rust
use embassy_net::{Config, DhcpConfig, StackResources, Runner};
use static_cell::StaticCell;

static RESOURCES: StaticCell<StackResources<3>> = StaticCell::new();
let (stack, runner) = embassy_net::new(
    wifi_interface,                 // esp-radio 的 STA/AP 接口
    Config::dhcpv4(DhcpConfig::default()), // 或 Config::dhcpv4(Default::default())，看版本
    RESOURCES.init(StackResources::new()),
    seed,                           // u64，一般用 RNG 拼出来
);
```

- `embassy_net::new(device, config, resources, seed) -> (Stack, Runner)`。
  `Runner` 是引擎，必须单独开一个任务跑 `runner.run().await` 到底
  （`net_task`），否则一切网络都不工作；`Stack` 是轻量 `Copy` 句柄，
  可以传值给各个任务去开 socket / 查 IP。
- `StackResources<3>` 的 `3` 是 socket 槽位数：DHCP 占 1、DNS 占 1、
  应用再占 1；多开 socket 就要加大。
- `Config`：`Config::dhcpv4(…)`（DHCPv4）或
  `Config::ipv4_static(StaticConfigV4 { address: Ipv4Cidr::new(Ipv4Address::new(192, 168, 1, 10), 24), dns_servers: Vec::new(), gateway: Some(Ipv4Address::new(192, 168, 1, 1)) })`。
  AP 口一般用后者。
- `Stack` 方法（docs.rs `Stack` 页已核实）：`is_link_up()`、
  `is_config_up()`、`wait_link_up()` / `wait_link_down()`、
  `wait_config_up()` / `wait_config_down()`（DHCP 拿不到 IP 时可能永不返回，
  调用方自己加超时）、`config_v4()` / `config_v6()`、
  `set_ipv4_config(…)`、`dns_query(…)`、`hardware_address()`、
  `join_multicast_group(…)` / `leave_multicast_group(…)`。

### 11.2 TcpSocket / UdpSocket / DnsSocket

**`embassy_net::tcp::TcpSocket`**（需 `tcp` feature）：

```rust
let mut rx_buffer = [0u8; 1024];
let mut tx_buffer = [0u8; 1024];
let mut socket = TcpSocket::new(stack, &mut rx_buffer, &mut tx_buffer);
socket.set_timeout(Some(Duration::from_secs(10)));
socket.connect((Ipv4Address::new(192, 168, 1, 1), 8000)).await?; // 客户端
socket.accept(80).await?;                                          // 服务端：没有 TcpListener，
// 每个 TcpSocket 自己 accept() 进入监听；想同时接多个客户端就多建几个 socket
let n = socket.read(&mut buf).await?;
socket.write_all(response).await?; // embedded-io-async::Write
socket.flush().await?;
socket.close(); // 只关写半边；abort() 是强制关
```

还可 `split()` 成 `TcpReader` / `TcpWriter`，各有 `read` / `write` / `flush`。
`read` 返回 `Ok(0)` 表示对端已关。

**`embassy_net::udp::UdpSocket`**：`UdpSocket::new(stack, &mut rx_meta,
&mut rx_buffer, &mut tx_meta, &mut tx_buffer)`（`rx_meta` / `tx_meta`
是 `PacketMetadata` 数组），`bind(endpoint)`（端口传 `0` 会自动分配），
`recv_from(&mut buf).await -> (usize, IpEndpoint)`，
`send_to(&buf, remote_endpoint).await`。

**DNS**：Book 的 HTTP 例子用 `DnsSocket::new(stack)` 配合 `reqwless` 的
`HttpClient`；`Stack` 本身也有 `dns_query(…)`。另外 Book 里 `reqwless`
用的不是裸 `TcpSocket`，而是 `embassy_net::tcp::client::TcpClient::new(stack,
mk_static!(TcpClientState<1, 1500, 1500>, …))`——这是 `reqwless` 客户端的
配套写法，**未在 docs.rs 的 `TcpSocket` 页核实**，用 `reqwless` 时再对版本。

### 11.3 最小异步 Wi-Fi + TCP 骨架（代码草图）

以下按 Book + Molloy 的结构综合，名字都是上面核实过的；
`Config` / `Interfaces` 的字段名按你锁定的版本二选一（见 8.1 节）：

```rust
#![no_std]
#![no_main]
use embassy_executor::Spawner;
use embassy_net::{Config, DhcpConfig, Runner, Stack, StackResources};
use embassy_time::{Duration, Timer};
use esp_radio::wifi::{WifiController, WifiEvent};
use static_cell::StaticCell;

#[esp_rtos::main]
async fn main(spawner: Spawner) -> ! {
    let peripherals = esp_hal::init(esp_hal::Config::default());
    let timg0 = esp_hal::timer::timg::TimerGroup::new(peripherals.TIMG0);
    esp_rtos::start(timg0.timer0); // 新版可能要加 software interrupt 参数，见 10 章
    // esp_alloc::heap_allocator!(...); // esp-radio 需要堆，先配好
    let radio = &*mk_static!(esp_radio::Controller<'static>, esp_radio::init().unwrap());
    let (controller, interfaces) =
        esp_radio::wifi::new(radio, peripherals.WIFI, Default::default()).unwrap();
    // Book 1.x 写法：wifi::new(peripherals.WIFI,
    //   ControllerConfig::default().with_initial_config(Config::Station(...)))

    static RESOURCES: StaticCell<StackResources<3>> = StaticCell::new();
    let seed = /* RNG 拼出的 u64 */;
    let (stack, runner) = embassy_net::new(
        interfaces.sta, // 新版 interfaces.station
        Config::dhcpv4(DhcpConfig::default()),
        RESOURCES.init(StackResources::new()),
        seed,
    );
    spawner.spawn(connection(controller)).unwrap(); // 版本不同 spawn 形态见 2.5 节
    spawner.spawn(net_task(runner)).unwrap();
    stack.wait_config_up().await;
    // 这里 stack 已拿到 IP，可再 spawn 一个用 TcpSocket 的任务
    loop { Timer::after(Duration::from_secs(60)).await; }
}

#[embassy_executor::task]
async fn connection(mut controller: WifiController<'static>) {
    loop {
        if controller.connect_async().await.is_ok() {
            controller.wait_for_event(WifiEvent::StaDisconnected).await;
        }
        Timer::after(Duration::from_secs(5)).await;
    }
}

#[embassy_executor::task]
async fn net_task(mut runner: Runner<'static, esp_radio::wifi::Interface<'static>>) -> ! {
    runner.run().await // 旧版类型是 Runner<'static, WifiDevice<'static>>
}
```

注意：`net_task` 的 `Runner` 泛型在 0.16–0.17 是 `WifiDevice<'static>`，
0.18 是 `Interface<'static>`；`spawner.spawn(…)` 在有的版本返回 `Result`
要 `.unwrap()`，有的版本直接返回 token 或用 `must_spawn`——编译器报错会
直接告诉你该用哪种。

---

## 12. `esp-bootloader-esp-idf` —— 启动描述和 OTA

原文一句话：

> Offers additional support for the ESP-IDF 2nd stage bootloader, including OTA.

中文：为 ESP-IDF 二级 bootloader 提供额外支持，包括 OTA。

### 翻译详解

现在的 `no_std` 工程大多仍用 ESP-IDF 的二级 bootloader 来启动，
这个 bootloader 要求固件里嵌一小块应用描述信息（版本、名称、
编译时间等元数据）。`esp-bootloader-esp-idf` 就是提供这块信息的 crate，
用法是在 crate 根部调用一次宏：

```rust
esp_bootloader_esp_idf::esp_app_desc!();
```

漏了它会发生什么，是新手最容易卡住的地方之一：
程序能编译、能烧录，但板子就是不启动，而且编译阶段没有任何报错，
因为缺的只是一块启动时要读的元数据。从 `espflash` v4 起，
预置的 ESP-IDF bootloader 对这块描述的要求更严格，
所以官方示例里这一行是必备的，不要删。

它还提供 OTA（空中升级）相关的支持：配合分区表里的 OTA 数据区，
可以实现“下载新固件到备用分区、下次启动切过去”的升级流程，
具体分区和流程要按 ESP-IDF 的 OTA 文档来配，不是只加一个依赖就自动有的。
双 OTA 分区表见第 6 章。

---

## 13. 原来完全没进翻译单的 crate（C3 用户一定会碰到）

| crate | 一句话用途 / C3 何时需要 | Cargo.toml（C3） |
|---|---|---|
| `esp-nvs` | 裸机 NVS 键值持久化（Wi-Fi 配置、计数器、校准） | 见第 7 章；`features = ["esp32c3"]` |
| `embassy-net` | Embassy 的 TCP/IP 栈（底层 smoltcp），Wi-Fi 联网必备 | `embassy-net = { version = "0.9", features = ["tcp", "udp", "dhcpv4", "dns", "medium-ethernet"] }`（feature 名未逐项核实，以 docs.rs 为准） |
| `embassy-executor` | Embassy 异步执行器，`#[embassy_executor::task]` + `Spawner` | `embassy-executor = "0.10"`；**注意：esp-rtos 文档明确说不要自己开 `arch-*` feature**，用 esp-rtos 提供的执行器时以 esp-rtos 文档为准 |
| `embassy-time` | `Timer::after(Duration::from_secs(1)).await`、超时、周期任务 | `embassy-time = "0.5"`（无芯片 feature；时间驱动由 `esp-rtos` 提供） |
| `embassy-sync` | 任务间 `Mutex` / `Channel` / `Signal` / `PubSubChannel`，代替 `std::sync` | `embassy-sync = "0.8"` |
| `embassy-futures` | `select`（先完成者胜）/ `join`（等全部完成）等 future 组合子 | `embassy-futures = "0.1"`（版本号未核实） |
| `trouble-host`（TrouBLE） | 纯 Rust BLE Host（GATT server/client），配 `esp-radio` 的 `BleConnector` | `trouble-host = { version = "0.6", features = ["default-packet-pool-mtu-255"] }`（0.6 为 2026-04 实测版；另有 0.8 的升级记录，**未核实最新版**） |
| `embedded-hal` / `embedded-hal-async` | 外设 trait 标准（GPIO / I2C / SPI / UART 的 `embedded-hal` trait，`esp-hal` 驱动已实现；写跨芯片驱动时用） | `embedded-hal = "1.0"`；`embedded-hal-async = "1.0"`（具体小版本未核实） |
| `embedded-io` / `embedded-io-async` | `no_std` 版 `Read` / `Write` trait | `embedded-io = "0.6"`（版本未核实） |
| `embedded-storage` | `ReadNorFlash` / `NorFlash` / `ReadStorage` / `Storage` trait，`esp-storage` 的实现对象 | `embedded-storage = "0.3"`（版本未核实） |
| `smoltcp` | 底层 TCP/IP 协议栈；**C3 用户一般不直接依赖**，经 `embassy-net` 间接使用 | 不直接写；`embassy-net` 会带 |
| `esp-config` | 编译期配置系统（`ESP_*_CONFIG_*` 环境变量，如 `ESP_HAL_CONFIG_*`） | 由 esp-hal 传递，一般不直接依赖；版本未核实 |
| `esp-sync` | esp 的同步原语（critical-section 等），esp-hal/esp-radio 的底层依赖 | 由 esp-hal 传递，一般不直接依赖；`features = ["esp32c3"]` 仅在直接依赖时 |
| `esp-preempt` | 线程与线程感知同步原语，主要给 `esp-radio` 用 | 由 esp-radio 传递，一般不直接依赖 |
| `esp-hal-proc-macros` | 过程宏：`#[main]`、`#[handler]`、`#[ram]`、`#[esp_rtos::main]`；**crates.io 上叫 `esp-hal-procmacros`，一般不要直接依赖**，由 `esp-hal` / `esp-rtos` 重导出：`esp_hal::{main, handler, ram}`、`esp_rtos::main` | 不直接写；`#[handler(priority = esp_hal::interrupt::Priority::Priority2)]` 为中断处理函数写法（已核实） |
| `esp-riscv-rt` | RISC-V 最小启动 / 运行时（向量表、复位入口），C3 的底层 | 由 esp-hal 传递，一般不直接依赖 |

---

## 14. 辅助 crate 速查表（翻译版，C3 视角）

以下来自《Rust on ESP Book》官方表，中文为翻译，稳定性为官方标注。
C3 上不存在的已在第 1 章列出，不再重复。

| crate | 中文含义 | 稳定性 |
|---|---|---|
| `esp-alloc` | 内存分配工具 | Unstable |
| `esp-backtrace` | 调用栈支持 | Unstable |
| `esp-bootloader-esp-idf` | ESP-IDF 二级 bootloader 支持，含 OTA | Unstable |
| `esp-build` | 构建脚本里用的构建工具，供 `esp-hal` 及相关包使用 | Unstable |
| `esp-config` | 配置系统 | Unstable |
| `esp-hal` | 面向所有 Espressif ESP32 设备的裸机 `no_std` HAL | Stable* |
| `esp-hal-proc-macros` | 供 `esp-hal` 家族使用的过程宏 | Unstable |
| `esp-lp-hal` | 面向部分芯片上低功耗 / 超低功耗核的裸机 HAL（**C3 无 LP 核，与 C3 无关**） | Unstable |
| `esp-metadata` | Espressif 设备元数据，主要供构建脚本使用 | Unstable |
| `esp-preempt` | 线程与线程感知同步原语，主要供 `esp-radio` 使用 | Unstable |
| `esp-println` | 打印与日志功能 | Unstable |
| `esp-riscv-rt` | 面向 Espressif RISC-V CPU 的最小启动 / 运行时 | Unstable |
| `esp-rom-sys` | ROM 代码支持 | Unstable |
| `esp-rtos` | `esp-radio` 的调度器实现，`esp-hal` 的 Embassy 支持 | Unstable |
| `esp-sync` | 面向 Espressif 设备的同步原语 | Unstable |
| `esp-storage` | 存储工具 | Unstable |
| `esp-nvs` | ESP-IDF 兼容的 NVS 键值存储 | Unstable |
| `esp-radio` | Wi-Fi、BLE、IEEE 802.15.4、ESP-NOW 功能（**C3 无 802.15.4**） | Unstable |
| `xtensa-lx` | Xtensa LX 处理器与外设的底层访问（**C3 是 RISC-V，与 C3 无关**） | Unstable |
| `xtensa-lx-rt` | 面向 Xtensa LX CPU 的最小启动 / 运行时（**C3 无关**） | Unstable |

\* `esp-hal` 的各个外设驱动稳定性并不一致，查某个外设时请看
该外设在文档里的稳定性标注，不要只看 crate 级别的 Stable。
1.0 的稳定集只有 GPIO、UART、SPI master、I2C master。

---

## 15. 示例索引（只列已核实文件名的，C3 相关）

**怎么判定一个示例能不能在 C3 上跑**：不要只看文件名。
每个示例文件头有 `//% CHIPS: esp32 esp32c2 esp32c3 …` 行，
`examples/Cargo.toml` 里有按芯片分的 feature，**C3 能不能编以它为准**。
构建命令（README 已核实）：`cargo xtask build examples esp-hal --chip esp32c3 all`；
单跑：`cargo xtask run embassy_hello_world --chip=esp32c3`。
官方示例默认按 DevKit-C 布线，**引脚必须按你的 C3 板子重对**。

### `esp-hal` 仓库 `examples/`

| 已核实示例 | 演示内容 | C3 注意 |
|---|---|---|
| `hello_world.rs` | 最小阻塞程序：初始化 + 打印 | C3 可用，改 `--chip esp32c3` 即可 |
| `embassy_hello_world.rs` | Embassy 异步入口 + `Timer::after` 闪灯 | C3 可用 |
| `embassy_i2c.rs` | 异步 I2C | 旧版 Cargo 要求 `async + embassy + embassy-executor-thread`；1.x 的 feature 名以当前 `examples/Cargo.toml` 为准 |
| `embassy_spi.rs` | 异步 SPI | 同上 |
| `embassy_serial.rs` | 异步串口（UART echo，旧名） | **1.x 是否改名未核实**，用前先在 `examples/` 里确认文件名 |
| `embassy_wait.rs` | `embassy_time::Timer` 等待 | C3 可用 |
| `examples/qa/` 下的 `sleep_timer` 等 | QA 用例 | 非教程，按需参考 |

另：`hil-test/tests/` 非示例，是硬件在环测试，可作 API 参考：
`gpio.rs`、`uart.rs`、`uart_async.rs`、`spi_full_duplex.rs`、
`spi_half_duplex_read.rs`、`spi_half_duplex_write.rs`、`i2s.rs`、`qspi.rs`。
测试头有 `//% CHIPS: esp32 esp32c2 esp32c3 …` 与
`//% FEATURES: unstable embassy`，C3 跑 UART/SPI/GPIO 测试需按头文件接线
（如 C3-DevKit 上 GPIO2–GPIO3 短接）。

**C3 不兼容 / 需改芯片 feature 的示例**：凡用到 PSRAM、Ethernet、
USB-OTG、摄像头、DAC、触摸、MCPWM、SDMMC、SPI3、第二核 /
`start_second_core`、LP 外设的，一律不能在 C3 上原样跑。

### `esp-radio` 仓库 `examples/wifi/`

| 示例 | 演示内容 |
|---|---|
| `embassy_dhcp` | STA 连 Wi-Fi + DHCP 取 IP（embassy-net） |
| `embassy_access_point` | 开热点 AP |
| `embassy_access_point_with_sta` | AP + STA 共存 |
| `embassy_coex` | Wi-Fi 与 BLE 共存 |
| `embassy_sntp` | Wi-Fi + SNTP 对时 |

另有 `qa-test/src/bin/` 下的 `embassy_wifi_bench`、`embassy_wifi_csi`、
`embassy_wifi_stress`、`wifi_survives_ble_drop`（测试用，非教程）。
ESP-NOW 官方例子的确切名字**未核实**，不要凭印象写。

### BLE / HTTP

| 例子 | 一句话 |
|---|---|
| Rust on ESP Book `intro/http-client`（`http-client.rs`） | STA + DHCP + `reqwless` HTTP GET（`TcpClient` + `DnsSocket`） |
| implrust `wifi/embassy/connecting-wifi`、`wifi/embassy/http-request`、`wifi/web-server/wifi` | STA 连接 / HTTP 请求 / Web Server（`accept(80)` + 固定 HTML） |
| implrust `bluetooth/trouble/ble-module` | trouble-host GATT Server：传感器读 / 通知 + 设置读 / 写两个特征 |
| `trouble` 仓库 `examples/esp32` | ESP32 系列的 BLE GATT 示例目录（C3 是否逐个可用未核实） |
| Derek Molloy《An Async Wi-Fi Web Server on the ESP32-C3 with Embassy and no_std Rust》（2026-06） | C3 异步 Wi-Fi 的端到端对照：esp-hal ~1.1、esp-radio 0.18、esp-rtos 0.3、embassy-executor 0.10、embassy-net 0.9.1、embassy-time 0.5，SuperMini 板、GPIO8 低电平 LED |

### `esp-nvs` / `esp-storage`

在已打开的 docs.rs / docs.espressif 页面中**未找到独立示例目录**，
只有 docs.rs 首页的 `Nvs::new` 代码片段（见第 7 章）；示例清单记为未核实。

---

## 16. 新手最容易踩的坑（中文总结，C3 版）

1. **芯片 feature 不一致**：`esp-hal` 是 `esp32c3`，`esp-println`
   却是 `esp32c6`，编译报错信息往往不在点子上，先对齐所有芯片 feature。
2. **漏了 panic 处理器**：`no_std` 二进制没有默认 panic 处理器，
   不引 `esp-backtrace`（或 `panic-probe`）根本编不过。
3. **漏了 `esp_app_desc!()`**：能编译、能烧录、板子不启动，
   先检查 `esp-bootloader-esp-idf::esp_app_desc!();` 有没有写。
4. **堆初始化太晚**：Wi-Fi、网络、显示库在用堆之前，
   `heap_allocator!` 必须已经执行过，否则分配失败。
5. **输出通道不一致**：`esp-backtrace` 和 `esp-println` 一个走 UART、
   一个走 USB-Serial-JTAG，panic 信息会发到你没看的那条线上。
6. **按错芯片的文档**：docs.rs 默认按一个芯片构建，
   去 esp-rs 官网按你的芯片选文档，C3 和 C6 的外设清单不一样。
7. **把 `esp-storage` 当文件系统**：它只是 Flash 裸读写，
   要文件或键值存储请在它上面再接 littlefs / `esp-nvs`。
8. **照抄老教程的 `esp-wifi` / `esp-hal-embassy`**：1.x 里无线是
   `esp-radio`、调度是 `esp-rtos`，老名字对应的是旧版本线，
   混用会导致依赖解析失败或功能对不上。
9. **在 C3 上用 PSRAM / DAC / 802.15.4 / 经典蓝牙**：C3 没有这些，
   `psram_allocator!`、DAC、Zigbee/Thread、经典蓝牙都不要写进 C3 工程。
10. **异步任务里不 `.await` 的长计算**：C3 单核、Embassy 协作式调度，
    长计算不让出 CPU 会饿死 Wi-Fi / BLE，表现为断线 / 丢包 / 超时。
11. **ADC2 + Wi-Fi 一起用**：开 Wi-Fi 后 ADC2（GPIO5）读数不可靠，
    做 Wi-Fi 传感器优先用 ADC1。
12. **把 Flash 脚 / strapping 脚当普通 GPIO**：GPIO11–GPIO17 接 Flash
    永远别用；GPIO2 / GPIO8 / GPIO9 复位时别拉错电平。
13. **照 C API 写 `esp-nvs`**：no_std `esp-nvs` 没有 `commit`、
    没有 `get_i32` / `put_i32`，是泛型 `get::<T>` / `set`，写即落盘。
14. **版本混用**：`esp-radio` 0.17 和 0.18 的 Wi-Fi API 不一样，
    `interfaces.sta` 和 `interfaces.station` 不要混用，成套升级。

---

## 17. 推荐的学习顺序（给 ESP32-C3 + Rust `no_std`）

1. `esp-generate --chip esp32c3` 生成骨架，先跑通 Blink（先确认你板子的 LED 脚）；
2. 加 `esp-println`，把串口日志跑通，再看 panic 时的 backtrace；
3. 加 `esp-alloc`，试 `Vec` / `String`，理解堆大小和 `HEAP.stats()`；
4. 试一个 I2C / SPI 外设，理解外设单例和 `embedded-hal` trait；
5. 加 `esp-storage` / `esp-nvs`，做一个掉电不丢的配置（开机计数器）；
6. 上 Embassy：先跑 `embassy_hello_world`，理解 `Timer` / 任务 /
   `Channel`，再试 GPIO `wait_for_*`；
7. 最后再上 `esp-radio` + `esp-rtos` + `embassy-net`，做 Wi-Fi 联网
   （STA + DHCP + TCP），再试 BLE（trouble-host）。
   无线是最重的一步，放在最后可以少走很多弯路。

---

## 未核实清单（下次修订前补）

1. RMT / I2S / TWAI / RNG / AES / SHA / RSA / HMAC / TSens 的具体类型与方法名；
   SPI 固有 `*_async` 方法名；ADC 是否有异步版；`embassy_serial.rs` 在 1.x 的现名；
   官方 `examples/` 的完整文件名清单（需本地 clone 后以 `examples/Cargo.toml` 为准）；
   GPIO 中断注册宏的确切名字。
2. `embassy-net` 最新版底层是 `smoltcp` 还是 `xarxa`（C3 官方例子用的是哪个版本）。
3. `controller.scan_async()`、`wait_for_disconnect_async()`、`sta_state()` /
   `WifiStaState` 在 0.16 的存在性。
4. `trouble_host::new()` / `Host` / `Runner` 的确切构造签名；`DnsSocket`
   除 `new(stack)` 外的方法；`TcpClient` / `TcpClientState` 的版本适用范围。
5. no_std 下 espflash 默认是否刷分区表、以及 `partitions.csv` 的传入方式；
   `esp-storage` 当前版是否仍需 `low-level` 才能配 `esp-nvs`；
   `esp-backtrace` 最新版是否保留 `exception-handler`；
   `embassy-executor` 与 `esp-rtos` 混用时 `arch-*` feature 的正确写法；
   `embassy-net` / `embassy-futures` / `embedded-io` 等的确切最新版本号；
   `esp-nvs` 的 `Get` / `Set` trait 是否需显式 `use` 才能调用 `get` / `set`。

---

## 资料来源（原文链接）

- 《Rust on ESP Book》辅助 crate 总览：
  https://docs.espressif.com/projects/rust/book/introduction/ancillary-crates.html
- 《Rust on ESP Book》异步章节：
  https://docs.espressif.com/projects/rust/book/application-development/async.html
- 《Rust on ESP Book》堆分配章节：
  https://docs.espressif.com/projects/rust/book/application-development/alloc.html
- 《Rust on ESP Book》HTTP 客户端：
  https://docs.espressif.com/projects/rust/no_std-training/03_6_http_client.html
- `esp-hal` 官方文档（按芯片选）：
  https://docs.esp-rs.org/esp-hal
- `esp-hal` 1.0.0 C3 文档：
  https://docs.espressif.com/projects/rust/esp-hal/1.0.0/esp32c3/esp_hal/index.html
- `esp-hal` 支持矩阵：
  https://docs.espressif.com/projects/rust/esp-hal/main/index.html
- `esp-hal` 官方示例：
  https://github.com/esp-rs/esp-hal/tree/main/examples
- `esp-alloc` 文档：
  https://docs.espressif.com/projects/rust/esp-alloc/0.9.0/esp_alloc/index.html
- `esp-backtrace` 文档：
  https://docs.rs/crate/esp-backtrace/latest
- `esp-println` 文档：
  https://docs.rs/crate/esp-println/latest
- `esp-storage` 0.8 文档：
  https://docs.espressif.com/projects/rust/esp-storage/0.8.0/esp_storage/struct.FlashStorage.html
- `esp-nvs` 文档：
  https://docs.rs/esp-nvs/latest/esp_nvs/
- `esp-radio` 0.16 Wi-Fi 文档：
  https://docs.espressif.com/projects/rust/esp-radio/0.16.0/esp32c3/esp_radio/wifi/struct.WifiController.html
- `esp-radio` 1.0.0-beta.0 C3 预览：
  https://preview-docs.espressif.com/projects/rust/esp-radio/1.0.0-beta.0/esp32c3/esp_radio/wifi/enum.Config.html
- `esp-rtos` 0.2 / 0.3 文档：
  https://docs.espressif.com/projects/rust/esp-rtos/0.2.0/esp32/esp_rtos/index.html
- `embassy-net` Stack 文档：
  https://docs.rs/embassy-net/latest/embassy_net/struct.Stack.html
- `trouble-host` 文档：
  https://docs.rs/trouble-host/0.3.0/trouble_host/prelude/index.html
- C3 数据手册：
  https://documentation.espressif.com/esp32-c3_datasheet_en.html
- C3 异步 Wi-Fi 实例（2026-06）：
  https://derekmolloy.ie/an-async-wi-fi-web-server-on-the-esp32-c3-with-embassy-and-no_std-rust-9/
- implrust ESP32 教程：
  https://esp32.implrust.com/wifi/embassy/connecting-wifi.html
