# 从协议文档到 commands.yaml 的精确模板

配合 `protocol-integration-workflow` 和 `document-analysis-workflow.md` 使用。
本文档覆盖从协议文档中的命令表到可用的 `commands.yaml` 的完整转换流程。

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
