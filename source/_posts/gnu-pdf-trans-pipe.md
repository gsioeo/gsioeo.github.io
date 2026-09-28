---
title: gnu-pdf-trans-pipe
date: 2026-08-15 06:29:48
tags:
---

# 硬核终端 PDF 翻译管线：用 Poppler + Translate-Shell 构建符合 Unix 哲学的翻译方案

> 在 Arch Linux 终端中，不依赖 GUI、不使用多模态大模型、不调用 Ollama，仅凭 GNU/Unix 命令行工具完成整本 PDF 的中文翻译。这是一篇完整的试错与改进记录。

## 起因：翻译一本 GNU C 编程教程

手头有一本 `ctut.pdf`——GNU C Programming Tutorial，共 290 页。目标很简单：把它翻译成中文纯文本，方便在终端中阅读。

要求：
- 纯命令行完成，不要 GUI 应用
- 不用 Python 应用、不用 Ollama、不用多模态大模型
- 保持 GNU 手册的朴素风格，不要"AI 教材腔"
- 在 Arch Linux 上用官方仓库的工具完成

## 第一阶段：Google 调研

在 Google 搜索"arch pdf翻译工具"时，AI 模式（Gemini）给出了三种方案：

### 方案一：PDFMathTranslate (pdf2zh)

现代 AI 排版保留翻译工具，基于 Python，支持 OpenAI、DeepL、Ollama 等：

```bash
pip install pdf2zh
pdf2zh example.pdf
```

优点是排版保留好、支持公式，但它依赖 Python 和外部 AI API，不符合"纯 GNU 命令行"的要求。

### 方案二：Poppler + Translate-Shell（最终选择）

Arch 官方仓库自带的硬核组合：

```bash
sudo pacman -S poppler translate-shell
pdftotext input.pdf - | trans -b :zh
```

其中 `poppler` 提供 `pdftotext`，`translate-shell` 提供 `trans` 命令行翻译工具。这是最符合 Unix 哲学的方案——管道组合、文本流式处理。

### 方案三：AUR 的 pdf_translator

```bash
yay -S pdf_translator
```

基于 Python 和 translate-shell 的 AUR 小脚本，但维护不活跃。

**最终选择了方案二**，因为它最纯粹。

## 第二阶段：初次尝试 pdftotext 直接管道

Google AI 模式给出了详细的进阶用法：

### 基础管道命令

```bash
pdftotext input.pdf - | trans -b :zh
```

- `-`：输出到 stdout 而非文件
- `-b`：brief 模式，只输出翻译结果
- `:zh`：目标语言中文

### 指定页面范围

```bash
# 仅翻译第 5 到第 10 页
pdftotext -f 5 -l 10 input.pdf - | trans -b :zh
```

### 保留段落布局

```bash
# -layout 保持原稿物理布局，减少断句错误
pdftotext -layout input.pdf - | trans -b :zh
```

### 导出为文件

```bash
pdftotext -layout input.pdf - | trans -b :zh > translated.txt
```

### 更换翻译引擎

```bash
# Bing 翻译
pdftotext input.pdf - | trans -e bing -b :zh

# DeepL
pdftotext input.pdf - | trans -e deepl -b :zh

# OpenAI
pdftotext input.pdf - | trans -e openai -b :zh
```

### translate-shell 配置文件

```bash
# ~/.config/translate-shell/init.trans
{
    :default-engine "bing"
    :hl "zh"
    :tl "zh"
    :verbose false
}
```

看起来很美好——一行命令搞定翻译。但实际运行时...

## 第三阶段：pdftotext 翻车

对 `ctut.pdf` 执行 `pdftotext`，发现提取出来的文本是乱码或空白。这本 PDF 虽然是数字版而非扫描件，但它的文本层编码有问题，`pdftotext` 无法正常抽取。

对 `ctut.pdf` 执行 `pdfinfo`，发现 ‘Creator: 	dvips(k) 5.86 Copyright 1999 Radical Eye Software |\ Producer: 	GNU Ghostscript 7.05’，使用了极老旧的 LaTex 工具链生成的 PDF， 没有嵌入 *`ToUnicode`* ，即文档中不包含符号对应的真实字符编码。 

**关键结论：如果 `pdftotext` 提取不出有效文本，就必须走 OCR（光学字符识别）路线。**

于是思路转变：

```
pdftotext input.pdf - | trans -b :zh   ← 此路不通

改为：
PDF 页面 → 渲染为图片 → OCR 识别文字 → 翻译
```

OCR 测试中出现 *`Syntax Warning: Bad bounding box in Type 3 glyph`* ，该 PDF 采用了极其古老的 Type3 字体（通常是 90 年代 LaTex 编译时未开启矢量字体的结果）。这类字体本质上是一堆低分辨率的位图，因此`pdftoppm`在栅格化渲染时会报边界框错误。不过图片仍然正常生成，只是一条 Poppler 提示内部参数不规范的警告。

## 第四阶段：第一版 OCR 管线脚本 (translate_pipe.sh)

写了第一版脚本，核心思想是 Unix 管线式处理：

```bash
#!/usr/bin/env bash

PDF_FILE="ctut.pdf"
OUTPUT_DIR="/tmp/ctut_pages"
FINAL_DOC="ctut_translated.txt"
START_PAGE=1
END_PAGE=290

mkdir -p "$OUTPUT_DIR"
rm -f "$FINAL_DOC"

for i in $(seq $START_PAGE $END_PAGE); do
    PAGE_PAD=$(printf "%03d" $i)
    IMG_PATH="$OUTPUT_DIR/page-$PAGE_PAD.png"

    echo "--- 正在处理第 $i 页 ---" | tee -a "$FINAL_DOC"

    # 1. PDF 单页渲染为 PNG
    pdftoppm -png -f $i -l $i -r 200 "$PDF_FILE" "$OUTPUT_DIR/page" 2>/dev/null

    # 2. OCR → 去空行 → 翻译 → 过滤错误
    if [ -f "$IMG_PATH" ]; then
        cat "$IMG_PATH" | \
            tesseract stdin stdout -l eng 2>/dev/null | \
            grep -v '^[[:space:]]*$' | \
            trans -b :zh | \
            grep -v -E "\[ERROR\]|Oops\!" >> "$FINAL_DOC"

        rm -f "$IMG_PATH"
    fi

    sleep 2
done
```

运行：

```bash
chmod +x translate_pipe.sh
./translate_pipe.sh
```

### 第一版暴露的问题

终端输出：

```
=== 开始处理: ctut.pdf (第 1 页 至 第 290 页) ===
--- 正在处理第 1 页 ---
grep: 警告：stray \ before !
--- 正在处理第 2 页 ---
grep: 警告：stray \ before !
[ERROR] Null response.
[ERROR] Oops! Something went wrong and I can't translate it for you :(
```

**问题一：grep 警告**

```bash
grep -v -E "\[ERROR\]|Oops\!"
```

在 `grep -E`（ERE）中，`!` 不需要转义，`\!` 反而导致 `stray \ before !` 警告。修复：

```bash
grep -v -E '\[ERROR\]|Oops!'
```

**问题二：`trans` 大量空响应**

整页文本一次性扔给 `trans`，翻译服务限流返回 `Null response`。长文本一次翻译极易触发 Google 翻译的 IP 频率限制。

**问题三：无法断点续跑**

每次运行 `rm -f "$FINAL_DOC"` 删除已有结果。290 页的 PDF，中途失败就前功尽弃。

**问题四：没有文本分块**

代码、URL、C 语言片段被翻译器误改。

## 第五阶段：确立 GNU 风格约束

既然是翻译 GNU 手册，翻译风格必须符合 GNU 语境。整理了规则：

1. `free software` → "自由软件"（不是"免费软件"）
2. `GNU Free Documentation License` → "GNU 自由文档许可证"
3. 保留 C 语言关键字、函数名、变量名、命令、文件名、URL
4. 语气朴素直接，不要"商业教程腔"或"AI 总结腔"
5. `you` 翻译为"你"，不要客服腔"您"

同时明确技术约束：

- **只用 `trans -b`，不用 Ollama，不用本地 LLM**
- 保持 Bash/GNU 工具/文本流的处理方式

## 第六阶段：OCR-first GNU/trans 版本 (v1)

重写为 OCR-first 架构：

```bash
# 核心管线
pdftoppm -singlefile -png -r "$DPI" -f "$p" -l "$p" "$PDF_FILE" "$prefix"
tesseract "$img" "$ocr_base" -l eng --psm "$PSM"
# 英文清洗 → 分块 → trans -b → GNU 术语后处理 → 缓存
```

新增特性：
- 默认 OCR-first（`EXTRACTOR=ocr`）
- `trans -b en:zh-CN` 作为翻译器
- 支持翻译引擎选择（`ENGINE=bing`）
- DPI 和 PSM 可配置
- 文本分块（`MAX_CHARS=850`）
- 翻译块之间等待（`SLEEP_SEC=7`）

但 v1 仍有问题：要求旧译文 `ctut_translated.txt` 必须存在，否则直接报错退出。

## 第七阶段：v2 — 旧译文改为可选

运行 v1 时：

```
[ERROR] old translation not found: ctut_translated.txt
```

修复：旧译文不存在时继续运行，按全新 OCR/trans 模式工作：

```bash
USE_OLD_IF_FAIL=0 START_PAGE=1 END_PAGE=10 ./translate_pipe_gnu_trans_ocr_v2.sh
```

## 第八阶段：v3 — 修复 Bash bad substitution

运行 v2 出现：

```
./translate_pipe_gnu_trans_ocr_v2.sh: line 181: ${out.ocr.tmp%.txt}: bad substitution
```

原因：Bash 变量名不能包含点号。原代码：

```bash
${out.ocr.tmp%.txt}   # 错误！
```

修复：

```bash
ocr_tmp="$out.ocr.tmp"
ocr_base="${ocr_tmp%.txt}"
tesseract "$img" "$ocr_base" -l eng --psm "$PSM"
```

## 第九阶段：v4 — 修复 Tesseract 输出文件名判断 + 合并输出

运行 v3 时，每页都显示 "OCR failed"，但 `page-{i}.failed.txt` 为空。

分析发现：`ocr_tmp` 本身不以 `.txt` 结尾，`${ocr_tmp%.txt}` 根本没有截断。Tesseract 实际生成的文件是 `xxx.en.txt.ocr.tmp.txt`，但脚本检查的是 `xxx.en.txt.ocr.tmp`——永远找不到。

**v4 修复**：

```bash
# 正确的做法：告诉 tesseract 输出 basename，它自动加 .txt
ocr_base="$out.ocr.tmp"         # 这是 basename
ocr_tmp="$ocr_base.txt"         # tesseract 实际生成的文件
```

同时新增两个功能：

```bash
MERGE_FINAL=1   # 合并每页译文到最终 TXT
STDOUT=1        # 中文译文输出到终端（进度信息走 stderr）
```

测试结果：

```
--- 正在处理第 1 页 ---
[OK] 第 1 页完成
--- 正在处理第 2 页 ---
[OK] 第 2 页完成
--- 正在处理第 3 页 ---
[OK] 第 3 页完成
--- 正在处理第 4 页 ---
[WARN] page 4: OCR extraction failed.
--- 正在处理第 5 页 ---
[OK] 第 5 页完成
...
```

第 4 页"失败"了——但实际上它本来就是空白页！

## 第十阶段：v5（最终版）— 空白页不再误报

改进逻辑：OCR 后如果没有有效英文文本（`< 10` 个非空白字符），不调用 `trans`，直接标记为空白页：

```bash
en_nonspace=$(tr -d '[:space:]' < "$en_file" | wc -c | tr -d ' ')
if [[ "${en_nonspace:-0}" -lt 10 ]]; then
    printf '[本页为空白页，或 OCR 未识别出有效英文文本。]\n' > "$out_file"
    continue
fi
```

输出：

```
--- 正在处理第 4 页 ---
[BLANK] 第 4 页无有效 OCR 文本，已写入占位并跳过 trans
```

## 最终方案架构

```
┌─────────────────────────────────────────────────────────────────────┐
│  ctut.pdf                                                           │
│      │                                                              │
│      ▼                                                              │
│  pdftoppm -singlefile -png -r 300 -f $p -l $p                      │
│      │                                                              │
│      ▼                                                              │
│  tesseract page.png output -l eng --psm 3                           │
│      │                                                              │
│      ▼                                                              │
│  sed / awk / grep  (清洗：去控制符、统一引号、去空行)                │
│      │                                                              │
│      ▼                                                              │
│  split_records()  (分块：T=可翻译文本，P=保护的代码/URL)             │
│      │                                                              │
│      ▼                                                              │
│  trans -b en:zh-CN  (翻译，每块 ≤ 850 字符，间隔 7 秒)             │
│      │                                                              │
│      ▼                                                              │
│  gnu_postprocess()  (后处理："免费软件"→"自由软件"等)               │
│      │                                                              │
│      ▼                                                              │
│  page cache (ctut_gnu_ocr_work/zh/page-001.zh.txt)                  │
│      │                                                              │
│      ▼                                                              │
│  合并 → ctut_translated_gnu_ocr.txt                                 │
└─────────────────────────────────────────────────────────────────────┘
```

## 安装依赖（Arch Linux）

```bash
sudo pacman -S poppler tesseract tesseract-data-eng translate-shell coreutils gawk sed grep
```

验证：

```bash
trans -b en:zh-CN 'This is a test.'
# 输出：这是一个测试。
```

## 使用方法

### 测试前 10 页

```bash
chmod +x scripts/translate_pipe_gnu_trans_ocr.sh

USE_OLD_IF_FAIL=0 START_PAGE=1 END_PAGE=10 FORCE=1 \
  ./scripts/translate_pipe_gnu_trans_ocr.sh
```

### 跑全书

```bash
USE_OLD_IF_FAIL=0 START_PAGE=1 END_PAGE=290 \
  ./scripts/translate_pipe_gnu_trans_ocr.sh
```

### 输出到终端

```bash
STDOUT=1 USE_OLD_IF_FAIL=0 START_PAGE=1 END_PAGE=10 FORCE=1 \
  ./scripts/translate_pipe_gnu_trans_ocr.sh
```

### 同时保存到文件

```bash
STDOUT=1 USE_OLD_IF_FAIL=0 START_PAGE=1 END_PAGE=10 FORCE=1 \
  ./scripts/translate_pipe_gnu_trans_ocr.sh > terminal_output_zh.txt
```

### 指定翻译引擎

```bash
ENGINE=bing USE_OLD_IF_FAIL=0 START_PAGE=1 END_PAGE=290 \
  ./scripts/translate_pipe_gnu_trans_ocr.sh
```

### 调整 OCR 参数

```bash
DPI=360 PSM=6 USE_OLD_IF_FAIL=0 START_PAGE=1 END_PAGE=10 FORCE=1 \
  ./scripts/translate_pipe_gnu_trans_ocr.sh
```

### 降低 trans 空响应概率

```bash
MAX_CHARS=600 SLEEP_SEC=12 RETRIES=6 USE_OLD_IF_FAIL=0 \
  START_PAGE=1 END_PAGE=290 \
  ./scripts/translate_pipe_gnu_trans_ocr.sh
```

## 配置选项速览

| 变量 | 默认值 | 作用 |
| --- | --- | --- |
| `PDF_FILE` | `ctut.pdf` | 输入 PDF |
| `FINAL_DOC` | `ctut_translated_gnu_ocr.txt` | 合并后的中文输出 |
| `WORK_DIR` | `ctut_gnu_ocr_work` | 工作目录 |
| `START_PAGE` | `1` | 起始页 |
| `END_PAGE` | `290` | 结束页 |
| `EXTRACTOR` | `ocr` | `ocr` / `auto` / `pdftotext` |
| `DPI` | `300` | 渲染分辨率 |
| `PSM` | `3` | Tesseract 页面分割模式 |
| `ENGINE` | 空（Google） | `trans -b -e ENGINE` |
| `MAX_CHARS` | `850` | 每块最大字符数 |
| `SLEEP_SEC` | `7` | 翻译块间等待秒数 |
| `RETRIES` | `4` | 失败重试次数 |
| `FORCE` | `0` | 是否重跑已缓存页 |
| `MERGE_FINAL` | `1` | 是否合并输出 |
| `STDOUT` | `0` | 是否输出到终端 |
| `BLANK_MARK` | `1` | 是否标记空白页 |
| `USE_OLD_IF_FAIL` | `1` | 失败时是否用旧译文回退 |
| `ONLY_BAD` | `0` | 只重跑坏页 |

## 工作目录结构

```
ctut_gnu_ocr_work/
├── en/          # 每页 OCR 英文文本
├── zh/          # 每页中文译文缓存
├── old_zh/      # 旧译文切分的页面草稿
├── log/         # trans/tesseract/pdftoppm 日志
├── img/         # 临时页面图像（处理后删除）
└── tmp/         # 临时文件
```

## 关键设计决策

### 1. 为什么 OCR-first 而不是 pdftotext-first？

很多 PDF 的文本层编码有问题。`pdftotext` 对 `ctut.pdf` 提取出乱码。OCR 路线虽然慢，但对任何 PDF 都能工作。

脚本保留了 `EXTRACTOR=auto` 模式：先试 pdftotext，如果结果质量差（有效 ASCII 字母太少、控制字符太多），自动切换到 OCR。

### 2. 为什么要分块翻译？

整页一次性扔给 `trans`：
- 容易触发 Google 翻译限流，返回 `Null response`
- 代码片段被误翻译
- 一个块失败导致整页丢失

分块后每块 ≤ 850 字符，块间等待 7 秒，极大降低了失败率。

### 3. 为什么要区分 T（可翻译）和 P（保护）记录？

`split_records()` 函数用 AWK 判断每一行是否是代码/URL/C 语言语句：

```awk
function protected(line) {
    if (line ~ /https?:\/\//) return 1
    if (line ~ /^[[:space:]]*#(include|define|if|ifdef)/) return 1
    if (line ~ /;[[:space:]]*$/ && line ~ /[()=]/) return 1
    if (line ~ /==|!=|<=|>=|\+\+|--|&&|\|\|/) return 1
    ...
}
```

保护块直接原样输出，不送翻译器。

### 4. GNU 术语后处理

翻译器可能把 "free software" 翻译成 "免费软件"。`gnu_postprocess()` 自动修正：

```bash
gnu_postprocess() {
    sed -E \
        -e 's/免费软件/自由软件/g' \
        -e 's/免费文档/自由文档/g' \
        -e 's/GNU免费文档许可证/GNU 自由文档许可证/g' \
        -e 's/原始码/源代码/g' \
        -e 's/连结器/链接器/g' \
        -e 's/函式/函数/g' \
        -e 's/阵列/数组/g' \
        -e 's/字串/字符串/g' \
        -e 's/您/你/g'
}
```

### 5. 断点续跑

每页译文缓存到 `ctut_gnu_ocr_work/zh/page-XXX.zh.txt`。下次运行时已完成的页直接跳过：

```bash
if [[ -s "$out_file" && "$FORCE" != 1 ]]; then
    say "--- 第 $p 页已存在，跳过 ---"
    continue
fi
```

中途 Ctrl-C 不丢失进度。

## 试错过程时间线

| 时间 | 事件 |
| --- | --- |
| 开始 | Google AI 模式推荐三种方案，选择 poppler + translate-shell |
| 早期 | `pdftotext ctut.pdf - \| trans -b :zh` 发现文本层乱码 |
| v1 | 写第一版 OCR 管线，发现 grep 警告、trans 限流、无法续跑 |
| 中间 | 曾考虑 Python 校译方案和 Ollama，最终否决，坚持 GNU 风格 |
| v2 | 旧译文改为可选，不再强制要求 |
| v3 | 修复 Bash bad substitution（变量名不能含点号） |
| v4 | 修复 Tesseract 输出文件名 bug + 增加合并和 STDOUT 功能 |
| v5 | 空白页不再误报为 OCR 失败，增加 BLANK_MARK |
| 最终 | 稳定版定名为 `scripts/translate_pipe_gnu_trans_ocr.sh` |

## 总结

这个项目从一个简单的 `pdftotext | trans` 管道，经过多次试错，演化为一个完整的、可断点续跑的、符合 Unix 哲学的 PDF 翻译管线。

核心工具链：
- **pdftoppm**（Poppler）：PDF 页面渲染为 PNG
- **tesseract**：OCR 识别英文
- **sed / awk / grep**：文本清洗、分块、保护代码
- **trans**（translate-shell）：翻译为中文
- **sed**：GNU 术语后处理

没有 Python，没有 Node.js，没有 Docker，没有大模型 API Key（除非你自己配 trans 引擎）。纯粹的 Bash + GNU 工具 + 管道。

这就是 Unix 哲学：每个工具做一件事，通过管道组合完成复杂任务。

---

项目地址：[gnu-pdf-trans-pipe](https://github.com/gsioeo/gnu-pdf-trans-pipe)

```
git clone git@github.com:gsioeo/gnu-pdf-trans-pipe.git
cd gnu-pdf-trans-pipe
chmod +x scripts/translate_pipe_gnu_trans_ocr.sh
USE_OLD_IF_FAIL=0 START_PAGE=1 END_PAGE=10 FORCE=1 ./scripts/translate_pipe_gnu_trans_ocr.sh
```
