# Dell Precision 3660 AVX-512 解锁分析（BIOS 1.2.38）

> 这是一份**只读分析 / 研究记录**，不是可刷写的 mod。
> 作者实测：改 `SiInitFsp` 后刷机不启动。BootGuard + MHTS + write-once MSR 构成闭环，请勿在生产机或唯一机器上尝试。

## TL;DR

- 1.2.38 的 `SiInitFsp` 在 P-core / 非 ADL-N 上写 **MSR 0xAF bit1**，禁用 AVX-512。
- 改 FSP-S 不可行：BootGuard 验证 IBB（FSP-M + 校验器），IBB 里的 `DellTrustChainingPei` 再用 `__MHTS__` 验证 FSP-S；改 FSP-S 会破坏 FV 哈希，直接不启动。
- 运行时写 MSR 0xAF 也不行：0xAF 表现为 **write-once**；且这台 T3660 每次重启掉电，FSP-S 每次重跑并重写。
- 3/4 的来源不是 NVRAM/Setup：FSP-M 通过 **PCIEXBAR → ECAM → CAPID0_E+0xFC** 读平台能力位；公开 CAPID0 字段里没有 CPU Segment。
- 但 3/4 分支 `jne 0x134d8` 是跳到函数 epilogue，跳过的是 **整段 CPU init tail（MSR 0xCF/0xF1 + 0x1317e + 0x1320d）**，不是“只跳过 AVX512”。3/4 是 CPU-class/feature-path 的适用性 gate，不是可用的 AVX512 开关。
- 只剩：换消费级 B660/Z690 老 BIOS、ES/换 CPU 赌 CAPID0/DID0、PCH + BIOS 大改，或接受 AVX2。

## 0. 对早期版本的修正

早期 README 的 6 步是 first-pass 记录，以下地方需要修正：

| 早期写法 | 修正 |
|---|---|
| `B9 AF 00 00 00 mov eax, 0xAF` | `MOV ECX, 0xAF`（ECX 是 MSR 号，EDX:EAX 才是写入值） |
| `83 C8 02 or ecx, 2` | `OR EAX, 2`（ModRM 0xC8 对应 EAX，不是 ECX） |
| 只改 `83 C8 02` | 还有 bit0 分支 `0x1317e` 和外层 PCD gate；只改一处不一定够 |
| “任何 BIOS 修改都会触发 BootGuard” | 实际是两阶段：BootGuard 验 IBB；IBB 验 FSP-S（`__MHTS__`）；不是简单的全镜像哈希 |
| 12500 的微码 | 1.2.38 的 FIT 只有 **90672 / 0x1A**，没有 90675；12500 跑 CPU 自带老微码。**不要升级 BIOS**，新版可能加入 90675 补丁 |
| 微码 ≤ 0x15 | 公开资料：0x16 及更早支持 AVX-512，0x18+ 禁用（0x17 未知）；以实际 CPUID/微码为准 |

## 1. 官方 BIOS 包

下载：

- `https://dl.dell.com/FOLDER08339759M/1/Precision_3660_1.2.38.exe`
- `https://dl.dell.com/FOLDER08339940M/1/BIOS_IMG.rcv`

实测两个文件**完全相同**：

```
SHA256 771dee952a723731fcb936c79528f12f3cc8fb64a5ab8e63b15de6d85a8173ee
```

Dell PFS 解包（BIOSUtilities 的 `Dell_PFS_Extract`）得到两个固件变体：

- `2 RotaAdl_16.0.15.1662_EVT_X02_64`
- `3 RotaAdl_16.0.15.1662_EVT_X02_CHPB_64`

每个变体含：

- 32MB `System BIOS with BiosGuard`
- 9MB Intel ME Corporate
- EC / Thunderbolt / Cypress PD 等

32MB 是 **BIOS region**（映射基址 `0xFE000000`），不是完整 SPI dump。完整 SPI 还需要描述符 + ME；实际 flash 容量与布局以机器为准。

## 2. FIT / BootGuard / IBB

FIT（`UEFIExtract` 解析）：

```
Microcode: CpuSignature 00090672h, Revision 0000001Ah, Date 03.01.2022
Startup ACM: 11.11.2021, ACM SVN 0003
BootGuard Key Manifest / Boot Policy
```

IBB segments（BootGuard 验证范围）：

```
FFE20000 + 000B0000
FFED0000 + 00010000
FFEE0000 + 0011A080
FFFFAB80 + 00005480
```

- 顶部约 1.9MB。
- **SiInitFsp 在 FFDB38A8，位于 IBB 之外**；微码在 FFC81000，也在 IBB 之外。
- 但 IBB 里的校验器会用 MHTS 校验 FSP-S，所以“在 IBB 外”不等于安全。

## 3. 第二阶段：`__MHTS__` 哈希清单

```
文件 99 85CB2173-621F-4745-A81E-98B9C5771AFE
IBB 副本:   FFF31FB8, 0x1018
主 BIOS 副本: FFB31FB8, 0x1018（与 IBB 副本逐字节相同）
数据: +0x18 起 "__MHTS__"
8 个条目，每个含 SHA1 / SHA256 / SHA384 / SHA512
```

FSP-S 外层 FV（`FFDB0000`，`0x70000`）的四组哈希位于：

```
FFF323B0  SHA1
FFF323C4  SHA256
FFF323E4  SHA384
FFF32414  SHA512
```

实测与 FSP-S FV 完全一致：

```
SHA1   2eadaa7a42266402bd2ea454dba56232c3f66730
SHA256 ea26b061b4b826072227e391b873b2ec6eb41553cbe265449abf04dd128f1b11
SHA384 f879de4a56f114c5e0a3d5f109fcf9505fbfef76a5234094f095d01266d74f3e78f6dfff0da70e6d88c1ad634f9d6030
SHA512 0198618ea8a3bab77275e678a6bc4c1fbf4296c4446e5028564129149b78c6ac155ef52fb02b1d3da343f518cb1c0b9b3b0c03e3539a37320e71f8642abcaf16
```

相关模块：

- `DellTrustChainingPei`（IBB，GUID `84E90BA3-CB79-4267-AE2F-437B86DAA6F4`）：按 MHTS GUID 定位 `99` 文件并解析/比对。
- `BiosInfo`（`4A4CA1C6-871C-45BB-8801-6910A7AA5807`）/ `BiosInfoChecker`（`9581983A-3663-46F0-83EC-8A0506EB63E7`）：解析 FFS/FV、MHTS。
- `PostIbbVerificationCollectorPei`（`CB992535-8713-4ADE-94F2-5D32CCAB8C5E`）：收集 post-IBB 校验结果。
- `PcdPeim` FFS GUID `9B3ADA4F-AE56-4C24-8DEA-F03B7558AE50`（raw 默认 PCD 库 `0x1E28`）。
- `PcdDxe` FFS GUID `80CF7257-87AB-47F9-A3FE-D50B76D89541`（raw `0x7D8`）。
- `PcdNvStore` 变量 GUID `EA296D92-0B69-423C-8C28-33B4E0A91268`。

## 4. SiInitFsp 的 AVX 策略

`SiInitFsp` FFS 基址 `FFDB38A8`；PE32 body 内：

### 4.1 bit0 分支：`0x1317e`

- 由 `ds:0x35d58 + 0x28 bit6`（FSP-S UPD 配置）决定 set/clear **MSR 0xAF bit0**。
- bit0 = 1：AVX2 + AVX512 全禁；bit0 = 0：清 bit0。

### 4.2 bit1 分支：`0x1320d`

```asm
13224: call 0x3955          ; CPUID.1.ECX bit26 = XSAVE
13229: cmp  al,1
1322b: jne  0x1329c         ; 无 XSAVE -> 跳过
1322d: rdmsr 0xaf
1323c: test al,3
1323e: jne  0x1329c         ; bit0/bit1 已置位 -> 跳过（但已经禁用）
13240: cmp  byte [ebp-1],0x40
13244: jne  0x13289         ; 不是 0x40 -> 跳过 OR，写回原值
13286: or   eax,2           ; 置 bit1
13295: wrmsr
```

`[ebp-1]` 由 `0x3b08` 计算：

```
CPUID(7,0).EDX bit15 = HYBRID
  hybrid:      CPUID(0x1A,0).EAX >> 24  -> 0x40 P-core / 0x20 E-core
  non-hybrid:  CPUID(1,0).EAX & 0xFFF0FF0 == 0xB06E0 ? 0x20 : 0x40
```

12500（非混合、非 ADL-N）→ `0x40` → 置 bit1。

### 4.3 外层 PCD gate：`0x1345e`

```asm
1345e: test bl,bl
13462: call 0x562d          ; PCD 读取 == 4 ?
13467: mov  ds:0x35e8f,al
1346c: call 0x5643          ; PCD 读取 == 3 ?
13471: mov  ds:0x35e8c,al
1347d: cmp  byte ds:0x35e8f,0
13484: jne  0x134d8         ; PCD==4 -> 跳到函数 epilogue
13486: test al,al
13488: jne  0x134d8         ; PCD==3 -> 跳到函数 epilogue
134ce: call 0x1317e
134d3: call 0x1320d
134d8: pop esi; pop ebx; mov esp,ebp; pop ebp; ret   ; epilogue
```

注意 `jne 0x134d8` 是跳到函数 epilogue，不是跳到某个“AVX 之后”的标签。它跳过的是整段 tail：
MSR 0xCF/0xF1 的 workaround、config bit、`0x1317e`（bit0）、`0x1320d`（bit1）。
所以 3/4 不是“只跳过 AVX512”，而是“这个 CPU 类型不走这段 CPU init tail”。

`0x562d` / `0x5643` → `0x10b24` → `0x2f007`：

```asm
2f007: LocatePpi(PCD PPI 01F34D25-4DE2-23AD-3FF3-36353FF323F1)
       GetPtr(GUID @ FSP 0x31234, Token 0x200002)
10b24: 读 [PCD + 0xFC]
```

### 4.4 3/4 的真实来源：PCIEXBAR → ECAM → CAPID0_E

`FspPcdInit`（FSP-M，GUID `88C17E54-EBFE-4531-A992-581029F58126`）里：

```asm
; 读 PCI 0:0.0 offset 0x60 = PCIEXBAR
mov  edx,0xCF8
mov  eax,0x80000060
out  dx,eax
mov  edx,0xCFC
in   eax,dx
mov  ebx,eax
...
push (ebx & 0xFC000000)     ; ECAM base
push 0x200002
push 0xFFE295E4             ; GUID 977C97C1-47E1-4B6B-9669-436699CBE45B
call PcdSet32
```

因此：

```
PCIEXBAR (0x60) -> PCD 0x200002 = ECAM base
FSP-S -> [ECAM + 0xFC] -> 和 3/4 比较
```

`ECAM + 0xFC` = **D0:F0 offset 0xFC** = **CAPID0_E 的高字节（bits 24–31）**。

但 3/4 不是 AVX512 policy：它命中的是 `0x13484/0x13488` 的整段 tail 跳过。把它当成“AVX512 enable”是误读；即使能改到 3/4，也会跳过 MSR 0xCF/0xF1 的 workaround，CPU init 可能不完整。

### 4.5 3/4 跳过的代码量（统计）

| 范围 | 字节 | 指令 | 3/4 路径 |
|---|---:|---:|---|
| 函数 `0x132a1–0x134dd` | 573 | 171 | 部分执行 |
| ├ pre-gate `0x132a1–0x1345d` | 445 | 133 | 执行 |
| ├ gate + PCD `0x1345e–0x13488` | 43 | 12 | 执行 |
| ├ skipped-in-fn `0x1348a–0x134d7` | 78 | 21 | 跳过 |
| └ epilogue `0x134d8–0x134dd` | 6 | 5 | 执行 |
| bit0 `0x1317e` | 143 | 54 | 跳过 |
| bit1 `0x1320d` | 148 | 55 | 跳过 |
| XSAVE `0x3955` | 35 | 17 | 跳过 |
| CPU type `0x3b08` | 67 | 29 | 跳过 |
| HYBRID `0x3ae4` | 36 | 19 | 跳过 |
| XGETBV `0x3b0` | 16 | 11 | 跳过 |
| XSETBV `0x3c0` | 16 | 5 | 跳过 |

- 本路径独占跳过：约 **539 bytes / 211 条指令**。
- 函数 + 这些 helper：约 **1112 bytes / 382 条指令**；3/4 跳过约 **48% 字节 / 55% 指令**。
- 整个 FSP-S PE32 body：**225,248 bytes**；跳过部分约 **0.24%**。
- 3/4 路径实际执行：pre-gate 的 CPU init（`0x12f6e`、`0x388c`、`0x7757`、`0x3904`、MSR 0xFE/`0x13108`、`0x12f09`、config bit `+0x28 bit5`、MSR 0x33）+ PCD 检查 + epilogue + **所有其他 FSP-S 函数**。
- 3/4 **不会**启用另一条 AVX512 路径；它只是跳过这段 tail。被跳过的 AVX/MSR 0xCF/0xF1 代码在本函数内没有替代。

## 5. CAPID0 调研

### 5.1 寄存器布局（12th Gen Datasheet Vol.2）

| Offset | 寄存器 |
|---|---|
| **0x60** | **PCIEXBAR**（PCIe ECAM base，不是 CAPID0） |
| 0xE4 | CAPID0_A |
| 0xE8 | CAPID0_B |
| 0xEC | CAPID0_C |
| 0xF0 | CAPID0_E |

### 5.2 公开字段里没有 “CPU Segment”

- **CAPID0_A**：NVMe/PCIe disable、DMI width、ECC、VT-d、DDR size、IGD、DID0OE、2LM、DDR overclock 等。
- **CAPID0_B**：IPU/Trace Hub/OC/SMT/Cache/SVM/DDR/PCIe Gen3 等。
- **CAPID0_C**：PCIe Gen5/Gen4、DDR4/LPDDR4 等。
- **CAPID0_E**：CRASHLOG_DIS、VDDQ、IBECC、DDR5/LPDDR5 等。

**没有 “CPU Segment” 字段。** 3/4 只能落在 CAPID0_E bits 24–31 的 **Reserved / 私有位**，或这个 PCD 并非我理解的 ECAM 路径。公开文档无法给出 3/4 的映射。

### 5.3 访问属性与“写死”

- CAPID0 字段基本都是 **RW/L**（read-write/lock），不是硬熔丝。
- 默认值来自 **CPU / 板级 strap / ME / BIOS**；BIOS/ME 可在 lock 前写，lock 后只读。
- FSP-M 在 BIOS/OS 之前读；等 OS 能跑，已经 lock。
- CAPID0_A bit10 **DID0OE** 的文档描述是 **“For debug and samples only”**：允许 override DID0，供 Intel / 样片调试。这是公开资料里最接近“调试后门”的位；除此之外没有发现通用 Intel 后门。
- 公开研究（coreboot / EDK2 / BIOS modding）主要用 CAPID0 的 DMI width、PCIe disable、SMT、DDR、ECC 等平台能力位；**没看到对 3/4 私有位的逆向**。

### 5.4 对“换 CPU / ES / 换 PCH”的含义

- CAPID0 在 **CPU 的 Host Bridge**；换 PCH 不会改 CPU 的 CAPID0。PCH 换 + FSP 改是绕 BootGuard 的路径，不是改 CAPID0。
- ES/QS 样片的 DID0/CAPID0 可能不同（DID0OE 就是给 debug/samples 的）；3/4 有可能对应样片/调试平台 ID。这是“有依据的赌博”，不保证。
- 消费级 B660/Z690 换的是 **另一套 FSP/BIOS**，策略可能不同；12500 无 E-core，老 BIOS/老微码下开 AVX512 是已知可行的。
- 3/4 不是“AVX512 开关”；它是整段 CPU init tail 的跳过条件。即使能改，也会跳过 MSR 0xCF/0xF1 的 workaround，可能破坏 CPU init，不是可用的解锁路径。

### 5.5 进一步推测：3/4 的类别与 MSR 0xCF/0xF1

> 以下内容为代码推断。公开 Intel 资料没有定义 3/4 字段和 MSR 0xCF/0xF1 的名称/语义，结论请按“推测”看待。

#### 5.5.1 3/4 是字段值，不是“bit3/bit4”

- `0x10b24` 读 `[ECAM + 0xFC]` = CAPID0_E 高字节（bits 31:24）。
- FSP-S 用 `cmp al,3` / `cmp al,4` 比较整个 byte，所以 3/4 是**字段值**，不是单个 bit。
- 公开 CAPID0_E：bit24 = CRASHLOG_DIS，bits31:25 = Reserved。
- 3 = `0b00000011`（bit24+25），4 = `0b00000100`（bit26）；只能落在 Reserved 位里，具体位宽未公开。

#### 5.5.2 调用点线索

| 调用点 | 3/4 命中时 | 线索 |
|---|---|---|
| `0x2b76` / `0x2b7f` | 跳过 MSR 0x650/0x651 读取 | 0x650/0x651 像 memory/DDR 相关 |
| `0x2d19` / `0x2d26` | 跳过一段 memory 初始化 | 同上 |
| `0x4adc` / `0x4b2e` | type4 走另一条 PCH/PCIe 路径（`0x434`） | PCH 配置不同 |
| `0x3bdf` | 影响 ratio/frequency 计算 | DDR/fabric ratio |
| `0x128c` | type4 跳过 `[0xFC]` 读取/策略 | policy 不同 |
| `0x13462` | 跳过整段 AVX tail | AVX 策略不同 |

`0x12a0` 的 type4 分支还会检查 `CPUID == 0x906A0`（ADL-P），说明 3/4 至少和移动端/ADL-P 相关。

**推测**：3/4 是 **平台/CPU class**（移动端 ADL-P / 小核 ADL-N / 其他 SoC 段位），不是 AVX 开关。3 和 4 具体哪个对应哪个类，公开资料无法确认。12500 是桌面大核 ADL-S，不属于 3/4，所以走 AVX tail 并被禁。

#### 5.5.3 MSR 0xCF / 0xF1

```asm
1348a: mov  ecx,0xCF
1348f: rdmsr
13495: test al,0x2          ; 0xCF bit1
13497: je   0x134c0
13499: mov  ecx,0xF1
1349e: rdmsr
134a4: and  eax,0xFFFFFF00  ; 清 0xF1 低 8 位
134b9: mov  ecx,0xF1
134be: wrmsr
```

`0xCF` 还有第二处使用：

```asm
571: mov  ecx,0xCF
576: rdmsr
57c: test al,0x4          ; 0xCF bit2
57e: je   0x589
580: ... call 0x164a3     ; 另一个 workaround
```

- `0xCF` 更像 **capability/erratum bitfield**（bit1、bit2 触发不同 workaround）。
- `0xF1` 低字节更像 **feature bitmap**；清零 = 禁用一组 feature。
- 这是**条件性 feature-disable workaround**；它在 AVX tail 里，且 3/4 类会跳过，所以大概率与 **AVX/大核 feature** 有关；具体是 AVX512、AMX 还是别的，公开资料无法确认。
- **MSR 0xCF / 0xF1 不在公开 Intel SDM 里**，是 FSP/硅片私有；不要从 OS 乱写。

#### 5.5.4 结论

- 3/4 是 **平台 class**，不是 AVX512 enable；3/4 类跳过 AVX tail 是因为它们的 memory/PCH/AVX 策略不同。
- 0xCF/0xF1 是**未公开的 capability/feature workaround**，不是通用开关。
- 这两点都不改变最终结论：T3660 上不存在可用的 3/4 配置后门；软件路径仍然闭环。

## 6. 为什么改 FSP 会砖

1. 改 `SiInitFsp` → FSP-S 外层 FV 哈希变。
2. MHTS 里 FSP-S 的四组哈希对不上。
3. `DellTrustChainingPei` / `BiosInfoChecker` 校验失败 → 不启动。
4. 把 MHTS 改掉 → MHTS 在 IBB 里 → IBB 哈希变 → BootGuard 失败。
5. 主 BIOS 里还有一份相同 MHTS；如果校验器用的是 IBB 副本，改主副本也没用；`PostIbbVerificationCollectorPei` 还会做 post-IBB 校验。

作者实测：改 FSP 后刷机不启动；即使能通，也要 PCH + 大规模 BIOS 修改，代价极高。

## 7. 运行时 MSR 0xAF

- FSP-S 在 PEI 阶段第一次实际写入（12500 上是 bit1）后，MSR 0xAF 若为 **write-once** 即锁死。
- bit0/bit1 在同一个 MSR，锁语义一致；“隔壁 AVX 策略”不可能不同。
- 这台 T3660 每次重启掉电（global reset / 安全设计），FSP-S 每次重跑并重写；S3/热复位也绕不过。
- 因此 `wrmsr 0xaf 0` 的运行时路径在当前证据下是死的。除非后续实测证明 MSR 可写。

## 8. 哪些情况 FSP-S 不禁 AVX512

| # | 条件 | 12500 |
|---|---|---|
| 1 | PCD 3/4（CAPID0_E 私有位 / ECAM 字节）—— 整段 tail 跳过，不是 AVX512 专用 | 否 |
| 2 | CPUID 类型 0x20：E-core 或 ADL-N（0xB06E0） | 否 |
| 3 | XSAVE 不支持（CPUID.1.ECX bit26 = 0） | 否 |
| 4 | MSR 0xAF bit0/bit1 已置位 | 否（当前是 0） |
| 5 | 外层 CPU init 函数未被调用（phase/policy） | 不适用 |
| 6 | 微码 ≤ 0x16、未熔断（CPU 侧） | **是**，但 FSP 仍会禁 |

注：真正的 AVX512-specific skip 只有 #2（CPUID 0x20）和 #3/#4；#1 是 CPU-class/feature-path gate，不是可用的“AVX512 enable”。

## 9. 结论与可选路径

- **T3660 + 1.2.38 的纯软件解锁是闭环的**：CAPID0/ECAM 堵配置（而且 3/4 不是 AVX512 开关），BootGuard + MHTS 堵代码，write-once MSR 堵运行时。
- 可选：
  1. **消费级 B660/Z690 + 老 BIOS/老微码**：最可靠；12500 无 E-core，AVX512 可开。
  2. **ES CPU / ES 板**：赌 CAPID0/DID0 样片值；不保证。
  3. **PCH 换 + FSP 改**：能通但贵、风险高。
  4. **接受 AVX2**：如果负载收益不大，这是理性选择。
- **不要升级 BIOS**：1.2.38 没有 90675 微码补丁；新版可能加入，覆盖 CPU 自带老微码。

## 10. 复现方法 / 工具

- Dell PFS：`BIOSUtilities` 的 `Dell_PFS_Extract`。
- UEFI 解析：`UEFIExtract`（NE）、`UEFITool`。
- 反汇编：`objdump -d -Mintel`（PE32/TE，i386）。
- BootGuard / SPI：`chipsec`（只读检查；不要在目标机上随意写）。
- 关键偏移：
  - BIOS region base `0xFE000000`（32MB）。
  - FSP-S FV `0xFFDB0000`，size `0x70000`。
  - SiInitFsp FFS `0xFFDB38A8`；bit1 站点 PE32 offset `0x13286`。
  - MHTS `0xFFF31FB8` / `0xFFB31FB8`；FSP-S 哈希 `0xFFF323B0`。
  - CAPID0_E `0xF0`；PCIEXBAR `0x60`。
- 验证：对 FSP-S 外层 FV 计算 SHA1/256/384/512，与 MHTS 比较。

## 11. 免责声明

- 本文是分析记录，不提供改好的 BIOS，不保证任何刷写结果。
- 刷改过的 FSP 会触发 BootGuard / MHTS 校验失败；作者实测不启动。
- 不要在生产机 / 唯一机器上尝试；保留完整 SPI dump 和恢复手段。
