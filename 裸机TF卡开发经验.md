# TF 卡驱动开发问题与解决方案总结

## 1. SDIO 时钟配置错误（最致命）

```c
uint16_t clk_div = SDIOC_CLK_DIV256;  // 必须给默认值！
```
- 不要 `(void)` 忽略返回值
- 函数可能返回失败但不写 `clk_div`，栈上垃圾值导致时钟跑 120MHz，卡不响应
```c
if (LL_OK != SDIOC_GetOptimumClockDiv(..., &clk_div))
    clk_div = SDIOC_CLK_DIV256;
```

## 2. 每发一条命令都要等 CC（Command Complete）

不等 CC 就读 RESP 寄存器，拿到的全是上条命令的残留数据（RCA 错误、选卡失败）。

## 3. 数据读：先读 FIFO，不等 TC

FIFO 满了不读，TC 永远不会置位 → **死锁**。

**正确流程：** 等 CC → 有 BRE 就读 FIFO → TC 到了才结束。

## 4. ACMD41 参数要带电压范围

```c
CMD41(0x40FF8000)  // 不只是 0x40000000（HCS），还要电压位
```

不给电压范围，卡不返回 ready 位。

## 5. FIFO 字节序用 32-bit 读

```c
uint32_t w = *(volatile uint32_t *)&SDIOCx->BUF0;
```

不要自己拼 `BUF0` + `BUF1`，直接按 RT-Thread / DDL 方式读 32-bit。

## 6. Superfloppy = 无 MBR

APP（RT-Thread）格式化的卡没有分区表，VBR 直接起始于扇区 0。0x55AA 签名在扇区 0 末尾。`fat_init` 要兼容这种格式。
区表，VBR 直接起始于扇区 0。0x55AA 签名在扇区 0 末尾。fat_init 要兼容这种格式。
  ```
