# CH32H417 probe-rs 适配记录

> 日期: 2026-05-03
> 作者: wuxx
> 目标芯片: WCH CH32H417 (QingKe-V5F RISC-V)

## 修改概要

### 1. wlink/probe 驱动 (`probe-rs/src/probe/wlink/mod.rs`)

添加 CH32H41x 芯片家族支持：

```rust
// RiscvChip 枚举新增
CH32H41X = 0xC6,  // CH32H415/CH32H416/CH32H417

// try_from_u8 新增映射
0xC6 => Some(RiscvChip::CH32H41X),

// support_flash_protect 新增
| RiscvChip::CH32H41X
```

chip_id `0xC6` 来源: wlink 库 (`ch32-rs/wlink/src/lib.rs`)

### 2. Target 描述 (`probe-rs/targets/CH32H4_Series.yaml`)

包含:
- CH32H417/CH32H415/CH32H416 三个变体
- V5F 核心内存映射 (Flash 480K + ITCM 128K + DTCM 255K)
- Flash algorithm (952 bytes PrgCode)

Flash algorithm 函数偏移:

| 函数 | 偏移 | 说明 |
|------|------|------|
| pc_erase_sector | 0x0 | EraseSector |
| pc_init | 0x90 | Init (解锁 FLASH + MODEKEYR) |
| pc_program_page | 0x106 | ProgramPage (256B 页编程) |
| pc_uninit | 0x220 | UnInit (锁定 FLASH) |

### 3. Flash Algorithm (`flash-algorithms/ch32h417/`)

独立 crate，使用裸指针直接操作 FLASH 寄存器。

**关键发现**: CH32H417 FLASH 控制器有 ACTLR 寄存器(offset 0x00)，
导致所有寄存器偏移比 CH32V30x 多 4 字节。不能直接复用 ch32v3 PAC。

| Register | CH32V30x | CH32H41x |
|----------|:--------:|:--------:|
| KEYR     | 0x00     | 0x04     |
| STATR    | 0x08     | 0x0C     |
| CTLR     | 0x0C     | 0x10     |
| ADDR     | 0x10     | 0x14     |
| MODEKEYR | 0x20     | 0x24     |

FLASH 控制器基地址相同: `0x40022000`

## 调试流程

```bash
# 编译 probe-rs
cd probe-rs
cargo build --release

# 烧录
probe-rs run --chip CH32H417 ./firmware.elf

# 调试
probe-rs attach --chip CH32H417

# 擦除
probe-rs erase --chip CH32H417
```

## 注意事项

1. WCH-Link 需要切换到 RV 模式 (VID:PID `1a86:8010`)
2. V3F 小核暂不支持 (target 描述仅包含 V5F)
3. Flash algorithm 尚未在真实硬件验证
4. 960K 双 Bank 模式暂未实现 sector 描述

## 参考

- wlink: https://github.com/ch32-rs/wlink (CH32H417 chip_id 和 flash_op 来源)
- WCH EVT: https://github.com/openwch/ch32h417
- CH32V307 target: `probe-rs/targets/CH32V3_Series.yaml`
