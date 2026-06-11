# 协议文档类型描述 → FieldType 映射指南

> 📏 ~2,000 tokens | 必读等级: ★★☆（Phase 1 对照用，非通读）| 前置: SKILL.md §4
> ⏩ 如果你只需要快速查类型，跳到 §9 快速决策矩阵

配合 `protocol-integration-workflow` 和 `document-analysis-workflow.md` 使用。
本文档覆盖所有 22 种已注册 FieldType 的协议文档描述识别、映射规则和配置要点。

---

## 1. 映射速查总表

| 协议文档典型描述 | FieldType | length | 特殊配置 |
|-----------------|-----------|--------|---------|
| "1 字节无符号整数"/ "u8" / "uint8" / "byte" | `uint8` | 1 | - |
| "2 字节无符号整数"/ "u16" / "uint16" / "word" | `uint16` | 2 | byte_order |
| "4 字节无符号整数"/ "u32" / "uint32" / "dword" | `uint32` | 4 | byte_order |
| "2 字节有符号整数"/ "s16" / "int16" | `int16` | 2 | byte_order |
| "4 字节有符号整数"/ "s32" / "int32" | `int32` | 4 | byte_order |
| "N 字节十六进制"/ "hex[16]" / "固定长度原始数据" | `hex` | N | - |
| "变长数据"/ "data[length]" / "动态数据区" | `raw_hex` | 0 | dynamic: true |
| "N 字节字符串"/ "char[N]" / "UTF-8" / "ASCII" | `string` | N | - |
| "无符号 BCD 码"/ "8421 码" / "压缩 BCD" | `bcd_u` | N | - |
| "有符号 BCD 码"/ "带符号字节的 BCD" / "±金额 BCD" | `bcd` | N | signed_layout |
| "BCD 数组" / "多个 BCD 值连续" | `bcd_u_array` | N | - |
| "年/月/日/时/分/秒 BCD" / "YYMMDDhhmmss" | `yymmddhhmmss` | 6 | - |
| "年/月/日/时/分 BCD" | `yymmddhhmm` | 5 | - |
| "年/月/日/时 BCD" | `yymmddhh` | 4 | - |
| "年/月/日 BCD" / "日期 BCD" | `yymmdd` | 3 | - |
| "时/分/秒 BCD" | `hhmmss` | 3 | - |
| "时/分 BCD" | `hhmm` | 2 | - |
| "年/月 BCD" | `yymm` | 2 | - |
| "位域" / "D0=xxx, D1=xxx" / "bit 0~7" | `bitfield` | N | bits: [...] |
| "枚举" / "0=xxx, 1=yyy" / "状态码" | `enum` | N | enum_values: {...} |
| "嵌套结构体" / "TLV" / "子记录" | `record` | N | 见下文 |
| "结构体数组" / "重复记录 N 条" | `record_array` | N | 见下文 |

---

## 2. 整数类型详细映射

### 2.1 uint8 — 1 字节无符号整数

**协议文档描述识别**：
- `u8` `uint8` `byte` `unsigned char` `1字节无符号` `无符号8位`
- 范围 0~255
- 常见用途：标志位、通道号、状态码（非枚举时）、版本号

**config 示例**：
```yaml
- name: channel
  type: uint8
  length: 1
  description: 设备通道号，从 1 开始
```

### 2.2 uint16 — 2 字节无符号整数

**协议文档描述识别**：
- `u16` `uint16` `word` `unsigned short` `2字节无符号` `无符号16位`
- 范围 0~65535
- 常见用途：结果码、长度值、电压值（缩放后）

**字节序判断**：
- 文档明确说 "高字节在前" / "大端" / "MSB first" → `byte_order: big`（默认）
- 文档明确说 "低字节在前" / "小端" / "LSB first" → `byte_order: little`
- 文档未明确 → 查看示例 hex dump 中该字段的两个字节顺序

**config 示例**：
```yaml
- name: result_code
  type: uint16
  length: 2
  byte_order: big
  description: 结果码，0x0000=成功
```

### 2.3 uint32 — 4 字节无符号整数

**协议文档描述识别**：
- `u32` `uint32` `dword` `unsigned int` `4字节无符号` `无符号32位`
- 范围 0~4294967295
- 常见用途：时间戳、地址、电压值（uV）、CRC32

**config 示例**：
```yaml
- name: target_uv
  type: uint32
  length: 4
  byte_order: big
  unit: uV
  description: 目标输出电压（微伏）
```

### 2.4 int16 / int32 — 有符号整数

**协议文档描述识别**：
- `s16` `int16` `signed short` `2字节有符号`
- `s32` `int32` `signed int` `4字节有符号`
- 常见用途：温度（可能为负）、差值、偏移量

**config 示例**：
```yaml
- name: temperature_offset
  type: int16
  length: 2
  byte_order: big
  unit: "0.1°C"
  scale: 0.1
  description: 温度补偿偏移（有符号）
```

---

## 3. 十六进制/原始数据类型详细映射

### 3.1 hex — 定长十六进制块

**适用场景**：
- 预留字段（内容不关心但必须占位）
- 固定长度的二进制标识符（如 UID、序列号原始字节）
- 不需要结构化解析的数据块

**config 示例**：
```yaml
- name: reserved
  type: hex
  length: 4
  default: "00000000"
  description: 保留字段
```

### 3.2 raw_hex — 变长十六进制数据（关键类型）

**适用场景**：
- EEPROM/Flash 写入的 data 段（长度由前面的 length 字段指定）
- 固件升级数据块
- 任意长度的二进制 payload

**必须配合 `dynamic: true` 和 `length: 0`**：

```yaml
- name: length
  type: uint16
  length: 2
  description: 后续数据长度（字节数）
- name: data
  type: raw_hex
  length: 0
  dynamic: true
  description: 写入数据（长度由 length 字段指定）
```

**原理**：`BinaryFormatter._normalize_dynamic_hex()` 对 `raw_hex` + `dynamic: true` 的字段不做长度补齐/截断，原样转为 bytes。

**常见错误**：
- 给 `raw_hex` 设置了错误的 `length` → 数据被截断或补零
- 忘记 `dynamic: true` → 按固定长度处理

---

## 4. 字符串类型详细映射

### 4.1 string — 定长字符串

**协议文档描述识别**：
- `string` `char[]` `ASCII` `UTF-8` `字符串` `字符数组`
- 文档说 "16 字节字符串，不足补 0x00"

**config 示例**：
```yaml
- name: fw_version
  type: string
  length: 16
  description: 固件版本字符串（UTF-8，不足补 0x00）
```

**默认值**：空字符串 `""`，序列化时按 length 补零。

---

## 5. BCD 类型详细映射

### 5.1 bcd_u — 单个无符号 BCD 值

**适用场景**：
- "6 字节 BCD 编码的用气量"
- "4 字节 8421 码表示的累积脉冲数"

**config 示例**：
```yaml
- name: total_usage
  type: bcd_u
  length: 6
  description: BCD 编码的累计用气量
```

### 5.2 bcd_u_array — BCD 数组

**适用场景**：
- 多个 BCD 值连续排列（如 12 个月的用气量，每月 4 字节 BCD）

### 5.3 bcd — 有符号 BCD（含符号字节）

**适用场景**：
- "8 字节有符号 BCD 金额，首位是符号字节"
- "带正负号和小数点的 BCD 数值（如余额、温度、压力）"
- 协议文档提到 "符号+数值" 或 "sign byte + BCD digits"

**默认行为**：
- `length == 8` 时，自动启用 `signed_layout`，默认 `integer_bytes=4, decimal_bytes=3`
  （即 1 字节符号 + 4 字节整数 + 3 字节小数）
- 其他长度默认行为与 `bcd_u` 一致（无符号），除非显式设置 `signed_layout: true`

**符号字节约定**：
- `0x00`：正数或零
- `0x01`：负数

**config 示例（显式声明）**：
```yaml
- name: 剩余金额
  type: bcd
  length: 8
  signed_layout: true
  integer_bytes: 4
  decimal_bytes: 3
  unit: 元
  description: "有符号 BCD 金额，8字节=1字节符号+4字节整数+3字节小数"
```

**编码示例**（`integer_bytes=4`, `decimal_bytes=3`）：
- `123.456` → `00 00 00 01 23 45 60 00`
- `-567.89` → `01 00 00 05 67 89 00 00`
- `0` → `00 00 00 00 00 00 00 00`

**字段属性清单**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `signed_layout` | bool | 是否启用有符号布局，默认 `length==8` 时自动为 `true` |
| `integer_bytes` | int | 整数部分字节数，默认 `4` (length=8) / `length-1` (其他) |
| `decimal_bytes` | int | 小数部分字节数，默认 `3` (length=8) / `length-1-integer_bytes` (其他) |
| `byte_bias` | int | 字节偏移（与 `bcd_u` 相同），默认 0 |

### 5.4 日期时间类型（BCD/HEX 双编码模式）

所有日期时间类型（`yymmddhhmmss` ~ `yymm`）均继承自 `DateTimeField`，**默认使用 BCD 编码**，同时支持 `encoding: hex` 切换为 HEX 编码。

#### 编码模式对比

| 编码 | 行为 | 适用场景 | 示例 |
|------|------|---------|------|
| `bcd`（默认） | 每字节高/低 nibble 各存一个十进制数字 | 电力/燃气行业（DL/T 645 风格） | `0x26 0x06 0x10` → "260610" |
| `hex` | 每字节解释为 uint8 十进制值（0~255） | 部分自定义协议（非标准 BCD） | `0x1A 0x06 0x0A` → "260610" |

**选择规则**：
- 协议文档未提及编码方式 → 默认 `bcd`（不写 `encoding` 字段）
- 协议文档明确 "每字节存十进制值"/"非 BCD"/"uint8 编码的时间" → `encoding: hex`
- 不确定时：检查协议的示例 hex dump。如果 `2026-06-10` 的字节是 `20 06 10`，说明是 HEX 编码；如果是 `26 06 10`，则是 BCD 编码。

**config 示例（HEX 编码的日期）**：
```yaml
- name: 当前时间
  type: yymmddhhmmss
  length: 6
  encoding: hex
  description: 每字节存十进制值（非 BCD），如 2026-06-10 22:15:30 编码为 26 06 10 22 15 30
```

#### 日期时间格式一览（7 种）

| 格式 | 长度 | 示例值 | 说明 |
|------|------|--------|------|
| `yymmddhhmmss` | 6 | `260610221530` | 2026-06-10 22:15:30 |
| `yymmddhhmm` | 5 | `2606102215` | 2026-06-10 22:15 |
| `yymmddhh` | 4 | `26061022` | 2026-06-10 22 时 |
| `yymmdd` | 3 | `260610` | 2026-06-10 |
| `hhmmss` | 3 | `221530` | 22:15:30 |
| `hhmm` | 2 | `2215` | 22:15 |
| `yymm` | 2 | `2606` | 2026-06 |

**选择规则**：协议文档中该字段用几个字节表示时间，就选对应格式。例如文档说 "5 字节时间 YYMMDDhhmm" → 选 `yymmddhhmm`。

---

## 6. 复合类型详细映射

### 6.1 bitfield — 位域

**适用场景**：
- `"bit0=GPIO_WRITE 能力, bit1=GPIO_READ 能力, ..."`
- `"D0=电源状态, D1~D2=运行模式, D3~D7=保留"`
- 状态字节中每个位有独立含义

**config 示例**：
```yaml
- name: capability_bitmap
  type: bitfield
  length: 4
  byte_order: big
  bits:
    - {name: "gpio_write",  start_bit: 0,  description: "GPIO 写能力"}
    - {name: "gpio_read",   start_bit: 1,  description: "GPIO 读能力"}
    - {name: "adc_once",    start_bit: 2,  description: "ADC 单次采样能力"}
    - {name: "adc_window",  start_bit: 3,  description: "ADC 窗口采样能力"}
    - {name: "eeprom_read", start_bit: 4,  description: "EEPROM 读能力"}
    # ... bit 5~31
```

**位序号约定**：`start_bit: 0` = 最低位（LSB），`start_bit: 7` = 最高位（MSB）。
协议文档中常见的 "bit0" 通常指 LSB；"D0" 也是 LSB。

### 6.2 enum — 枚举

**适用场景**：
- `"状态: 0=停止, 1=运行, 2=挂起"`
- `"run_mode: 1=连续, 2=脉冲计数, 3=时长"`
- `"result_code: 0x0000=成功, 0x0001=失败"`

**config 示例**：
```yaml
- name: state
  type: enum
  length: 1
  enum_values:
    "0": "STOPPED"
    "1": "RUNNING"
    "2": "HOLDING"
  description: "脉冲引擎运行状态"
```

**注意**：
- enum_values 的 key 是**字符串**（因为 yaml 解析后字段值是字符串）
- 如果协议文档中的枚举值以 hex 给出（0x0000），转换时保持原格式作为 key

### 6.3 record — 嵌套记录

**适用场景**：
- 一个命令的响应中包含嵌套的子结构体
- 例如 "每条事件记录包含 4 字节时间戳 + 1 字节事件类型 + 4 字节事件数据"

### 6.4 record_array — 记录数组

**适用场景**：
- "返回 N 条日冻结记录，每条 16 字节"
- "历史数据批量读取，每条记录结构相同"

---

## 7. 字节序决策流程图

```
协议文档描述
  │
  ├─ 明确声明 "大端" / "高字节在前" / "MSB first" / "Big-Endian"
  │   → 不写 byte_order（默认 big），或显式写 byte_order: big
  │
  ├─ 明确声明 "小端" / "低字节在前" / "LSB first" / "Little-Endian"
  │   → byte_order: little
  │
  ├─ 未明确声明，但有全局约定（如 "本协议统一大端"）
  │   → 按全局约定
  │
  └─ 完全未提及字节序
      → 检查示例 hex dump
          例如字段声明为 u16，值为 0x0001
          hex dump 中该字段显示为 00 01 → 大端
          hex dump 中该字段显示为 01 00 → 小端
```

**常见协议的字节序习惯**：
- 电力/燃气行业协议：通常大端（类似 DL/T 645）
- Modbus 系协议：大端
- 互联网协议（TCP/IP 风格）：大端
- x86 PC 端自定义协议：可能是小端
- 不确定时 → 从示例 hex dump 反推

---

## 8. scale 与 unit 的使用

当协议字段的传输值与实际物理值存在缩放关系时：

```yaml
- name: avg_uv
  type: uint32
  length: 4
  scale: 1        # 不需要缩放
  unit: uV        # 仅用于 GUI 展示的单位后缀
  description: "平均电压"

- name: temperature
  type: int16
  length: 2
  scale: 0.1      # 传输值 * 0.1 = 实际值
  unit: "°C"
  description: "温度（传输值÷10）"
```

---

## 9. 快速决策矩阵

你应该对照以下表格，快速判断每个协议字段的 FieldType：

| 判断条件 | FieldType | 确认项 |
|---------|-----------|--------|
| 就是 0x00~0xFF 一个字节的值 | `uint8` | - |
| 2 或 4 字节整数，文档说 "无符号" | `uint16` / `uint32` | 字节序 |
| 2 或 4 字节整数，文档说 "有符号" | `int16` / `int32` | 字节序 |
| 固定字节数的原始字节，不关心内部结构 | `hex` | 长度 |
| 数据段的长度由另一个字段指定 | `raw_hex` + `dynamic: true` | 确认哪个字段是长度 |
| 以 0x00 结尾或补零的文本 | `string` | 字符编码 |
| 十进制数字用十六进制表示（如 99=0x99） | `bcd_u` | 确认是 BCD 不是 hex |
| BCD 值带正负号和小数点（如金额） | `bcd` | 确认符号字节位置和整数/小数位数 |
| 年/月/日/时/分/秒 编码 | 选对应日期类型 | 确认格式；区分 BCD/HEX 编码 |
| 单字节表示 8 个独立状态 | `bitfield` | 列出每个 bit |
| 有限个预定义取值 | `enum` | 列出所有取值 |
| 字段内还有子字段结构，且结构固定 | `record` | 子结构定义 |
| 结构体重复多次 | `record_array` | 重复次数来源 |
