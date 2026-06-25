# CherryUSB 移植总结

> HC32F4A0 + DWC2(内部FS PHY) + Host-Only + 裸机
> 在项目作用是再BOOT初始化枚举U盘，读取U盘里面的BIN文件

---

## 编译了哪些 CherryUSB 源文件（共6个）

```
USB/cherryusb/
├── core/usbh_core.c           # USB主机框架
├── class/hub/usbh_hub.c       # Hub处理
├── class/msc/usbh_msc.c       # MSC类驱动
├── port/usb_hc_dwc2.c         # DWC2控制器驱动（CherryUSB官方）
├── port/usb_glue_hc.c         # HC32板级适配（你自己写）
└── osal/usb_osal_bare.c       # 裸机OSAL（你自己写）
```

**封装层**（你自己写的，封装CherryUSB给上层调用）：

```
USB/usb_drv.h + usb_drv.c     # usb_host_start/ wait/ read_sector
USB/cfg_usb.h + cfg_usb.c     # 引脚定义 + IRQ
```

---

## 关键配置

**cherryusb_config.h**（Host-Only 裁剪）：

```c
#define CONFIG_USBHOST_MAX_RHPORTS     1      // 只有RootHub
#define CONFIG_USBHOST_MAX_MSC_CLASS   2      // 最多2个U盘
#define CONFIG_USBHOST_MSC_TIMEOUT     5000   // MSC读写超时
#define CONFIG_USB_ALIGN_SIZE          4      // 4字节对齐
/* #define CONFIG_USB_DCACHE_ENABLE */        // 未使能DCACHE
/* #define CONFIG_USB_HS */                   // 未使能HS，用内部FS PHY
```

**DWC2 参数**（`usb_glue_hc.c`）：

```c
.phy_type = DWC2_PHY_TYPE_PARAM_FS,           // 内部FS PHY
.host_rx_fifo_size        = 64,  // RX FIFO 256字节（单位：32-bit words）
.host_nperio_tx_fifo_size = 64,  // 非周期TX FIFO 256字节
.host_perio_tx_fifo_size  = 64,  // 周期TX FIFO 256字节
```

**中断**：`INT_SRC_USBHS_GLB` → `INT031_IRQn` → 优先级 15（最低）
所有 USB 事件走一个中断入口，内部按 GINTSTS 分发。

---

## 几个要注意的点

### 1. OSAL 是裸机轮询，不是 RTOS

- 信号量、MQ 都是 `delay_ms(1)` 轮询等待
- `usb_read_sector()` 长时间读文件时**没有调 `timer_process()`**，可能导致 hub 超时断开。建议大文件读取循环里加一下

### 2. 旧 DDL 代码要排除

项目里有两套 USB host 栈，**不要同时编译**：

```
USB/host_core/*.c              ← 删/排除
USB/host_class/msc/*.c         ← 删/排除
```

### 3. DMA 强制 4 字节对齐

DWC2 DMA 模式要求 `transfer_buffer` 和 `setup` 地址 4 字节对齐，否则断言失败。栈上 `uint8_t buf[512]` 默认 8B 对齐，没问题。

### 4. DCACHE 没开，但代码里有调用

`CONFIG_USB_DCACHE_ENABLE` 未定义时，`usb_dcache_clean/invalidate` 要是空操作。确认你的 `usb_dcache.h` 里处理好了，否则链接失败。

### 5. MSC 设备查找

`usbh_find_class_instance("/dev/sda")` 只找到第一个 MSC 实例，多 LUN 的 U 盘只会初始化第一个。

---

## 启动时序

```c
usb_host_start():
  usbh_initialize(0, CM_USBHS_BASE, NULL)   // 创建bus、MQ
  usb_hc_init(&g_usbhost_bus[0])             // 手动初始化DWC2（裸机无hub线程）

usb_wait_device_ready():
  轮询 hub MQ → 处理事件 → 检查 /dev/sda 是否就绪
```

裸机下 `usbh_initialize()` 创建线程的调用是 stub（返回1），所以必须手动调用 `usb_hc_init()`。
