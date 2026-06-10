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
