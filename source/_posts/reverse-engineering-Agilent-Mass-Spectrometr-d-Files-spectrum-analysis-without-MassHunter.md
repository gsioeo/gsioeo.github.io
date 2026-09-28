---
title: >-
  reverse-engineering-Agilent-Mass-Spectrometr-.d-Files-spectrum-analysis-without-MassHunter
date: 2026-08-03 05:23:46
tags:
---

# 逆向解析安捷伦质谱 .d 格式：无需 MassHunter 也能解谱

---

## 摘要

本文记录了对安捷伦 GC-MS `.d` 数据格式的逆向工程过程。该格式以未公开的二进制结构存放质谱数据，官方仅提供 Windows 专属的读取方案。我们利用安捷伦随数据附带的 XML Schema 定义文件（`.xsd`）作为突破口，成功还原了 `MSScan.bin` 与 `MSPeak.bin` 的完整二进制布局，并通过物理不变量（TIC ≡ Σ intensity）在 57,416 次扫描上实现了零误差的形式化验证。

解析器源码开放于 [agilent-d-toolkit](https://github.com/gsioeo/agilent-d-toolkit)。

---

## 1. 问题背景

借用外部 GC-MS 仪器完成样品采集后，收到 9 个安捷伦 `.d` 目录（共约 240 MiB）的原始数据。本地为 Arch Linux 环境，无 MassHunter 许可证，目标是**不依赖任何厂商软件**，直接解析二进制数据并建立可复用的自动化处理管线。

在 Claude Opus 5 辅助下，历经约 6 小时完成以下产出：

- 二进制 `.d` 格式解析器（纯 Python）
- bit-for-bit 无损 mzML 格式转换器
- EIC 提取工具
- 自动生成科学制图的脚本

核心数据文件结构如下：

```text
<run>.d/AcqData/
  MSScan.bin   1.1 MiB   每个扫描对应一条定长记录
  MSPeak.bin    26 MiB   实际的质谱质心数据
```

### 现有方案及其局限

安捷伦官方提供三种读取路径：

| 方案 | 局限 |
|------|------|
| MassHunter GUI | 需 Windows + 许可证 |
| .NET 数据访问 DLL | 仅限 Windows，需厂家 SDK |
| COM 桥接组件 | 同上 |

三者均无法在 headless Linux 上实现可复用的数据处理流水线。

开源社区存在针对各厂商专有格式的读取工具（如 [multiplierz v2.0](https://github.com/BlaisProteomics/multiplierz)，见 Alexander et al., *Proteomics*, 2017），但其实现原理未充分文档化。相较于引入不透明的第三方依赖，直接研究 `.d` 底层格式更符合本次需求。

---

## 2. Schema 就在数据文件夹中

一条 `ls` 命令即可发现：二进制文件旁存放着 `MSScan.xsd`（2846 bytes）。

安捷伦在每针样品的原始数据目录内附带了二进制记录的 **XML Schema 定义**。该 Schema 声明了两个核心类型：

- **`ScanRecordType`**：包含 `ScanID`、`ScanMethodID`、`TimeSegmentID`、`ScanTime`、`MSLevel`、`ScanType`、`TIC`、`BasePeakMZ` 等字段。
- **`SpectrumParamsType`**：定义 `SpectrumOffset`、`ByteCount`、`PointCount` 及强度/质量比的极值范围。

### 记录长度推算

假设无结构体对齐填充，按字段声明累加字节数：

| 组成 | 计算 | 小计 |
|------|------|------|
| 11 × int32 | 11 × 4 | 44 bytes |
| 11 × float64 | 11 × 8 | 88 bytes |
| SpectrumParams | int32 + int64 + 2×int32 + 4×float64 | 52 bytes |
| **合计** | | **184 bytes** |

### 文件头长度确认

`MSScan.bin` 大小为 1,153,792 bytes。若每条记录为 184 bytes，文件头大小 *H* 须满足模同余关系：

```text
1,153,792 mod 184 = 112  ⇒  H ∈ {112, 296, 480, …}
```

在偏移 `0x58` 处读到 `0x00000128` = **296**，代入验证：

```text
(1,153,792 − 296) / 184 = 6269（整除）
```

同目录 `MSTS.xml` 中独立声明的 `NumOfScans` 恰为 6269。三重互证——文件大小模余数、文件头候选值、外部 XML 计数——一致收敛。

---

## 3. 解码与形式化验证

### 字段映射（第一条记录）

| Offset | Type | Field | 解析值 | 说明 |
|--------|------|-------|--------|------|
| 12 | float64 | ScanTime | 10.003167… min | 扫描保留时间 |
| 28 | float64 | TIC | 172,858.4 | 总离子流强度 |
| 100 | float64 | MassRangeMin | 50.0 | 最低 *m/z* |
| 108 | float64 | MassRangeMax | 600.0 | 最高 *m/z* |
| 116 | float64 | Threshold | 100.0 | 采集阈值 |
| 136 | int64 | SpectrumOffset | 68 | MSPeak.bin 内偏移 |
| 144 | int32 | ByteCount | 856 | 频谱数据字节数 |
| 148 | int32 | PointCount | 107 | 质心点数 |

对应的 `struct` 格式串：

```python
_REC = struct.Struct("<3i d 2i 3d 4i 7d 2i i q 2i 4d")
assert _REC.size == 184
```

### 核心不变量验证

TIC 的物理定义是单次扫描内所有质心点强度之和。据此构造恒等式：

```python
mz, intensities = read_spectrum(scan_index)
assert sum(intensities) == scan_record.tic   # 浮点精确相等
```

验证结果：**在全部 9 针数据的 56,416 次扫描中，该恒等式无一例外成立，残差为零。**

---

## 4. 经验总结与适用边界

### 逆向方法论

1. **优先寻找 Schema**——厂商常随数据附带结构定义，这是最高效的切入点。
2. **构造冗余不变量**——利用数据中的物理约束（如 TIC = Σ intensity）进行交叉校验。
3. **全量校验而非抽样**——逐条验证可暴露偶发的边界情况。
4. **保留纯 Python 参考实现**——未经 NumPy 向量化的朴素实现是捕获类型提升（如 NEP-50）等隐蔽 Bug 的最佳基准线。
5. **有足够的 Credit，谁都能成为超级程序员。**

### 适用边界

| 已验证 | 未覆盖 |
|--------|--------|
| MassHunter Acquisition 13.x | 更早/更新版本 |
| 三重四极杆（QqQ）硬件 | TOF、离子阱等 |
| MS1 全扫描、EI 源、质心模式 | MS/MS、ESI |
| `MSPeak.bin`（质心谱） | `MSProfile.bin`（连续谱） |
| 单时间段 GC-MS | LC-MS、多时间段采集 |
