# 从协议文档到 commands.yaml 的精确模板

配合 `protocol-integration-workflow` 和 `document-analysis-workflow.md` 使用。
本文档覆盖从协议文档中的命令表到可用的 `commands.yaml` 的完整转换流程。

**重要**：第 1-11 节面向**固定 schema** 协议（字段按固定顺序出现）。
如果你的协议使用 **TLV / Tag-Length-Value** 自描述格式，请从第 12 节开始阅读。

---

## 1. commands.yaml 文件结构总览

```yaml
# ===== 可选：指令级别属性声明 =====
instruction_attributes:
  <属性名>:
    values: [可选值列表]
    codec_override: true      # 标记为 codec_override 的属性会通过 PackContext.overrides 传递

# ===== 可选：字段属性声明 =====
field_attributes:
  common:
    - name
    - description
    - type
    - length
    - default
    - unit
    - byte_order
    - scale

# ===== 必须：命令定义 =====
commands:
  <命令名>:
    description: "<描述>"
    request:
      name: "<可选，请求的显示名>"
      description: "<可选>"
      command_id: 0x<hex>
      fields:
        - name: <中文字段名>
          type: <FieldType注册名>
          length: <字节数>
          # ... 其他属性
    response:
      command_id: 0x<hex>
      fields:
        - name: result_code
          type: uint16
          length: 2
        # ... 更多字段
```

---

## 2. 命令名命名约定

### 2.1 命名原则

- **用中文**：与 GUI 展示一致，方便测试同事使用
- **动词开头或名词明确**：`读取板卡信息`、`设置GPIO`、`修改累积量`
- **与协议文档的命令名保持一致**：如果文档叫 `GET_INFO`，翻译为 `读取板卡信息`

### 2.2 常见模式

| 文档命名 | 推荐中文命名 |
|---------|------------|
| GET_XXX / READ_XXX | 读取XXX |
| SET_XXX / WRITE_XXX | 设置XXX / 写入XXX |
| XXX_CTRL / XXX_CONFIG | XXX控制 / XXX配置 |
| QUERY_XXX / REQ_XXX | 查询XXX |
| 主动上报 / EVT_XXX | XXX事件（响应） |
| HEARTBEAT / PING | 心跳 / 探活 |

---

## 3. command_id 格式

**必须使用 `0x` 前缀的 hex 字符串**：

```yaml
command_id: 0x0001    # 正确
command_id: 0x1A69    # 正确（大写 hex 字母）
command_id: 0001      # 错误 — 会被解析为十进制
command_id: 0x0001H   # 错误 — 不要带 H 后缀
```

框架在加载时用 `int(cmd_id, 16)` 转换。确保协议文档中的命令码（无论是 `0001H`、`0x0001`、`01H`）统一转为此格式。

### 从协议文档格式转换

| 文档写法 | commands.yaml 写法 |
|---------|-------------------|
| `0001H` | `0x0001` |
| `CMD=01` | `0x0001`（补齐到实际位数） |
| `命令码: 1` | `0x0001` |
| `0x1A69` | `0x1A69`（保持原样） |

---

## 4. request 与 response 的分离规则

### 4.1 单向命令（仅下行）

有些命令没有响应（如广播命令），只定义 `request`：

```yaml
commands:
  广播校时:
    request:
      command_id: 0x0008
      fields:
        - name: 当前时间
          type: yymmddhhmmss
          length: 6
```

### 4.2 单向命令（仅上行）

有些协议有主动上报/事件帧，只定义 `response`：

```yaml
commands:
  GPIO变化事件:
    description: 板卡主动上报 GPIO 输入电平变化
    response:
      command_id: 0x8001
      fields:
        - name: channel
          type: uint8
          length: 1
        - name: new_level
          type: uint8
          length: 1
```

### 4.3 双向命令（最常见）

大部分协议命令有 request 和 response：

```yaml
commands:
  设置GPIO:
    request:
      command_id: 0x0101
      fields:
        - name: 通道
          type: uint8
          length: 1
        - name: 电平
          type: uint8
          length: 1
    response:
      command_id: 0x0101
      fields:
        - name: result_code
          type: uint16
          length: 2
        - name: 通道
          type: uint8
          length: 1
        - name: 回读电平
          type: uint8
          length: 1
```

### 4.4 request/response 共用 command_id

大部分协议中 request 和 response 的 command_id 相同。如果不同（极少见），按协议文档填写。

---

## 5. 字段定义详细模板

### 5.1 字段属性完整清单

| 属性 | 必须 | 类型 | 说明 |
|------|------|------|------|
| `name` | ✓ | string | 中文字段名，用于 GUI 显示和脚本引用 |
| `type` | ✓ | string | FieldType 注册名（见 field-type-mapping.md） |
| `length` | ✓ | int/string | 字节数，`"1"` 或 `1` 均可 |
| `description` | - | string | 字段说明，用于 GUI tooltip |
| `default` | 推荐 | string | 未显式传值时的默认值 |
| `byte_order` | - | string | `"big"` 或 `"little"`，默认 `"big"` |
| `unit` | - | string | 物理单位，仅用于 GUI 展示 |
| `scale` | - | string/float | 缩放系数，传输值 × scale = 实际值 |
| `enum_values` | enum时 | dict | 枚举映射 `{"0": "停止", "1": "运行"}` |
| `bits` | bitfield时 | list | 位域定义 `[{name, bit, description}]` |
| `dynamic` | raw_hex时 | bool | `true` 表示变长 hex |
| `range` | - | string | 值范围限制，如 `"0..255"` |

### 5.2 典型字段定义示例

**简单整数**：
```yaml
- name: 通道
  type: uint8
  length: 1
  description: 硬件通道号，1 起始
```

**带字节序的多字节整数**：
```yaml
- name: 采样间隔us
  type: uint16
  length: 2
  byte_order: big
  description: 采样间隔，微秒
```

**带缩放的量值**：
```yaml
- name: 平均电压
  type: uint32
  length: 4
  unit: uV
  scale: 1
  description: 采样窗口内的平均电压（微伏）
```

**枚举**：
```yaml
- name: 运行模式
  type: enum
  length: 1
  enum_values:
    "1": "CONTINUOUS"
    "2": "PULSE_COUNT"
    "3": "DURATION_MS"
  description: "脉冲引擎运行模式"
```

**变长 hex 数据**：
```yaml
- name: 长度
  type: uint16
  length: 2
  description: 后续数据字节数
- name: 数据
  type: raw_hex
  length: 0
  dynamic: true
  description: 写入数据块（长度由"长度"字段指定）
```

**位域**：
```yaml
- name: 能力位图
  type: bitfield
  length: 4
  byte_order: big
  bits:
    - {name: "gpio_write",  bit: 0}
    - {name: "gpio_read",   bit: 1}
    - {name: "adc_once",    bit: 2}
    # ... bit3~31
  description: 硬件能力位图
```

**保留字段**：
```yaml
- name: reserved0
  type: uint8
  length: 1
  default: "0"
  description: 保留，发送端填 0
```

---

## 6. 字段顺序 = 字节顺序（关键规则）

`commands.yaml` 中 `fields` 数组的顺序**就是**序列化/反序列化的字节顺序。

**验证方法**：
1. 从协议文档中找到该命令的字段表
2. 按字节顺序（不是语义分组）排列
3. 用测试向量的 hex dump 验证：payload 字节流是否与字段顺序一致

**常见错误**：
```yaml
# 错误 — 把 result_code 放到了最后
fields:
  - name: 通道
    type: uint8
  - name: 电平
    type: uint8
  - name: result_code
    type: uint16

# 正确 — result_code 是响应首字段
fields:
  - name: result_code
    type: uint16
  - name: 通道
    type: uint8
  - name: 电平
    type: uint8
```

---

## 7. 交叉引用字段的处理

### 7.1 通用响应字段

如果协议文档规定所有响应都以 `result_code` 开头，**每条命令的 response 必须显式列出 `result_code`**，不能省略。框架不会自动插入。

### 7.2 字段名复用

不同命令可以有同名字段（如多个命令都有 `通道` 字段），每个命令独立定义。

---

## 8. instruction_attributes 的使用

当协议中的某些命令有不同于默认行为的特殊参数时，使用 `instruction_attributes`。

```yaml
instruction_attributes:
  security_mode:
    values: [auto, plain_mac, cipher_mac, none]
    codec_override: true
  mid_policy_override:
    values: [auto_increment, echo_last_terminal_mid, manual]
    codec_override: true
  operation_mode:
    values: [normal, maintenance, calibration]
    codec_override: true
```

被标记 `codec_override: true` 的属性在 ProtocolChannel 发送时从 field_data 中剥离，传给 `PackContext.overrides`，不参与 BinaryFormatter 序列化。

---

## 9. 完整命令文件示例

以下是一个完整的 `commands.yaml` 示例，展示各种模式：

```yaml
field_attributes:
  common:
    - name
    - description
    - type
    - length
    - default
    - unit
    - byte_order
    - scale

commands:
  读取板卡信息:
    description: 读取固件版本、能力位图和构建时间
    request:
      description: 无 payload
      command_id: 0x0001
      fields: []
    response:
      description: 所有响应都以 result_code 开头
      command_id: 0x0001
      fields:
        - name: result_code
          type: uint16
          length: 2
          unit: code
        - name: 协议主版本
          type: uint8
          length: 1
        - name: 协议次版本
          type: uint8
          length: 1
        - name: 固件版本
          type: string
          length: 16
          description: 固件版本字符串，不足补 0x00
        - name: 板卡标识
          type: hex
          length: 16
        - name: 能力位图
          type: bitfield
          length: 4
          byte_order: big
          bits:
            - {name: "gpio_write",  bit: 0}
            - {name: "gpio_read",   bit: 1}
            - {name: "adc_once",    bit: 2}
        - name: 构建时间
          type: uint32
          length: 4
          unit: unix_time_s

  设置GPIO:
    request:
      command_id: 0x0101
      fields:
        - name: 通道
          type: uint8
          length: 1
          description: 硬件通道号，1 起始
        - name: 电平
          type: enum
          length: 1
          enum_values:
            "0": "低"
            "1": "高"
    response:
      command_id: 0x0101
      fields:
        - name: result_code
          type: uint16
          length: 2
        - name: 通道
          type: uint8
          length: 1
        - name: 回读电平
          type: uint8
          length: 1

  读取EEPROM:
    request:
      command_id: 0x0301
      fields:
        - name: bus_id
          type: uint8
          length: 1
        - name: 从机地址
          type: uint8
          length: 1
          description: EEPROM 7-bit 地址
        - name: 地址
          type: uint32
          length: 4
        - name: 长度
          type: uint16
          length: 2
    response:
      command_id: 0x0301
      fields:
        - name: result_code
          type: uint16
          length: 2
        - name: 地址
          type: uint32
          length: 4
        - name: 长度
          type: uint16
          length: 2
        - name: 数据
          type: raw_hex
          length: 0
          dynamic: true
```

---

## 10. 从协议文档表格到 commands.yaml 的转换脚本模板

当你面对一个含几十条命令的 PDF 时，半自动化的转换流程：

```python
# 步骤 1: 手工建立命令索引表（csv）
# 从 PDF 表格中逐行提取：
#   command_id_hex, command_name_cn, direction, field_list_raw
# 保存为 commands_index.csv

# 步骤 2: 对每个命令，手工填写字段表
#   field_name_cn, field_type_doc, field_length, field_default, field_desc
# 保存为 fields_<command_name>.csv

# 步骤 3: 运行转换脚本生成 commands.yaml
import csv, yaml

def build_commands_yaml(index_csv, fields_dir):
    commands = {}
    for row in csv.DictReader(open(index_csv)):
        cmd_name = row["command_name_cn"]
        cmd_id = row["command_id_hex"]  # 如 "0x0001"
        direction = row["direction"]     # "REQ" / "RSP" / "BOTH"

        entry = {}
        if direction in ("REQ", "BOTH"):
            entry["request"] = {
                "command_id": cmd_id,
                "fields": load_fields(f"{fields_dir}/fields_{cmd_name}_req.csv"),
            }
        if direction in ("RSP", "BOTH"):
            entry["response"] = {
                "command_id": cmd_id,
                "fields": load_fields(f"{fields_dir}/fields_{cmd_name}_rsp.csv"),
            }
        commands[cmd_name] = entry
    return {"commands": commands}

# 步骤 4: 人工审查生成的 YAML
# 逐命令检查字段顺序是否等于字节顺序
# 检查 enum 的 enum_values 是否完整
# 检查 bitfield 的 bits 定义
```

---

## 11. 验证检查清单

生成 `commands.yaml` 后，逐条核对：

- [ ] 每条命令的 `command_id` 格式正确（`0x` 前缀 hex）
- [ ] request/response 分离正确（单向命令只有一侧）
- [ ] 所有 RSP 都包含 `result_code` 字段（如果协议有此约定）
- [ ] 字段顺序 = 字节顺序（不是语义分组）
- [ ] 保留字段已显式列出，`default` 已填写
- [ ] 变长数据字段已标记 `dynamic: true`
- [ ] 枚举字段的 `enum_values` 完整
- [ ] 位域字段的 `bits` 逐位定义
- [ ] 字节序与协议文档一致
- [ ] 所有字段的 `length` 与实际字节数一致
- [ ] 字段 type 在 TypeRegistry 中已注册（或计划注册新类型）

---

## 12. TLV / 自描述载荷的处理策略

第 6 节"字段顺序 = 字节顺序"适用于**固定 schema** 协议——每个字段按固定偏移出现在帧中。
当协议使用 **TLV（Tag-Length-Value）**、**key-value**、**Tagged Union** 等自描述格式时，
字段不再靠"第几个字节"定位，而是靠 **Tag/ID 标识**。这时候需要不同的策略。

### 12.1 自描述格式的识别信号

在协议文档中发现以下特征时，说明载荷是自描述格式，不能简单用 `fields` 数组按序定义：

| 信号 | 示例 |
|------|------|
| 数据域有 Tag + Length 头 | `Tag(1) + Length(2) + Data(n)` |
| 字段用 ID 号标识，不是按偏移描述 | `ID=0 表示 IMEI, ID=1 表示 IMSI` |
| 文档写"可选字段"、"按需上报" | `"报警状态每日必须上传一次"` 、 `"密集数据根据需要上传"` |
| 同一个命令的响应，不同表型返回不同字段集 | `"仅适用于超声波表或流量计"` |
| 字段可以乱序出现 | 文档未规定"字段出现的先后顺序" |
| 存在嵌套子结构且子记录数可变 | `BYTE[4*N]` — N 由另一个字段决定 |

**注意**：仅 `data` 段变长（长度字段 + raw_hex）不算自描述格式——那仍属于固定 schema，
按第 5.2 节"变长 hex 数据"处理即可。自描述格式的核心特征是**字段存在性和顺序不固定**。

### 12.2 `decoded_fields_mode` 决策树

```
协议 payload 结构
  │
  ├─ 每个字段都在固定位置，总出现，顺序固定
  │   → mode: "schema"（默认）
  │   → commands.yaml: 完整 fields 数组，BinaryFormatter 全权序列化/反序列化
  │   → 字段定义粒度: 每个字节都要在 fields 中出现
  │
  ├─ 部分字段固定 + 部分字段可选/变长（如 TLV）
  │   → mode: "merge"
  │   → commands.yaml: 定义所有**可能**出现的字段（含固定字段 + 可选字段的元数据）
  │   → codec: 在 decoded_fields 中填入**实际出现**的字段值
  │   → 框架: schema 字段打底，decoded_fields 的同名键覆盖值
  │
  └─ 字段结构完全不由 commands.yaml 描述（加密 blob、复杂嵌套、自定义编码）
      → mode: "replace"
      → commands.yaml: 最小定义（命令名 + command_id），fields 可为空或仅含占位 raw_hex
      → codec: 全权负责解析，decoded_fields 完整构建展示用字段
      → 框架: 完全使用 decoded_fields，忽略 schema
```

**关键判断**：如果你的 codec 需要写 `_build_decoded_fields()` 来解析 TLV，那你就处于 `merge` 或 `replace` 分支，
不再是 `schema` 分支。`schema` 分支下 codec 不需要构建 `decoded_fields`。

### 12.3 `merge` 模式下的字段定义粒度

`merge` 是最常用的自描述格式模式。要点：

**REQ（下行/请求）字段**：
- 用户需要通过 GUI 填写的每个参数，都必须定义为独立字段
- 包括嵌套子结构的每一层——不要压缩成 `raw_hex`
- 示例（查询气表历史数据的 REQ）：
  ```yaml
  # 好 — 用户能逐个填写
  request:
    command_id: "0x0004"
    fields:
      - name: 查询起始时间
        type: yymmdd
        length: 3
        description: BCD码，精确到天，格式YYMMDD
      - name: 查询模式
        type: enum
        length: 1
        enum_values: {"0": "按天查询", "1": "按月查询"}
      - name: 查询个数
        type: uint8
        length: 1
        description: 按天最大31天，按月最大60月
  ```

**RSP（上行/响应）字段**：
- 定义所有**可能**出现的字段的全量元数据（即使某次响应中不一定全有）
- 每个可选字段的 `description` 中标注其 TLV Tag/ID 来源
- 固定字段（如 result_code）按正常位置列出
- 示例（数据上报 RSP 的部分字段）：
  ```yaml
  # 好 — 列出 TLV 中可能出现的字段，codec 的 decoded_fields 会填入实际值
  response:
    command_id: "0x0081"
    fields:
      - name: IMEI
        type: bcd_u
        length: 8
        description: Tag 0x01 ID=0, 8字节BCD, 最高位补0
      - name: IMSI
        type: bcd_u
        length: 8
        description: Tag 0x01 ID=1
      - name: RSRP
        type: uint16
        length: 2
        description: Tag 0x02 ID=0, 信号强度, 取绝对值
      - name: 主电电压
        type: uint16
        length: 2
        unit: "0.01V"
        scale: 0.01
        description: Tag 0x02 ID=6, 扩大100倍上传
      # ... 逐一列出 Tag 0x01~0x05 的所有子记录
  ```

**原则**：
- 如果字段值来自 `decoded_fields`，其类型定义（type/length/unit/scale）主要服务于 GUI 展示格式，
  不影响反序列化逻辑（反序列化由 codec 完成）
- `description` 必须标注 TLV 来源（Tag + ID），方便后续维护时对照协议文档

### 12.4 `replace` 模式下的字段定义粒度

当 codec 全权负责解析时（如 100 协议的记录数据、加密后的嵌套结构）：

```yaml
# replace 模式 — commands.yaml 只需最小定义
response:
  command_id: "0x0081"
  fields:
    - name: 数据摘要
      type: string
      length: 0
      dynamic: true
      description: 由 codec 解析 TLV 后提供的结构化数据摘要
```

**`replace` vs `merge` 的选择**：
- 当字段很多（>30种）、结构深层嵌套、或不同表型返回完全不同的字段集时 → 倾向 `replace`
- 当字段类型稳定、主要差异在"有无"而非"结构"时 → 倾向 `merge`（GUI 能利用 schema 的 unit/scale/enum_values 元数据）

### 12.5 codec ↔ commands.yaml 的契约

使用 `merge` 模式时，codec 返回的 `decoded_fields` 的 **key 名必须与 `fields[].name` 完全一致**：

```python
# codec.py — unpack() 中
decoded_fields = {
    "IMEI": "0866674051156284",      # 必须匹配 commands.yaml 中 fields[].name
    "IMSI": "0460113240741862",
    "RSRP": 1,
    "主电电压": 24.36,                # codec 已除以 100 的最终值
}
return UnpackResult(
    command_id=func_code,
    payload=tlv_data,
    decoded_fields=decoded_fields,
    decoded_fields_mode="merge",
)
```

```yaml
# commands.yaml — 对应的字段定义
- name: IMEI          # ← key 必须匹配
  type: bcd_u
  length: 8
- name: IMSI
  type: bcd_u
  length: 8
- name: RSRP
  type: uint16
  length: 2
- name: 主电电压        # ← key 必须匹配
  type: uint16
  length: 2
  unit: V
  scale: 0.01
```

**契约规则**：
1. `decoded_fields` 的 key 与 `fields[].name` 同名时 → 框架用 decoded_fields 的值覆盖 schema 解析值
2. `decoded_fields` 有但 schema 没有的 key → 框架仍会加入最终字段（但没有 type/length 元数据，GUI 显示为原始值）
3. schema 有但 `decoded_fields` 没有的 key → 框架尝试用 BinaryFormatter 从 payload 解析（通常失败/跳过）

**最佳实践**：codec 只返回实际出现的字段，不返回未出现的字段（值为 `None` 或空字符串）。
这样 GUI 不会被大量空值占满。

### 12.6 TLV 协议 commands.yaml 编写流程

```
Phase 0 产出: 协议文档分析 → 整理出所有 TLV Tag 及其子记录表
    │
    ▼
步骤 1: 确定每条命令 (func_code) 的数据域包含哪些 TLV Tag
    │  例: 0x01(数据上报) → Tag 0x01, 0x02, 0x03, 0x04, 0x05
    │       0x03(查询信息) → Tag 0x01, 0x06
    │
    ▼
步骤 2: 罗列每个 Tag 的所有子记录 ID、名称、类型、长度
    │  例: Tag 0x02 ID=6 → 主电电压, uint16, 2B, 0.01V
    │
    ▼
步骤 3: 为 REQ (下行) 定义用户需要填写的字段
    │  原则: 每个可填参数独立列为一个字段
    │
    ▼
步骤 4: 为 RSP (上行) 定义全量子记录字段
    │  原则: 所有可能出现的子记录都列出（即使不总出现）
    │  每个字段的 description 标注 Tag/ID 来源
    │
    ▼
步骤 5: 确认 codec 的 decoded_fields key 名与 fields[].name 一致
    │
    ▼
步骤 6: 走验证检查清单（见下方 12.7）
```

### 12.7 TLV 模式验证检查清单

在标准检查清单（第 11 节）之外，TLV 模式额外核对：

- [ ] 已判断协议属于固定 schema / `merge` / `replace` 哪种模式
- [ ] 数据域的每个 TLV Tag 都有对应的字段定义（在 description 中标注了 Tag/ID）
- [ ] REQ 字段粒度足够用户逐项填写（没有把多个参数压缩成一个 raw_hex）
- [ ] RSP 字段名称与 codec `decoded_fields` 的 key 名一致
- [ ] codec 只返回实际出现的字段，不返回空值占位
- [ ] 可选字段的 `description` 中注明了适用条件（如"仅超声波表"）
- [ ] `decoded_fields_mode` 在 codec 返回时显式指定为 `"merge"` 或 `"replace"`
- [ ] 如果字段值由 codec 计算（如除以 scale 后的最终值），已确认 codec 的值与 schema 的 scale 不重复应用

---

## 13. 表具行业敏感数据强制解析规则

**适用范围**：燃气表、水表、热量表、电表等表具行业协议。

表具协议的核心目的是**计量与计费**。上位机软件必须对计量数据（累积量/金额/流量/压力/温度等）
做精确解析、展示、记录和校验。**绝不允许**将这些数据压缩为 `raw_hex` 模糊透传——
这是表具协议接入的"一票否决"级违规。

### 13.1 核心原则：计量数据是"一等公民"

> **红线**：气量、水量、金额、流量、压力、温度、单价、结算数据——这些是表具协议存在的根本理由。
> 每一条计量数据都必须作为独立字段出现在 `commands.yaml` 中，具有明确的 type、length、unit、scale。

**为什么不能 raw_hex**：
- 上位机需要展示计量数值给用户（不是 hex dump）
- 上位机需要记录计量数据到数据库（需要数值类型，不是 hex 字符串）
- 上位机需要做数据校验（如累积量单调递增检查）
- 上位机需要做金额结算（需要精确数值计算）
- 上位机需要做报警联动（如余额不足关阀、超流量报警）

**Skill 层面的强制要求**：
当 AI 助理接入表具协议时，遇到计量相关数据**必须**追问自己：
"这个字段上位机是否需要展示/记录/校验/计算？" → 如果是 → 必须独立定义。

### 13.2 敏感数据分类与强制定义要求

以下数据类别在表具协议中属于**强制独立定义**范畴：

| 数据类别 | 典型字段名 | 强制属性 | 说明 |
|---------|-----------|---------|------|
| **累积量** | 累计用气量、正向累积量、反向累积量、累计用水量、日结累积量 | type + length + unit + scale | 表具核心数据，必须精确解析 |
| **金额** | 剩余金额、累计金额、本次扣费、透支金额 | type + length + unit + scale | 涉及计费结算，数值精度不可丢失 |
| **流量** | 瞬时流量、工况流量、标况流量、平均流量 | type + length + unit + scale | 涉及工况判断和报警联动 |
| **压力** | 管道压力、大气压力、阀前压力、阀后压力 | type + length + unit + scale | 涉及安全报警 |
| **温度** | 介质温度、环境温度、表内温度 | type + length + unit + scale | 温补修正和高温报警 |
| **单价** | 气价、水价、阶梯单价、结算单价 | type + length + unit + scale | 涉及计费 |
| **结算数据** | 结算日、结算周期、阶梯气量阈值、预结算限值 | type + length + unit | 涉及计费规则 |
| **阀门/报警** | 阀门状态、余额不足报警、磁干扰报警、超流量报警 | bitfield 或 enum | 涉及安全联动 |
| **事件记录** | 事件类型、发生时间、结束时间、事件关联数据（如事件时刻累积量/金额/流量） | type + length, 事件类型用 enum, 时间用 BCD datetime | 涉及安全审计、故障追溯、计费纠纷裁决 |
| **设备标识** | IMEI、IMSI、表号、户号 | type + length | 涉及设备管理和密钥派生 |

### 13.3 判定方法：关键词触发

在分析协议文档时，遇到以下字段名或描述，**无条件**定义为独立字段：

**中文触发词（任一命中即触发）**：
```
气量 水量 热量 电量 金额 余额 流量 压力 温度 声速
单价 气价 水价 电价 结算 累积 累计 用量 消费 扣费
充值 阶梯 阀门 报警 关阀 开阀 透支 欠费 预留量
事件 告警 故障 恢复 泄漏 攻击 断电 上电 掉电 拆表
```

**英文触发词（任一命中即触发）**：
```
volume amount balance flow rate pressure temperature
price tariff accumulated consumption valve alarm
overdraft arrears
```

**判定流程**：
```
字段名/描述中是否包含上述触发词？
  ├─ 是 → 【强制】独立定义字段
  │      ├─ 固定偏移 → 正常 fields 定义（完整 type/length/unit/scale）
  │      ├─ TLV 自描述 → merge 模式下在 fields 中列出 + description 标注 Tag/ID
  │      └─ 变长数组（BYTE[X*N]）→ 见 §13.5
  └─ 否 → 按普通字段规则处理，可酌情使用 raw_hex
```

### 13.4 正确 vs 错误示例

**错误 1** — 将实时计量数据压缩为 raw_hex：
```yaml
# 错！累计用气量、剩余金额等核心数据被模糊化为 hex
response:
  fields:
    - name: 实时数据hex
      type: raw_hex
      length: 0
      dynamic: true
      description: 由 codec 解析 TLV 实时数据
```
**为什么错**：上位机无法从 commands.yaml 得知"累计用气量"的 type/unit/scale，
即使 codec 在 decoded_fields 中提供了值，GUI 也没有元数据来格式化展示。

**正确 1** — 逐字段定义：
```yaml
response:
  fields:
    - name: 当前正向累计气量
      type: uint32
      length: 4
      unit: m³
      scale: 0.01
      description: Tag 0x02 ID=4, 上传时间点冻结数据, 扩大100倍. 仅膜式燃气表
    - name: 剩余金额
      type: uint32
      length: 4
      unit: 元
      scale: 0.01
      description: Tag 0x02 ID=11, 扩大100倍. 仅预结算功能仪表
    - name: 当前温度
      type: int16
      length: 2
      unit: "0.1℃"
      scale: 0.1
      description: Tag 0x02 ID=19, 有符号, 扩大10倍. 仅超声波表/流量计
```

**错误 2** — 以"字段太多"为由批量压缩：
```yaml
# 错！"Tag 0x02 有 23 条子记录太多了"不是压缩成 raw_hex 的理由
- name: 实时数据_blob
  type: raw_hex
  length: 0
  dynamic: true
```
**为什么错**：23 条子记录中至少 10 条是敏感计量数据（累积量×4、金额×1、流量×2、温度×1、压力×1、预留量×1），
每条都必须独立定义。

**正确 2** — 全量列出，用 description 区分适用范围：
```yaml
- name: 当前正向累计气量
  type: uint32
  length: 4
  unit: m³
  description: Tag 0x02 ID=4. 仅膜式燃气表
- name: 当前工况累积量
  type: uint32
  length: 4
  unit: m³
  description: Tag 0x02 ID=14. 仅超声波表/流量计
# ... 全部 23 条逐一列出
```

**错误 3** — 历史/周期数据中的累计用量被压缩：
```yaml
# 错！历史的累计用气量是计量数据，不是"通用hex"
- name: 历史_累计用气量
  type: raw_hex
  length: 0
  dynamic: true
  description: Tag 0x09 ID=2, BYTE[4*N], HEX格式
```
**为什么错**：历史的累计用气量仍然是累计用气量——上位机需要展示和比对历史用量趋势。

**错误 4** — 事件记录被压缩为 raw_hex：
```yaml
# 错！事件记录是安全审计的核心数据，不能模糊化为 hex blob
- name: 事件记录数据
  type: raw_hex
  length: 0
  dynamic: true
  description: N条事件记录，每条=事件类型(1)+发生时间(6)+结束时间(6)
```
**为什么错**：上位机需要按事件类型筛选、按时间段查询、统计某类事件的频次——这些都是靠字段级别的定义才能做到的。
如果把事件记录压缩为 hex，上位机就失去了对事件的检索和分析能力。

**正确 4** — 事件记录逐字段定义，数组用 record_array：
```yaml
- name: 事件记录条数
  type: uint8
  length: 1
  description: 后续事件记录的条数N
- name: 事件记录列表
  type: record_array
  length: 13           # 每条事件记录 13 字节
  description: N条事件记录，每条结构见 element
  element:
    - name: 事件类型
      type: enum
      length: 1
      enum_values:
        "0": "开阀"
        "1": "关阀"
        "2": "余额不足"
        "3": "强磁攻击"
        "4": "燃气泄漏"
        "5": "超流量"
        "6": "断电"
        "7": "上电"
        "8": "外壳被拆"
        "9": "温度过高"
      description: 事件类型编码
    - name: 事件发生时间
      type: yymmddhhmmss
      length: 6
      description: BCD码，事件触发时刻
    - name: 事件结束时间
      type: yymmddhhmmss
      length: 6
      description: BCD码，事件恢复时刻（未恢复时填全F）
```

### 13.5 变长数组型敏感数据的处理

表具协议中常见的 `BYTE[4*N]`、`BYTE[2*N]` 型数组数据（如 N 天的历史累计用量），
虽然是变长的，但**仍不能**用 `raw_hex` 模糊化。处理策略分两层：

**策略 A（优先）— 使用 `record_array` 类型**：

如果框架支持 `record_array` 类型，将数组元素定义为嵌套记录：

```yaml
- name: 历史累计用气量
  type: record_array
  length: 4          # 每条记录的字节数
  unit: m³
  scale: 0.01
  description: Tag 0x09 ID=2, 数组长度由"历史_数据个数"字段指定
  element:
    name: 日累计用气量
    type: uint32
    length: 4
```

**策略 B（次选）— 明确标注元素结构**：

如果框架暂不支持 `record_array` 解析，在 description 中**明确标注**元素的结构，
确保后续可以升级为结构化定义：
```yaml
- name: 历史累计用气量
  type: raw_hex
  length: 0
  dynamic: true
  unit: m³
  scale: 0.01
  description: >
    Tag 0x09 ID=2. ⚠️ 待升级为 record_array.
    元素结构: uint32 × N, 每4字节一个累计用气量值.
    分辨率 0.01m³. 数组长度由"历史_数据个数"字段指定.
```

**关键区别**：策略 B 与"偷懒用 raw_hex"的区别在于——
- 策略 B 有 `unit`、`scale` 元数据
- 策略 B 在 description 中标注了 ⚠️ 待升级标记
- 策略 B 明确了元素结构，而不是模糊的 "hex数据"

### 13.6 例外：可以使用 raw_hex 的场景

以下数据**可以**使用 `raw_hex`，但必须满足"非计量数据"的前提：

| 数据类型 | 可以的理由 | 示例 |
|---------|-----------|------|
| 透传数据（与主站通信的原始报文） | 上位机不解析其内部结构 | Tag 0x0A 透传数据 |
| 加密后的完整载荷（在解密前） | 解密前无业务含义 | TLV 密文整体 |
| 固件升级包 | 纯二进制，非计量数据 | OTA 升级数据块 |
| 厂商自定义扩展区且无文档 | 无法得知内部结构 | 未文档化的私有扩展 |
| 纯辅助数据（非计量） | 不影响计量与计费 | 调试日志、心跳填充 |

**不能作为 raw_hex 借口的理由**：
- "TLV 字段太多了" → 不是理由，必须逐个列出
- "不同表型返回不同字段" → merge 模式天然支持可选字段
- "数组长度不固定" → 见 §13.5 策略 B
- "协议文档没写清楚" → 标注 ⚠️ 待确认，不能直接用 raw_hex 代替

### 13.7 表具协议 commands.yaml 审查检查清单

在 §11（通用）和 §12.7（TLV）的基础上，表具协议额外核对：

- [ ] 所有累积量字段（累计用气量/用水量/热量等）已独立定义，含 type/length/unit/scale
- [ ] 所有金额字段（剩余金额/累计金额/单价等）已独立定义，含 type/length/unit/scale
- [ ] 所有流量字段（瞬时流量/工况流量/标况流量）已独立定义，含 type/length/unit/scale
- [ ] 所有压力字段已独立定义，含 type/length/unit/scale
- [ ] 所有温度字段已独立定义，含 type/length/unit/scale
- [ ] 阀门状态/报警状态已用 enum 或 bitfield 定义
- [ ] 事件记录（事件类型/发生时间/结束时间/关联数据）已逐字段定义，事件类型使用 enum
- [ ] 事件记录的变长数组已使用 record_array 或标注元素结构
- [ ] 设备标识（IMEI/IMSI/表号）已用 bcd_u 或 hex 定义
- [ ] 变长数组型计量数据已标注元素结构（至少有 unit/scale 和元素类型说明）
- [ ] 没有以 "TLV 字段太多"、"数组不固定"、"不同表型不同" 为由压缩计量数据
- [ ] 透传数据（raw_hex）中不包含独立的计量数据字段
