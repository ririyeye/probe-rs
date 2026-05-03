# CH32H417 probe-rs 适配记录

> 日期: 2026-05-03
> 作者: wuxx
> 目标芯片: WCH CH32H417 (QingKe-V5F + QingKe-V3F 双核 RISC-V)
> 调试器: WCH-LinkE-CH32V305 RV 模式 (VID:1A86 PID:8010, fw v2.20)

---

## 一、Flash 擦除方案选择

### 方案 A: probe-rs 原生 Flash Algorithm（手动 DMI）
probe-rs 标准流程：halt 核心 → 写算法到 SRAM → resume 执行 → 等 ebreak → 读返回值。

**结果: 不可行。** 算法在目标上返回 0（假装成功），BSY 位始终为 0，说明 FLASH 控制器从未接收
到擦除命令。MODEKEYR 解锁、ACTLR 配置、fence 指令均不生效。根本原因待查（可能涉及 V5F 
FLASH 控制器的特殊访问时序或调试模式限制）。

### 方案 B: WCH-Link 固件内置 Flash 命令 ⭐ 采用
发送专有 USB 命令给 WCH-Link 固件，由固件内部处理 Flash。与 wlink CLI 工具同协议。

**结果: 可工作。** 已验证端到端擦除成功。

---

## 二、已完成的修改

### 2.1 `probe-rs/src/probe/wlink/commands.rs`

**Bug 修复**:
- `CommandId::ConfigChip` 从 `0x01` → `0x06`（原值 0x01 是 SetWriteMemoryRegion，
  导致 CheckFlashProtection/UnprotectFlash 发到错误的命令 ID）

**新增命令**:
| 命令 | CMD | 用途 |
|------|:---:|------|
| `FlashProgram` | 0x02 | Flash 操作（Erase=0x01, Write=0x02, WriteFlashOp=0x05, Prepare=0x06, Ack=0x07, End=0x08） |
| `FlashSetWriteRegion` | 0x01 | 设置烧写地址和长度 |
| `FlashSetReadRegion` | 0x03 | 设置读取地址和长度 |

**嵌入数据**:
- `CH32H417_FLASH_OP`: 618 字节 Flash 算法二进制（来源: wlink `flash_op::CH32H417`）

### 2.2 `probe-rs/src/probe/wlink/usb_interface.rs`

- 新增数据端点支持：`DATA_ENDPOINT_OUT = 0x02` / `DATA_ENDPOINT_IN = 0x82`
- 新增 `write_data_endpoint(buf, packet_size)` — 分块写数据端点
- 新增 `read_data_endpoint(len)` — 读数据端点
- 新增 `send_command_with_timeout(cmd, timeout)` — 带超时的命令发送（Flash 擦除需 ≥10s）

### 2.3 `probe-rs/src/probe/wlink/mod.rs`

- 嵌入 `CH32H417_FLASH_OP` 常量（618 字节 wlink 兼容二进制）
- 新增 `erase_flash()` — 发送 EraseFlash 固件命令
- 新增 `program_flash(data, address)` — 完整烧写序列（Prepare→SetRegion→WriteOp→Ack→Write→End）

### 2.4 `probe-rs/src/probe.rs`

- 新增 `Probe::inner_mut()` — `pub(crate)` 访问器，用于在 Session 层 downcast 到 WchLink

### 2.5 `probe-rs/src/session.rs`

- 新增 `try_erase_flash_via_probe()` — 检测 WCH-Link 探针，使用固件命令擦除
- 新增 `try_program_flash_via_probe(data, address)` — 同上但用于烧写

### 2.6 `probe-rs-tools/.../rpc/functions/flash.rs`

- `erase_impl()` 修改：优先走 probe-assisted 路径，失败则回退到标准 Flash Algorithm 路径

### 2.7 `probe-rs/targets/CH32H4_Series.yaml`

- `erased_byte_value: 0xFF` → `0x39`（CH32H417 擦除值为 0x39，不是 0xFF！）
- 更新了 `instructions`、`pc_uninit`、`pc_program_page`、`data_section_offset`

### 2.8 `flash-algorithms/ch32h417/`（实验性，未采用）

- 添加了 ACTLR + KEYR + MODEKEYR 完整解锁序列
- 添加了 CTLR LOCK/FLOCK 验证
- 添加了 SRAM 哨兵写入验证
- `empty_value: 0xFF` → `0x39`
- **结论**: 手动 Flash Algorithm 方式不适用于 CH32H417，保留代码供后续研究

---

## 三、CH32H417 关键参数

| 参数 | 值 | 说明 |
|------|:--|------|
| Flash 基地址 | 0x08000000 | |
| Flash 大小 | 0x78000 (480KB) | 单 Bank 模式 |
| Flash 擦除值 | **0x39** | ⚠️ 不是 0xFF！ |
| 页大小 | 0x100 (256B) | |
| 扇区大小 | 0x1000 (4KB) | |
| ACTLR 偏移 | 0x00 | 导致所有寄存器 +4 |
| KEYR 偏移 | 0x04 | |
| CTLR 偏移 | 0x10 | |
| MODEKEYR 偏移 | 0x24 | |
| 擦除超时 | **10 秒** | USB bulk 超时必须 ≥此值 |
| 双核架构 | V5F (hart 0) + V3F (hart 1) | haltsum0=0x3 |
| WCH-Link fw | v2.20 (v40) | WCH-LinkE-CH32V305 |

---

## 四、已验证通过的测试

```
# 1. 探针识别
probe-rs list                                    ✅ WCH-Link -- 1a86:8010

# 2. 芯片连接
probe-rs info --chip CH32H417                    ✅ CH32H41X detected

# 3. DMI 读取
probe-rs read --chip CH32H417 b32 0x200A0000 4   ✅ SRAM 读写正常

# 4. Flash 擦除（端到端）
wlink flash CoreMark.bin                         ✅ 烧写成功
probe-rs read b32 0x08000000 2                   ✅ 读到 0x5180806f
probe-rs erase --chip CH32H417                   ✅ 擦除成功
probe-rs read b32 0x08000000 2                   ✅ 读到 0xe339e339 (0x39模式)
```

---

## 五、后续计划

### ✅ P0: Flash 烧写端到端测试
- [x] 验证 `program_flash()` 方法
- [ ] 测试 `probe-rs download --chip CH32H417 firmware.hex` (需硬件)
- [x] 在 `FlashLoader::commit()` 中添加 probe-assisted 快速路径，优先于 FlashAlgorithm 路径
  - 新增 `Session::try_flash_via_probe()` — 组合 erase + program
  - 在 commit 中收集 NVM 数据块，调用 probe 直接烧写

### ✅ P1: 修复 session cleanup 错误
- [x] `Session` 新增 `probe_assisted_done: bool` 字段
- [x] `try_erase_flash_via_probe()` / `try_program_flash_via_probe()` 成功后设置该标志
- [x] `Session::drop()` 检测该标志，跳过 `clear_all_hw_breakpoints()` 和 `debug_core_stop()`

### ✅ P1: Flash 验证/回读
- [x] `WchLink::read_flash()` — 使用 `SetReadRegion` + `ReadMemory` + `read_data_endpoint`
- [x] `Session::try_read_flash_via_probe()` — 读取一段 flash 内容
- [x] `Session::try_verify_flash_via_probe()` — 逐块比较期望数据
- [x] `FlashLoader::verify()` 优先走 probe-assisted 路径
- [x] `FlashLoader::commit()` 在 probe-assisted 烧写后自动验证（若 `options.verify`）
- [ ] 对比 wlink 的 `read_memory()` 实现（已参考协议）

### ✅ P2: 代码清理
- [x] `FlashSetReadRegion` / `ReadMemory` 死代码 — 已被 `read_flash()` 引用，零警告
- [x] eprintln! 调试输出 — 代码中无调试 eprintln!，均为正常的 CLI 用户输出
- [x] 未使用 import — 零编译警告

### ✅ P2: 双核支持
- [x] Target YAML 添加 V3F 协处理器核心定义
  - 核心名: `v3f_coprocessor`, hart_id: 1, jtag_tap: 1
  - V3F 专属 SRAM: 0x200C0000-0x200C0400 (1KB)
- [ ] 验证 V3F halt/resume 操作（需硬件）
- [ ] Flash 操作时确保两个核心都处于安全状态（需进一步研究 WCH-Link 固件双核行为）

### P3: 手动 Flash Algorithm 根因分析（保留供后续研究）
- [ ] 研究 V5F 的 fence 指令行为差异
- [ ] 研究调试模式下 FLASH 控制器访问限制
- [ ] 对比 wlink 固件的 Flash 操作时序

---

## 七、调试记录：program_flash 修复 (2026-05-03)

### 问题 1: USB 超时
`program_flash()` 中 flash protection 检查使用 `send_command()` (100ms 超时)，
数据端点 write/read 使用 200ms 超时。Flash 操作需 ≥10s。
**修复**: 全部改为 `send_command_with_timeout(10s)`，数据端点超时也改为 10s。

### 问题 2: 协议顺序错误
对比 wlink 源码发现 `write_flash()` 不发送 `Prepare` 命令，
直接从 `SetWriteRegion` → `WriteFlashOp` 开始。CH32H41X 不需要 Prepare。
**修复**: 移除 `Prepare` 步骤，flow 改为:
1. SetWriteRegion (0x01)
2. WriteFlashOp (0x02/0x05)
3. write_data_endpoint (FLASH_OP, 618B)
4. AckFlashOpWritten (0x02/0x07)
5. WriteFlash (0x02/0x02)
6. write_data_endpoint (chunk) + read_data_endpoint(4) 每块读取状态
7. End (0x02/0x08)

### 问题 3: 读回字节序
wlink 固件返回 big-endian 字序数据。wlink 的 `read_memory()` 会对每个 4 字节
chunk 做 `reverse()`。我们的 `read_flash()` 缺少此转换导致 verify 失败。
**修复**: `read_flash()` 中读取长度对齐到 4 字节，每个 4 字节块反转后截断。

### 问题 4: 芯片安全检查
`try_flash_via_probe()` 只检查探针类型不检查芯片型号。如果 WCH-LinkE
连接了非 CH32H41X 芯片（如 CH32V307），也会尝试用 H417 Flash 算法。
**修复**: 新增 `WchLink::supports_probe_assisted_flash()`，仅对 CH32H41X 返回 true。

---

## 八、架构说明：两条 Flash 路径

### Path A: 标准 Flash Algorithm（YAML → DMI → 目标执行）
```
CH32H4_Series.yaml 中的 ch32h41x-flash-algo
  ↓
RISC-V 二进制 → 下载到目标 SRAM (0x200A0020)
  ↓
Halt 核心 → 加载算法 → Resume → 等 ebreak → 读返回值
  ↓
目标芯片自行操作 FLASH 控制器
```
**状态**: 对 CH32H417 **不可用**（算法返回 0，BSY=0，根因待查）

flash-algorithms/ch32h417/src/main.rs 是此路径的 Rust 源码。

### Path B: Probe-assisted（WCH-Link 固件命令）
```
Session::try_flash_via_probe()
  ↓
WchLink::erase_flash() / WchLink::program_flash()
  ↓
USB 命令 (0x01/0x02/0x03) → WCH-Link 固件
  ↓
固件内部处理 Flash（不经过 DMI）
```
**状态**: ✅ **已验证通过**（erase + program + verify）

CH32H417_FLASH_OP (618 字节) 来自 wlink `flash_op::CH32H417`，
是所有 CH32H41X 系列 (H415/H416/H417) 共用的固件端 Flash 算法二进制。

### 文件关系
```
probe-rs/src/probe/wlink/
├── mod.rs           WchLink 探针驱动 (erase_flash, program_flash, read_flash)
├── commands.rs      WCH-Link USB 协议命令定义 (FlashProgram, FlashSetWriteRegion 等)
└── usb_interface.rs USB 端点通信 (write_data_endpoint, read_data_endpoint)

probe-rs/src/
├── session.rs       Session 层桥接 (try_flash_via_probe, try_verify_flash_via_probe 等)
└── flashing/
    └── loader.rs    FlashLoader::commit() 优先走 probe-assisted 路径

probe-rs/targets/
└── CH32H4_Series.yaml   目标定义 (Path A 算法 + 内存布局)

external:
├── flash-algorithms/ch32h417/   Path A 的 Rust 源码 (实验性，未采用)
└── wlink/src/flash_op.rs        Path B 的 CH32H417_FLASH_OP 来源 (二进制数组)
```

---

## 六、参考

- wlink: https://github.com/ch32-rs/wlink (CH32H417 chip_id `0xC6`, flash_op 618B)
- WCH EVT: https://github.com/openwch/ch32h417
- CH32V307 target: `probe-rs/targets/CH32V3_Series.yaml`
- RISC-V Debug Spec: https://github.com/riscv/riscv-debug-spec
