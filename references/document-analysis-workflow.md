# 协议文档分析标准流程（Phase 0）

本文件是 `protocol-integration-workflow` 的 **Phase 0 补充**。
目标：把一份 PDF / Word / Markdown 格式的协议文档，系统性地转化为实现所需的全部结构化信息。

**原则**：在所有代码编写之前完成本流程。Phase 0 的输出是 Phase 1（config.yaml + commands.yaml）和 Phase 2（codec.py）的输入。

---

## 目录

1. [文档预处理：从二进制到可搜索文本](#1-文档预处理从二进制到可搜索文本)
2. [帧格式提取：精确定位每一字节](#2-帧格式提取精确定位每一字节)
3. [命令表提取：所有命令与字段](#3-命令表提取所有命令与字段)
4. [CRC 与校验算法提取](#4-crc-与校验算法提取)
5. [安全层参数提取](#5-安全层参数提取)
6. [错误码表提取](#6-错误码表提取)
7. [测试向量提取（关键）](#7-测试向量提取关键)
8. [交叉验证：三方互校](#8-交叉验证三方互校)
9. [输出物检查清单](#9-输出物检查清单)

---

## 1. 文档预处理：从二进制到可搜索文本

### 1.1 PDF 文档

PDF 是协议文档最常见的格式，也是信息提取最困难的格式。按优先级选择工具：

**首选：pymupdf（fitz）**

```bash
pip install pymupdf
```

```python
import fitz
doc = fitz.open("protocol_v2.0.pdf")
full_text = ""
for page in doc:
    full_text += page.get_text() + "\n--- PAGE BREAK ---\n"
Path("protocol_v2.0_extracted.txt").write_text(full_text, encoding="utf-8")
```

**备选：pdfplumber（表格提取更强）**

pdfplumber 对表格的提取质量优于 pymupdf，但文字提取速度较慢。建议先用 pymupdf 提取全文，再用 pdfplumber 专门处理表格页。

```python
import pdfplumber
with pdfplumber.open("protocol_v2.0.pdf") as pdf:
    for page in pdf.pages:
        tables = page.extract_tables()
        for table in tables:
            # table 是二维列表，第一行通常是表头
            print(table)
```

**常见问题与对策**：

| 问题 | 表现 | 对策 |
|------|------|------|
| 表格跨页 | 表头在上一页，数据行在下一页 | 合并相邻页输出后重新识别表格边界 |
| 图表中的帧格式 | `get_text()` 返回空或乱码 | 截图后人工对照，手工录入帧格式模板 |
| 公式/算法图片化 | 无法提取成文本 | 从正文描述中反推参数；必要时手工录入 |
| 中文字符乱码 | 提取结果是乱码 | 检查 PDF 内嵌字体；尝试 `page.get_text("rawdict")` 逐字符提取 |
| 页眉页脚混入正文 | 每页都有重复的"第X页/共Y页" | 正则过滤 `第\d+页` 模式 |
| 两栏排版顺序错乱 | 文字按栏交错 | 使用 `page.get_text("blocks")` 按坐标排序 |

### 1.2 Word 文档 (.docx)

```bash
pip install python-docx
```

```python
from docx import Document
doc = Document("protocol_v3.1.docx")
full_text = ""
for para in doc.paragraphs:
    full_text += para.text + "\n"

# 提取表格
for i, table in enumerate(doc.tables):
    full_text += f"\n--- TABLE {i+1} ---\n"
    for row in table.rows:
        cells = [cell.text.strip() for cell in row.cells]
        full_text += " | ".join(cells) + "\n"
```

### 1.3 Markdown 文档

直接读取即可，无需特殊处理。注意检查表格的列对齐和代码块中的 hex dump。

### 1.4 预处理后的人工清理

无论哪种格式，工具提取后的文本都需要人工快速过一遍：

1. **删除页眉页脚**（正则 `第\s*\d+\s*页`、`Page \d+`）
2. **恢复跨页断裂的段落**（删除孤立的 `--- PAGE BREAK ---`）
3. **标记表格区域**（用 `>>> TABLE START` / `>>> TABLE END` 标记）
4. **标记代码块/hex dump 区域**（用 `>>> HEX START` / `>>> HEX END` 标记）
5. **标注无法自动提取的内容**（用 `[MANUAL: 此处需从原文档手动录入]` 标记）

---

## 2. 帧格式提取：精确定位每一字节

### 2.1 帧格式搜索关键词

在协议文档中搜索以下关键词来定位帧格式描述：

**中文关键词**：`帧格式` `帧结构` `数据帧` `报文格式` `包格式` `消息结构` `通信格式`
**英文关键词**：`frame format` `packet structure` `message format` `data frame` `telegram`

### 2.2 帧格式文档化模板

从文档中提取的帧格式信息，**必须**填入以下模板，不允许有模糊地带：

```
帧格式名称: <协议简称> 帧
帧定界方式:
  - 定长帧 / 帧头帧尾 / HDLC 0x7E 转义 / 空闲超时分界

多字节整数字节序: 大端 (Big-Endian) / 小端 (Little-Endian)

┌───────┬──────┬──────┬──────────┬──────┬────────┬─────────┬─────────┬───────┐
│ Byte  │  0   │  1   │  2..3    │  4   │  5..6   │  7..8    │ 9..N-3  │N-2.. │
│       │      │      │          │      │         │         │         │ N-1  │
├───────┼──────┼──────┼──────────┼──────┼─────────┼─────────┼─────────┼───────┤
│ 字段  │ HEAD │ VER  │  SEQ     │ TYPE │  CMD    │  LEN     │ PAYLOAD │ CRC16 │
├───────┼──────┼──────┼──────────┼──────┼─────────┼─────────┼─────────┼───────┤
│ 长度  │  2   │  1   │   2      │  1   │   2     │   2      │  LEN    │   2   │
├───────┼──────┼──────┼──────────┼──────┼─────────┼─────────┼─────────┼───────┤
│ 值/说明│ 固定  │ 0x01 │ 递增,    │ 0x01 │  见命令 │  N≤1024 │ 见命令  │ CRC16 │
│       │0x55AA│      │ RSP回显  │=REQ  │   表    │         │   定义   │XMODEM │
│       │      │      │          │0x02  │         │         │         │       │
│       │      │      │          │=RSP  │         │         │         │       │
│       │      │      │          │0x03  │         │         │         │       │
│       │      │      │          │=EVT  │         │         │         │       │
└───────┴──────┴──────┴──────────┴──────┴─────────┴─────────┴─────────┴───────┘

帧总长范围: 最小=<N> bytes (无payload时), 最大=<N> bytes

CRC 参数:
  - 算法名称: CRC16-XMODEM
  - 多项式: 0x1021
  - 初始值: 0x0000
  - 输入反转: 否
  - 输出反转: 否
  - 最终异或: 0x0000
  - 校验范围: VER(byte2) 到 PAYLOAD 末尾（不含帧头/CRC自身）
  - 结果字节序: 大端写入帧尾

转义规则（如适用）:
  - 转义字符: 0x7D
  - 转义规则: 0x7E→0x7D 0x5E, 0x7D→0x7D 0x5D
  - 校验范围: 转义前 / 转义后 的字节? → <明确>
```

### 2.3 从文档中提取帧格式的策略

**策略 A：文档有 ASCII 帧图**

直接对照 ASCII 图填写模板。注意 ASCII 图可能省去长度字段或 CRC 的展示。

**策略 B：文档有表格描述**

逐行提取表格中的"字段名/长度/说明"三列，按字节顺序排列。注意：
- 有些表格是"从上到下=从低字节到高字节"
- 有些表格按语义分组而不是字节顺序
- 必须结合示例帧 hex dump 验证

**策略 C：文档只有文字叙述**

从叙述中提取字节级信息。关键词：
- "帧头"、"起始标志"、"SOF"、"HEAD"、"前导码"、"同步字"
- "帧尾"、"结束标志"、"EOF"、"TAIL"
- "长度域"、"LEN"、"数据长度"、"报文长度"
- "命令字"、"功能码"、"CMD"、"FUNC"、"控制码"
- "校验"、"CRC"、"校验和"、"CS"、"MAC"、"签名"

**策略 D：文档仅给出 hex dump 示例**

这是最困难的情况。从 hex dump 反推帧格式：

```python
# 示例：分析 GET_INFO_REQ = 55AA010100010001000062D3
# 已知信息：REQ 无 payload
hex_str = "55AA010100010001000062D3"
data = bytes.fromhex(hex_str)

# 逐字节假设验证
# 55AA → 帧头 2 bytes ✓
# 01   → 可能是 VER
# 01   → 可能是 TYPE（REQ=1）
# 0001 → 可能是 SEQ
# 0001 → 可能是 CMD
# 0000 → 可能是 LEN=0
# 62D3 → 可能是 CRC16

# 验证假设：如果有另一条命令的 hex dump，交叉验证
```

### 2.4 帧格式提取后的验证

- [ ] 帧头/帧尾的字节值是否与其他 hex dump 一致？
- [ ] LEN 字段的值是否等于 PAYLOAD 的实际长度？
- [ ] CRC 覆盖范围是否与文档描述一致？
- [ ] 如果有示例帧的"解析说明"，逐字段对照核对

---

## 3. 命令表提取：所有命令与字段

### 3.1 命令表搜索关键词

**中文**：`命令码` `功能码` `指令表` `命令列表` `命令定义` `通信命令` `报文类型`
**英文**：`command list` `function code` `command code` `instruction set` `opcode`

### 3.2 命令表提取模板

对于协议文档中的每个命令，填写以下结构化信息：

```
命令名: <命令的中文/英文名>
command_id: 0x<hex>  (文档中可能的写法: 0x0001 / 00 01 / 0001H / 01H)
方向: REQ(下行) / RSP(上行) / 双向

# REQ (下行) 字段:
| 序号 | 字段名 | 类型(文档原文) | 推断 FieldType | 长度 | 默认值 | 说明 |
|------|--------|---------------|----------------|------|--------|------|
| 1    | ...    | u16           | uint16         | 2    | -      | ...  |
| 2    | ...    | u8            | uint8          | 1    | 0      | 保留 |

# RSP (上行) 字段:
| 序号 | 字段名 | 类型(文档原文) | 推断 FieldType | 长度 | 默认值 | 说明 |
|------|--------|---------------|----------------|------|--------|------|
| 1    | result_code | u16      | uint16         | 2    | -      | 结果码 |
| 2    | ...    | ...           | ...            | ...  | ...    | ...  |
```

### 3.3 字段提取的常见陷阱

**陷阱 1：字段顺序 ≠ 字节顺序**

文档可能按"语义分组"而不是"字节顺序"列出字段。**必须以字节顺序为准**。
验证方法：找一条该命令的 hex dump 示例，逐字节对照。

**陷阱 2：隐式字段**

有些字段在表格里没列，但在帧格式总表中定义了（如所有 RSP 都带 result_code）。
必须把帧格式中的"通用字段"合并到每条命令的字段列表里。

**陷阱 3：保留字段**

文档中标注"保留"/"Reserved"/"RFU"的字段，也必须出现在字段列表中，`default` 填文档指定的值（通常是 `0` 或 `0xFF`）。

**陷阱 4：长度字段 + 数据字段的联动**

当 `length` 字段表示后续 `data` 字段的长度时，`data` 字段使用 `raw_hex` + `dynamic: true`：

```yaml
- name: length
  type: uint16
  length: 2
- name: data
  type: raw_hex
  length: 0      # 占位，实际由 dynamic 控制
  dynamic: true
```

**陷阱 5：变长帧**

如果整个帧的长度可变（不是固定命令+固定字段），需要判断：
- 仅 `data` 字段变长 → 使用 `raw_hex` + `dynamic: true`
- 整个字段结构可变（如 TLV 格式）→ 可能需要 `record_array` 或在 codec 中用 `decoded_fields_mode: replace` 自行解析

**陷阱 6：可选字段**

如果文档说"当 flag=X 时，附加字段 Y"：
- 简单情况：在 codec 中用条件逻辑处理
- 复杂情况：在 commands.yaml 中为不同 flag 值定义不同的命令变体

### 3.4 从 PDF 表格到 commands.yaml 的实战步骤

```python
# 步骤 1: 用 pdfplumber 提取命令表
import pdfplumber
import yaml

with pdfplumber.open("protocol.pdf") as pdf:
    for page in pdf.pages:
        tables = page.extract_tables()
        for table in tables:
            if table and "命令码" in str(table[0]):
                # 步骤 2: 解析表格行
                for row in table[1:]:  # 跳过表头
                    cmd_hex = row[0]   # 命令码列
                    cmd_name = row[1]  # 命令名列
                    direction = row[2] # 方向列
                    fields_raw = row[3]  # 字段描述列
                    # 步骤 3: 从字段描述列解析出字段列表
                    # 步骤 4: 映射文档类型 → FieldType
                    # 步骤 5: 生成 commands.yaml 结构
```

---

## 4. CRC 与校验算法提取

### 4.1 搜索关键词

**中文**：`CRC` `校验` `校验和` `循环冗余` `CS` `异或和` `累加和`
**英文**：`checksum` `CRC-16` `CRC-32` `XOR checksum` `FCS`

### 4.2 CRC 参数提取清单

从协议文档中提取以下参数（如有缺失，标注 `[文档未明确，需从示例帧反推]`）：

| 参数 | 可选值/示例 | 提取优先级 |
|------|-----------|----------|
| 算法名称 | CRC16-IBM / CRC16-XMODEM / CRC16-CCITT / CRC32-IEEE / 自定义 | 必须 |
| 多项式 (Poly) | 0x1021 / 0x8005 / 0x04C11DB7 | 必须 |
| 初始值 (Init) | 0x0000 / 0xFFFF / 0xFFFFFFFF | 必须 |
| 输入反转 (RefIn) | true / false | 必须 |
| 输出反转 (RefOut) | true / false | 必须 |
| 最终异或 (XorOut) | 0x0000 / 0xFFFF / 0xFFFFFFFF | 必须 |
| 校验范围 | 从哪个字节开始，到哪个字节结束（含/不含CRC自身） | 必须 |
| 结果字节序 | 大端/小端写入帧尾 | 必须 |
| 是否需要转义后计算 | 是/否 | 需要转义时必须 |

### 4.3 从示例帧反推 CRC 参数

当文档未明确 CRC 参数时，可以利用多个示例帧反推：

```python
# 使用 crccheck 库穷举常见 CRC 参数
# pip install crccheck

from crccheck.crc import Crc16, Crc32

# 示例：多条帧（已知 PAYLOAD，已知帧尾 CRC 值）
frames = [
    ("01000100010000", "62D3"),  # (校验范围hex, CRC值hex)
    ("0101000200020004", "F9BD"),
]

# 穷举常见 CRC16 参数，检查哪个匹配
# ...（详细代码见 test-vector-extraction.md）
```

### 4.4 非 CRC 校验

| 校验类型 | 描述 | 字节长度 |
|---------|------|---------|
| 累加和 (SUM8) | 所有字节累加，取低 8 位 | 1 |
| 异或和 (XOR8) | 所有字节异或 | 1 |
| 补码和 (2's complement) | 累加和取补码 | 1 |
| 纵向冗余校验 (LRC) | 每字节累加，取补码 | 1 |
| CS (Checksum) | 同累加和，但可能是 16 位 | 1-2 |

---

## 5. 安全层参数提取

### 5.1 搜索关键词

**中文**：`加密` `解密` `安全` `密钥` `认证` `MAC` `签名` `鉴权`
**英文**：`encrypt` `decrypt` `security` `AES` `DES` `3DES` `SM4` `HMAC` `authentication` `cipher`

### 5.2 安全层信息提取清单

- [ ] 加密算法：AES-ECB / AES-CBC / DES-ECB / 3DES / SM4 / 自定义
- [ ] 加密模式：ECB / CBC / CTR / GCM
- [ ] 填充方式：PKCS7 / Zero Padding / No Padding
- [ ] 密钥长度：128 / 192 / 256 bits
- [ ] 密钥来源：固定密钥 / 密钥卡 / 注册帧协商 / 主密钥+分散因子
- [ ] 加密范围：整个 payload / payload 前 N 字节 / 特定字段
- [ ] MAC 算法：HMAC-SHA256 / AES-CMAC / CBC-MAC / 自定义
- [ ] MAC 长度：4 / 8 / 16 / 32 bytes
- [ ] MAC 覆盖范围：加密前数据 / 加密后数据 / 哪段字节
- [ ] 安全模式决策规则：同一协议中不同功能码是否使用不同安全策略

### 5.3 密钥派生记录模板

如果协议涉及密钥协商，记录完整的派生链：

```
步骤 1: <注册帧返回随机码>
  ↓
步骤 2: 派生密钥 = <算法>(<主密钥>, <随机码>)
  算法: HMAC-SHA256 / AES-ECB / 自定义
  输入: 主密钥(来自用户配置) + 随机码(字节偏移 M..N)
  输出: 取前 16 bytes (128-bit) 或 32 bytes (256-bit)
  ↓
步骤 3: 如有 MAC 密钥
  MAC 密钥 = <算法>(<主密钥>, <随机码>)
  ↓
步骤 4: 注入 session_updates
  - enc_key → session.state
  - mac_key → session.state
```

### 5.4 安全模式映射（如适用）

| 功能码 | 方向 | 安全模式 | 说明 |
|--------|------|---------|------|
| 0x04 | 下行 | none | 无数据域，无加密无 MAC |
| 0x05 | 下行 | cipher_mac | 密文 + MAC |
| 0x09 | 上下行 | plain_mac | 明文 + MAC |

---

## 6. 错误码表提取

### 6.1 搜索关键词

**中文**：`错误码` `返回码` `状态码` `result_code` `异常码` `响应码`
**英文**：`error code` `status code` `return code` `response code`

### 6.2 错误码记录模板

```yaml
error_codes:
  "0x0000": {name: "OK", description: "成功"}
  "0x0001": {name: "ERR_UNKNOWN_CMD", description: "未知命令"}
  "0x0002": {name: "ERR_BAD_PARAM", description: "参数非法"}
  # ... 逐条列出
```

错误码用于：
1. `codec.unpack()` 解析响应帧时判断是否成功
2. GUI 展示中文错误消息
3. 自动化测试脚本断言 `result_code` 为 `0x0000`

---

## 7. 测试向量提取（关键）

这是确保"组帧和解包可以测试"的核心步骤。详细操作指南见 `references/test-vector-extraction.md`。

### 7.1 测试向量的识别特征

在文档中搜索：
- hex dump 块（连续的大写 hex 字符串，如 `55AA010100010001000062D3`）
- "示例"、"举例"、"例如"、"Example"、"Sample"
- 包含逐字段解析说明的段落（"SOF=55AA"、"CMD=0001" 等）
- 附录中的"报文示例"、"通信示例"章节

### 7.2 最少需要提取的向量

| 优先级 | 向量类型 | 数量 | 用途 |
|--------|---------|------|------|
| P0 | 简单请求（无 payload 或最小 payload） | ≥1 | 验证帧头/帧尾/CRC 正确 |
| P0 | 带 payload 请求 | ≥1 | 验证字段序列化正确 |
| P0 | 响应帧 | ≥1 | 验证解析逻辑正确 |
| P1 | 错误响应帧 | ≥1 | 验证错误码提取 |
| P1 | 加密帧 | ≥1 | 验证安全层正确 |

---

## 8. 交叉验证：三方互校

在完成上述提取后，**必须**进行三方互校，这是发现理解错误的最有效手段：

```
帧格式描述（文档文字/图表）
         ↕ 互校
命令表定义（字段列表）
         ↕ 互校
测试向量（hex dump + 解析说明）
```

**互校规则**：

1. **帧格式 vs 命令表**：命令表的字段总长度是否与 LEN 字段的值匹配？
2. **帧格式 vs 测试向量**：按帧格式模板解码 hex dump，逐字节是否吻合？
3. **命令表 vs 测试向量**：将 hex dump 的 payload 部分按命令表字段解析，值是否与文档解析说明一致？

**常见不一致情况及处理**：

| 不一致表现 | 可能原因 | 处理方式 |
|----------|---------|---------|
| hex dump 字节数 ≠ 帧格式总长 | 帧格式描述漏了字段或有冗余字段 | 以 hex dump 为准修正帧格式 |
| 字段解析值 ≠ 文档说明 | 字节序搞反 / 字段边界偏移 | 逐字节核对，修正偏移 |
| payload 长度 ≠ LEN 值 | CRC 范围理解错误 / 填充字节 | 确认 LEN 包含哪些部分 |
| CRC 值对不上 | CRC 参数/范围/字节序错误 | 穷举常见参数反推 |

---

## 9. 输出物检查清单

Phase 0 完成后，以下产物必须交付：

- [ ] 预处理后的可搜索文本文件（`<协议名>_extracted.txt`）
- [ ] 帧格式文档化模板（每个字节的偏移/长度/含义/常量值全部填写完毕）
- [ ] 完整命令表（每条命令的 request/response 字段，含推断的 FieldType）
- [ ] 类型映射表（协议文档原始类型描述 → FieldType 注册名）
- [ ] CRC/校验参数（所有参数全部确认，含校验范围）
- [ ] 安全层参数（加密算法/密钥来源/MAC 算法，密钥派生链）
- [ ] 错误码表（所有错误码及其中文含义）
- [ ] 测试向量列表（每条 hex dump + 逐字段解析 + 推断预期值）
- [ ] 三方互校记录（帧格式 vs 命令表 vs 测试向量的不一致项及处理）
- [ ] 标注清单（文档中哪些信息不明确、需要与硬件/固件方确认）
