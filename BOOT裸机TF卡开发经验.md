# TF 卡驱动开发问题与解决方案总结

---

## 由哪些文件构成

```
TF/
├── cfg_tf.h         ← 引脚定义 + 命令/响应宏 + API 声明
├── cfg_tf.c         ← SDIO 驱动层（初始化、读扇区、卡检测）
├── fat_reader.h     ← FAT 文件系统接口声明
└── fat_reader.c     ← FAT 解析（MBR/VBR、目录遍历、文件读取）

driver/
└── hc32_ll_sdioc.c  ← HC32 官方 SDIO 外设 LL 库
```

**依赖关系：**

```
iap_tf_update()          ← app/iap_inf.c
  └─ fat_find_bin()      ← fat_reader.c
       └─ TF_ReadBlock() ← cfg_tf.c
            └─ SDIOC_...  ← hc32_ll_sdioc.c
```

**注意：** `fat_reader.c` 的扇区读取函数可以切换——默认用 `TF_ReadBlock`，调用 `fat_set_reader(usb_read_sector)` 就能切到读 U 盘。FAT 解析代码被 TF 和 USB 两个 IAP 模式共用。

---

## 1. SDIO 时钟配置错误（最致命）

```c
uint16_t clk_div = SDIOC_CLK_DIV256;  // 必须给默认值！
```

- 不要 `(void)` 忽略返回值
- `SDIOC_GetOptimumClockDiv()` 可能返回失败但不写 `clk_div`，栈上垃圾值会导致时钟跑 120MHz，卡不响应

```c
if (LL_OK != SDIOC_GetOptimumClockDiv(SDIOC_OUTPUT_CLK_FREQ_400K, &clk_div))
    clk_div = SDIOC_CLK_DIV256;
```

---

## 2. 每发一条命令都要等 CC（Command Complete）

不等 CC 就读 RESP 寄存器，拿到的全是上条命令的残留数据（RCA 错误、选卡失败）。

```c
/* 发命令后必须等 CC 置位，才能读 RESP */
SDIOC_SendCommand(CM_SDIOC1, &cc);
while (!(CM_SDIOC1->NORINTST & SDIOC_NORINTST_CC));  // 等 CC
CM_SDIOC1->NORINTST = SDIOC_NORINTST_CC;              // 清标志
resp = read_resp();                                    // 再读响应
```

---

## 3. 数据读：先读 FIFO，不等 TC

FIFO 满了不读，TC 永远不会置位 → **死锁**。

**错误流程：**

```
等 CC → 等 TC → 读 FIFO      ← TC 永远不来（FIFO 满了卡住）
```

**正确流程：**

```
等 CC → 有 BRE 就读 FIFO → TC 到了才结束
```

```c
while (tout--) {
    if (ns & SDIOC_NORINTST_TC) { ... return 0; }  // 传输完成

    while (CM_SDIOC1->PSTAT & SDIOC_PSTAT_BRE) {    // FIFO 可读
        uint32_t w = *(volatile uint32_t *)&CM_SDIOC1->BUF0;
        // 按 4 字节读
    }
}
```

---

## 4. ACMD41 参数要带电压范围

```c
CMD41(0x40FF8000)  // 不只是 0x40000000（HCS），还要电压位
```

- `bit 31` = HCS（支持高容量卡）
- `bit 23:0` = 电压范围（0xFF8000 表示 2.7V~3.6V）
- 不给电压范围，卡不返回 ready 位

---

## 5. FIFO 字节序用 32-bit 读

```c
uint32_t w = *(volatile uint32_t *)&SDIOCx->BUF0;
```

不要自己拼 `BUF0` + `BUF1`，直接按 RT-Thread / DDL 方式读 32-bit。

---

## 6. Superfloppy = 无 MBR

APP（RT-Thread）格式化的卡没有分区表，VBR 直接起始于扇区 0。0x55AA 签名在扇区 0 末尾。`fat_init` 要兼容这种格式。

```c
/* fat_init() 中的处理逻辑 */
if (part_lba == 0 || part_sec == 0) {
    /* 无 MBR 分区表：尝试 Superfloppy */
    if (*(uint16_t *)&sec_buf[0x1FE] != 0x55AA)
        return -3;  // 无 0x55AA 签名则放弃
    goto parse_bpb;  // 直接解析 VBR
}
```
