---
name: protocol-integration-workflow
description: First-principles guide for integrating a new communication protocol into ToolsKing. Covers the layered architecture (Transport/Codec/Channel/Runtime/Facade), frame encoding/decoding pipelines, security layer patterns, data type boundaries, parameter persistence, and UI contract. Use when analyzing, designing, or implementing any protocol plugin.
---

# ToolsKing 协议接入：第一性原理

> **完整工作流**：本文件是协议接入的架构与实现参考。如果你刚拿到一份新的 PDF/Word/MD 协议文档，请**先按以下顺序阅读参考文档**：
> 1. `references/document-analysis-workflow.md` — Phase 0: 从文档中系统提取所有实现所需信息
> 2. `references/field-type-mapping.md` — 把文档中的类型描述映射到框架的 FieldType
> 3. `references/commands-yaml-template.md` — 生成 commands.yaml
> 4. 回到本文档 — 理解架构分层，实现 codec
> 5. `references/test-vector-extraction.md` — 用文档中的 hex dump 编写契约测试
> 6. `references/technical-extension.md` — 处理同构拷贝、异常路径等特殊情况

## 目录

1. [分层架构：数据如何流过每一层](#1-分层架构数据如何流过每一层)
2. [帧编解码：codec 的精确契约](#2-帧编解码codec-的精确契约)
3. [安全层：加解密管线的嵌入方式](#3-安全层加解密管线的嵌入方式)
4. [数据类型系统：边界内与边界外](#4-数据类型系统边界内与边界外)
5. [参数模型：持久化与会话的合并规则](#5-参数模型持久化与会话的合并规则)
6. [UI 契约：界面能做什么，能向下传递什么](#6-ui-契约界面能做什么能向下传递什么)
7. [接入决策树：同构 vs 异构](#7-接入决策树同构-vs-异构)
8. [实现检查清单](#8-实现检查清单)

---

## 1. 分层架构：数据如何流过每一层

工程协议栈从物理到界面共 6 层，每层职责单一且不可跨越：

```
设备字节流
    │
    ▼
┌──────────────────────────────────────────────┐
│ Transport (app/core/transport/)               │  ← 物理 I/O
│  职责：字节收发。只管 open/close/write/回调  │
│  不管：协议格式、字段含义、会话状态           │
│  实现：SerialTransport / TcpTransport / Mock   │
└──────────────┬───────────────────────────────┘
               │ raw bytes ↑↓ (set_data_callback / write)
               ▼
┌──────────────────────────────────────────────┐
│ ProtocolCodec (plugins/<key>/codec.py)        │  ← 帧编解码
│  职责：raw bytes ↔ 帧（pack/unpack）          │
│  输入 PackContext → 输出 bytes               │
│  输入 bytes → 输出 UnpackResult              │
│  不管：字段序列化、命令查找、UI               │
└──────────────┬───────────────────────────────┘
               │ frame bytes ↑↓
               ▼
┌──────────────────────────────────────────────┐
│ ProtocolChannel (app/core/protocol/channel.py)│  ← 管道编排
│  职责：把 Codec + Transport + Session +       │
│        CommandStore 串成一条流水线            │
│  send(): 字段 → BinaryFormatter → codec.pack  │
│         → transport.write                     │
│  receive: transport回调 → codec.split_frames  │
│         → codec.unpack → 字段反序列化         │
│         → session.apply_updates → 分发回调     │
└──────────────┬───────────────────────────────┘
               │ ParsedFrame / SentFrame
               ▼
┌──────────────────────────────────────────────┐
│ ProtocolRuntime (app/core/protocol/runtime.py)│  ← 消费端
│  职责：实例隔离 + wait_for/request 事务        │
│  每个 instance_id 一个 Runtime               │
│  持有自己的 Channel + Transport + Session     │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│ ProtocolFacade (app/core/protocol/)           │  ← Qt 桥接（唯一Qt依赖层）
│  职责：发射 Qt Signal、协议切换、参数持久化    │
│  data_parsed / data_sent / error_occurred      │
│  之下全为纯 Python，无 Qt 依赖                │
└──────────────┬───────────────────────────────┘
               │ Qt signals
               ▼
┌──────────────────────────────────────────────┐
│ GUI (app/gui/)                                │  ← 界面
│  ProtocolUI / ConnectionPanel / 参数面板       │
└──────────────────────────────────────────────┘
```

**关键原则**：
- Transport 不感知协议，Codec 不感知 UI，Channel 不感知 Qt
- 新协议接入只需动 `plugins/` 和 `config/protocols/`，不动框架层
- 唯一的例外：协议需要新的 `codec_override` 属性时，可能需要微调 Facade 的 `send_command()`

---

## 2. 帧编解码：codec 的精确契约

### 2.1 发送管线（下行）

```
GUI 字段值
  │
  ▼
ProtocolChannel.send(command_name, field_data)
  │
  ├─ 1. CommandStore 查找命令定义 → DirectionDef
  ├─ 2. BinaryFormatter.format_fields(field_data, fields) → payload bytes
  │     （逐字段按 FieldType.to_bytes() 序列化，顺序=命令定义顺序）
  ├─ 3. 构造 PackContext(command_id, payload, is_uplink, session, overrides)
  ├─ 4. codec.pack(ctx) → 完整帧 bytes
  └─ 5. transport.write(frame_bytes)
```

**PackContext 的 5 个字段是 codec 能拿到的全部输入**：

| 字段 | 类型 | 来源 | 含义 |
|------|------|------|------|
| `command_id` | `int` | commands.yaml 的 command_id | 命令标识，框架已从 hex 转为 int |
| `payload` | `bytes` | BinaryFormatter 序列化结果 | 业务字段的字节表示，codec 只需包装 |
| `is_uplink` | `bool` | Channel 根据 direction 计算 | True=上行(设备→主机), False=下行 |
| `session` | `dict` | ProtocolSession.snapshot() | params + state 合并快照（只读） |
| `overrides` | `dict` | GUI 高级选项面板 | 单次发送的临时覆盖值 |

**codec.pack() 的唯一职责**：把 payload 包装成完整帧。包括但不限于——
- 加帧头/帧尾
- 嵌入 command_id 到帧的对应字段
- 从 session 中取序列号/地址/密钥
- 计算并追加 CRC/MAC
- 加密 payload
- 处理转义

**返回值**：完整帧字节。失败必定抛 `ProtocolPackError`，不返回 None 或 b""。

### 2.2 接收管线（上行）

```
transport 回调 data(bytes)
  │
  ▼
ProtocolChannel._on_raw_data(data)
  │
  ├─ 1. codec.split_frames(data) → [frame1, frame2, ...]
  │     （默认 [data]，串口粘包协议可覆盖）
  │
  └─ 对每一帧:
       ├─ 2. codec.unpack(frame, session_snapshot) → UnpackResult
       ├─ 3. session.apply_updates(result.session_updates)
       ├─ 4. CommandStore 查 command_id → command_name + direction
       ├─ 5. 按 decoded_fields_mode 构建最终字段:
       │     schema: 用 BinaryFormatter 反向解析 payload
       │     replace: 直接用 codec 返回的 decoded_fields
       │     merge: schema 打底，decoded_fields 覆盖
       ├─ 6. 构造 ParsedFrame → 入 _rx_queue
       └─ 7. 通知 _parsed_callbacks（GUI 通过此路径拿到解析结果）
```

**UnpackResult 的 6 个字段是 codec 能输出的全部**：

| 字段 | 类型 | 含义 |
|------|------|------|
| `command_id` | `int` | 从帧中提取的命令 ID（整数） |
| `payload` | `bytes` | 解密/解包后的业务字节 |
| `session_updates` | `dict` | 需要写回 session.state 的键值 |
| `decoded_fields` | `dict\|None` | codec 自行解析的结构化结果 |
| `decoded_fields_mode` | `str` | "schema" / "replace" / "merge" |
| `meta` | `dict` | 仅调试/日志用，不参与业务 |

**decoded_fields_mode 的三种模式**：

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `schema` | 忽略 decoded_fields，仅用 commands.yaml schema 反向解析 | 标准命令，字段结构与定义一致 |
| `replace` | 用 decoded_fields 完全替代 schema 解析 | 协议层自行完成所有解析（如 100 协议记录数据） |
| `merge` | schema 解析打底，decoded_fields 覆盖同名键 | 部分字段需要 codec 特殊处理 |

### 2.3 split_frames：多帧拆分

默认 `split_frames(data) → [data]`，即假设一次回调=一帧。

当 transport 是串口/TCP 字节流，一次 idle-timeout 回调可能包含多帧时，覆盖此方法：

```python
def split_frames(self, data: bytes) -> list[bytes]:
    """按帧头+长度域拆出零或多个完整帧。"""
    frames = []
    buffer = bytearray(data)
    while buffer:
        frame = self._extract_one_frame(buffer)
        if frame is None:
            if not frames:
                return [data]  # 不完整单帧：保持旧行为让 unpack 报精确错
            raise ValueError(f"尾帧不完整: {len(buffer)} bytes")
        frames.append(frame)
    return frames
```

参见：`plugins/hil_protocol/codec.py` 的 `split_frames` 实现。

---

## 3. 安全层：加解密管线的嵌入方式

### 3.1 安全层的正确位置

**安全层必须完全在 codec 内部处理**，框架层和 commands.yaml 不感知加解密。

```
pack 流程:
  payload (明文) → 加密 → 加 MAC → 嵌入帧 → 加 CRC → 帧头帧尾
                   ↑_____ codec.pack() 内部 _________↑

unpack 流程:
  raw bytes → 去帧头帧尾 → 验 CRC → 验 MAC → 解密 → payload (明文)
              ↑_______________ codec.unpack() 内部 _______________↑
```

### 3.2 密钥来源

密钥从 `ctx.session` 获取，session 是 params + state 的合并快照：

```python
def pack(self, ctx: PackContext) -> bytes:
    aes_key_hex = ctx.session.get("aes_key", "")
    aes_key = bytes.fromhex(aes_key_hex) if aes_key_hex else None
```

密钥可以来自：
- **持久化参数**：用户在参数面板配置，存在 `settings.json`，通过 `ProtocolSession.params` 进入 session
- **运行时协商**：注册帧返回的随机码 → unpack 中计算派生密钥 → 通过 `session_updates` 写入 state
- **config.yaml 默认值**：`session` 段的默认值在 `ProtocolFacade.switch_protocol()` 时注入 params

### 3.3 三种安全层级模式

本工程中的协议呈现三种复杂度：

**模式 A：无安全层（HIL 协议）**
```
payload → CRC16 → 嵌入帧 → 返回
```
最简单的管道。pack 时计算 CRC 追加到帧尾，unpack 时校验 CRC。无加密无 MAC。

**模式 B：单层安全（新金卡协议 jk_protocol）**
```
payload → DES-ECB 加密前8字节 → CRC16 校验 → 嵌入帧 → CS 校验 → 帧头帧尾
```
加密和校验在一个 `SecurityLayer(Subconstruct)` 中处理，用 construct 库声明式定义。

**模式 C：双层安全（100 协议 / gh_protocol）**
```
payload → AES-ECB+PKCS7 加密 → HMAC-SHA256 MAC → 嵌入帧 → CRC16 → 帧头帧尾
```
最复杂管道。安全模式通过 `_function_rule_mode` 按功能码+方向自动决策：
- 04H 下行：无数据域 → mode="none"
- 05H 下行：密文+MAC → mode="cipher_mac"
- 09H 上下行：明文+MAC → mode="plain_mac"

密钥派生涉及主密钥 + 终端随机码：
```
enc_key = HMAC-SHA256(主密钥, 随机码)[:16]
mac_key = AES-128-ECB(主密钥, 随机码)[:16]
```

### 3.4 安全决策的可覆盖性

`ctx.overrides` 允许 GUI 单次覆盖安全策略。codec 在 pack 中合并：

```python
def pack(self, ctx: PackContext) -> bytes:
    # 自动决策
    auto_mode = self._decide_mode(ctx.command_id, ctx.is_uplink)
    # GUI 覆盖优先
    mode = ctx.overrides.get("security_mode") or auto_mode
```

---

## 4. 数据类型系统：边界内与边界外

### 4.1 类型注册表

`TypeRegistry` 是全局单例，存储 `type_name → FieldType 子类` 映射。所有类型在 `app/infrastructure/validators/core/registry.py` 中管理，在 `app/infrastructure/validators/types/builtin.py` 底部完成注册。

### 4.2 已注册类型完整清单

| 注册名 | 对应类 | 长度(byte) | 说明 |
|--------|--------|-----------|------|
| `hex` | `HexField` | 配置决定 | 定长十六进制块，不足自动补 0 |
| `raw_hex` | `RawHexField` | 动态 | 变长十六进制数据（需配合 `dynamic: true`） |
| `uint8` | `UInt8Field` | 1 | 无符号 8 位整数 |
| `uint16` | `UInt16Field` | 2 | 无符号 16 位整数 |
| `uint32` | `UInt32Field` | 4 | 无符号 32 位整数 |
| `int16` | `Int16Field` | 2 | 有符号 16 位整数 |
| `int32` | `Int32Field` | 4 | 有符号 32 位整数 |
| `string` | `StringField` | 配置决定 | UTF-8 字符串，不足补 `0x00` |
| `bcd` | `BCDField` | 配置决定 | BCD 编码 |
| `bcd_u` | `BCDUField` | 配置决定 | 无符号 BCD 编码 |
| `bcd_u_array` | `BCDUArrayField` | 配置决定 | BCD 数组 |
| `yymmddhhmmss` | `YYMMDDHHMMSSField` | 6 | 年月日时分秒 BCD |
| `yymmddhhmm` | `YYMMDDHHMMField` | 5 | 年月日时分 BCD |
| `yymmddhh` | `YYMMDDHHField` | 4 | 年月日时 BCD |
| `yymmdd` | `YYMMDDField` | 3 | 年月日 BCD |
| `hhmmss` | `HHMMSSField` | 3 | 时分秒 BCD |
| `hhmm` | `HHMMField` | 2 | 时分 BCD |
| `yymm` | `YYMMField` | 2 | 年月 BCD |
| `record` | `RecordField` | 配置决定 | 嵌套记录结构 |
| `record_array` | `RecordArrayField` | 配置决定 | 记录数组 |
| `bitfield` | `BitField` | 配置决定 | 位域，需配置 `bits` 列表 |
| `enum` | `EnumField` | 配置决定 | 枚举值，需配置 `enum_values` 映射 |

### 4.3 类型选择决策树

从协议文档中的类型描述到 FieldType 注册名的映射规则：

```
协议文档描述
  │
  ├─ 提到 "十六进制" / "hex" / "原始数据"
  │   ├─ 固定长度 → hex (length=N)
  │   └─ 变长（由其他字段指定长度）→ raw_hex + dynamic: true
  │
  ├─ 提到 "无符号整数" / "unsigned" / "U8/U16/U32"
  │   ├─ 1 字节 → uint8
  │   ├─ 2 字节 → uint16
  │   └─ 4 字节 → uint32
  │
  ├─ 提到 "有符号整数" / "signed" / "S16/S32"
  │   ├─ 2 字节 → int16
  │   └─ 4 字节 → int32
  │
  ├─ 提到 "字符串" / "ASCII" / "UTF-8" / "字符数组"
  │   └─ string (length=文档声明的字节数)
  │
  ├─ 提到 "BCD" / "8421码" / "压缩BCD"
  │   ├─ 单个 BCD 值 → bcd_u
  │   ├─ BCD 数组 → bcd_u_array
  │   └─ 日期时间 BCD → 按格式选择 yymmddhhmmss / yymmddhhmm / yymmdd / hhmmss 等
  │
  ├─ 提到 "位" / "bit" / "bit0~bit7" / "D0~D7"
  │   └─ bitfield (需逐位列出 bits 配置)
  │
  ├─ 提到 "枚举" / "取值: 0=xxx, 1=yyy" / "状态码"
  │   └─ enum (需列出 enum_values)
  │
  └─ 提到 "结构体" / "嵌套" / "TLV"
      ├─ 单个结构体 → record
      └─ 结构体数组 → record_array
```

**字节序注意事项**：
- 协议文档中 "高字节在前" / "大端" / "MSB first" → `byte_order: big`（默认值）
- 协议文档中 "低字节在前" / "小端" / "LSB first" → `byte_order: little`
- 不写 `byte_order` 时默认 `big`

### 4.4 自定义类型扩展

当 22 种内置类型无法覆盖协议需求时（例如特殊的浮点编码、非标准压缩算法），通过继承 `FieldType` 并注册来扩展：

```python
from app.infrastructure.validators.core.base import FieldType
from app.infrastructure.validators.core.registry import TypeRegistry


class Ieee754SingleField(FieldType):
    """IEEE 754 单精度浮点 (4 bytes)。"""
    TYPE_DEF = TypeDefinition(
        length=4, description="IEEE 754 single-precision float"
    )

    @classmethod
    def get_default_value(cls, length: int) -> str:
        return "0.0"

    def to_bytes(self, value: Any) -> bytes:
        import struct
        return struct.pack(">f", float(value))

    def from_bytes(self, data: bytes) -> str:
        import struct
        return str(struct.unpack(">f", data)[0])

    # _convert_value, _check_range 按需实现


TypeRegistry.register("ieee754_single", Ieee754SingleField)
```

自定义类型放在 `plugins/<key>/` 目录下，与 codec.py 同级，在 codec 的 `on_load()` 中触发 import 即可完成注册。

---

## 5. 参数模型：持久化与会话的合并规则

### 5.1 参数的三层来源

会话状态 (`ProtocolSession`) 由三个来源合并，优先级从高到低：

```
优先级 高 → 低

1. ctx.overrides (GUI 高级选项面板单次覆盖)
     ↓ 覆盖
2. session.state (运行时协商结果，如注册帧返回的随机码)
     ↓ 覆盖
3. session.params (持久化参数 + config.yaml 默认值)
```

### 5.2 config.yaml 中的参数定义

```yaml
parameters:
  des_key:
    label: "DES 密钥"          # GUI 显示名
    default: "C83E7386FA4DB629" # 默认值
    persistent: true            # true=存 settings.json，关闭软件后保留
    secret: true                # true=GUI 用密码框显示
    description: "DES-ECB 加密密钥"
```

参数类型 (`type` 字段) 决定 GUI 用什么控件展示：

| type | GUI 控件 | 适用场景 |
|------|---------|---------|
| (不写) | 单行文本框 | 普通字符串/数字参数 |
| `combo_box` | 下拉框 | 有限选项（需配 `options` 列表） |
| `line_edit` | 单行输入 | 明确需要行编辑的场景 |
| `text_edit` | 多行文本 | 长文本/多行参数 |

### 5.3 session 段的默认值注入

```yaml
session:
  random_000c: ""           # 运行时协商密钥的占位
  main_key_default: ""      # 主密钥（默认）
  main_key_non_default: ""  # 主密钥（非标）
```

`session` 段的 key 在 `ProtocolFacade.switch_protocol()` 时作为 state 初始值注入。运行时 codec 通过 `session_updates` 写回新值。

### 5.4 参数合并的完整流程

```
软件启动
  │
  ├─ 1. 加载 config.yaml → 解析 parameters 段 → 注入 params
  ├─ 2. 加载 settings.json → 覆盖 params 中有 persisted 标记的值
  ├─ 3. 解析 session 段 → 注入 state 初始值
  │
协议切换 (switch_protocol)
  │
  ├─ 4. 重新执行步骤 1-3
  │
每次 send()
  │
  ├─ 5. ProtocolSession.snapshot() → {**params, **state} → 作为 ctx.session
  └─ 6. codec.pack(ctx) 从 ctx.session 取密钥/序列号/地址
```

---

## 6. UI 契约：界面能做什么，能向下传递什么

### 6.1 GUI 的唯一向下通道

GUI 通过两个渠道向协议层传递信息：

| 渠道 | 路径 | 生命周期 | 用途 |
|------|------|---------|------|
| 参数面板 | params → session.snapshot() | 持久化 | 密钥、地址、序列号起始值 |
| 高级选项 | overrides → PackContext.overrides | 单次发送 | 安全模式覆盖、MID 策略覆盖 |

**GUI 不能直接做的事**：
- 不能直接修改 codec 内部状态
- 不能直接构造帧字节
- 不能绕过 Channel 直接调用 transport.write()

### 6.2 overrides 的声明与传递

在 `commands.yaml` 中声明哪些指令属性可以被 GUI 覆盖：

```yaml
instruction_attributes:
  security_mode:
    values: [auto, plain_mac, cipher_mac, none]
    codec_override: true      # 标记此属性可通过 overrides 传递
```

`codec_override: true` 的属性在 `ProtocolChannel.send()` 中从 `field_data` 剥离，放入 `PackContext.overrides`，不参与 BinaryFormatter 序列化。

### 6.3 GUI 展示的元数据来源

| GUI 展示内容 | 数据来源 |
|-------------|---------|
| 协议列表 | `ProtocolRegistry` 扫描 `plugins/` 目录 |
| 协议名 | `config.yaml` 的 `protocol.name` |
| 命令列表 | `commands.yaml` 的 `commands` 键 |
| 命令名（中文） | 命令的 `name` 字段或键名 |
| 字段名（中文） | `fields[].name` |
| 字段类型信息（提示） | `fields[].type` + `fields[].length` + `fields[].description` |
| 参数字段 | `config.yaml` 的 `parameters` 段 |
| 实时数据 | `ParsedFrame` 通过 Qt Signal 推送 |

---

## 7. 接入决策树：同构 vs 异构

### 7.1 判断流程

拿到新协议文档后，按以下顺序判断是走"同构拷贝"还是"全新开发"路径：

```
新协议文档
  │
  ├─ 步骤 1：对比帧格式
  │   Q: 帧头/帧尾/长度域位置/CRC 算法与已有协议完全一致？
  │   ├─ 是 → 继续
  │   └─ 否 → 全新开发 (codec 从头写)
  │
  ├─ 步骤 2：对比安全层
  │   Q: 加密算法/MAC 算法/密钥派生逻辑与已有协议完全一致？
  │   ├─ 是 → 继续
  │   └─ 否 → 检查是否可以只替换安全层，帧结构部分仍可复用
  │       ├─ 帧结构相同，仅安全层不同 → 同构拷贝，替换安全层
  │       └─ 帧结构也不同 → 全新开发
  │
  ├─ 步骤 3：对比命令模型
  │   Q: 命令表结构（command_id 编码方式、字段序列化规则）与已有协议一致？
  │   ├─ 是 → **同构拷贝** 从最接近的已有协议拷贝底层文件
  │   └─ 否 → 全新开发
  │
  └─ 步骤 4：确认独立性
      无论同构还是全新，最终产物必须是独立目录，禁止跨协议 import
```

### 7.2 同构拷贝的具体步骤

参见 `references/technical-extension.md` 的 C 节，补充以下检查点：

1. 拷贝后必须修改的项（逐项核对）：
   - [ ] 类名（如果 frame_security.py 中有协议相关的类名）
   - [ ] 帧头/帧尾常量（HEAD/TAIL）
   - [ ] command_id 映射表（DID 范围、功能码映射）
   - [ ] 随机码偏移量和长度（如果安全层有密钥协商）
   - [ ] MAC 长度
   - [ ] 默认安全模式
2. 拷贝后必须保留的项：
   - CRC 算法实现（如果一致）
   - 加密/MAC 算法实现（如果一致）
   - 帧构建/解析的流程代码
3. 独立性验证：
   ```python
   # 在 codec.py 中运行 ast 解析，确认无跨插件 import
   import ast, sys
   with open("plugins/new_protocol/codec.py") as f:
       tree = ast.parse(f.read())
   for node in ast.walk(tree):
       if isinstance(node, ast.ImportFrom):
           if "plugins" in node.module and "new_protocol" not in node.module:
               print(f"违规跨协议import: {node.module}")
   ```

### 7.3 完全异构时的全新开发检查清单

见第 8 节"实现检查清单"。

---

## 8. 实现检查清单

### 8.1 Phase 0：文档分析（本 phase 检查点）

- [ ] 协议文档已转为可搜索的文本格式
- [ ] 帧格式已精确文档化（每个字节的偏移/长度/含义/常量值）
- [ ] 所有命令的 command_id 已提取
- [ ] 每个命令的 request/response 字段已提取（名称/类型/长度/字节序）
- [ ] CRC/校验算法参数已提取（多项式/初始值/校验范围/字节序）
- [ ] 安全层参数已提取（加密算法/密钥来源/MAC 算法）
- [ ] 错误码表已提取
- [ ] 测试向量已提取（至少 3 条：1 条简单请求，1 条含 payload 请求，1 条响应）
- [ ] 类型映射已完成（协议文档类型 → FieldType 注册名）

### 8.2 Phase 1：配置与命令定义

- [ ] `plugins/<key>/config.yaml` 已创建
  - [ ] `protocol.name` / `protocol.version` 已填写
  - [ ] `parameters` 段已定义（密钥、地址等持久化参数）
  - [ ] `session` 段已定义（运行时状态初始值）
  - [ ] `frame` 段已定义（帧头/帧尾等常量，如有）
  - [ ] `security` 段已定义（安全配置，如有）
- [ ] `config/protocols/<Display Name>/commands.yaml` 已创建
  - [ ] 所有命令的 request/response 已定义
  - [ ] 所有字段的 type/length/byte_order/default 已填写
  - [ ] 枚举字段的 enum_values 已列出
  - [ ] 位域字段的 bits 已逐位定义
  - [ ] 动态字段（raw_hex + dynamic: true）已标记
  - [ ] instruction_attributes 已按需配置（codec_override 标记）

### 8.3 Phase 2：Codec 实现

- [ ] `plugins/<key>/codec.py` 已创建
  - [ ] `Codec` 类继承 `ProtocolCodec`
  - [ ] `on_load(config)` 已实现（读取 config.yaml 参数）
  - [ ] `pack(ctx)` 已实现（payload → 完整帧）
  - [ ] `unpack(data, session)` 已实现（完整帧 → UnpackResult）
  - [ ] 需要时覆盖 `split_frames(data)`（字节流粘包处理）
  - [ ] `explain_decision()` 已实现（安全模式决策的可视化说明）
- [ ] 安全层完全在 codec 内部处理（框架不感知加解密）
- [ ] 异常处理：pack 失败抛 `ProtocolPackError`，unpack 失败抛 `ProtocolUnpackError`
- [ ] 无跨协议 import（独立性验证通过 ast 检查）

### 8.4 Phase 3：测试

- [ ] 契约测试：从协议文档提取的测试向量已转为 pytest
  - [ ] 每条向量测试：`codec.pack()` 输出与 hex dump 逐字节一致
  - [ ] 每条向量测试：`codec.unpack()` 解析结果与文档字段一致
- [ ] 单元测试：`test_pack_minimal` / `test_unpack_valid`
- [ ] 异常路径测试：CRC 错误 / MAC 错误 / 帧头错误
- [ ] 集成测试：`MockTransport + ProtocolChannel` 收发闭环
- [ ] 往返测试：`pack → unpack` 后 payload 不变

### 8.5 Phase 4：验证与交付

- [ ] 在 GUI 中协议列表可看到新协议
- [ ] 命令列表正确加载
- [ ] 参数面板正确展示
- [ ] 发送简单命令可看到帧出现在日志中
- [ ] 如有可能，与真实设备联调验证
- [ ] 更新 `docs/references/` 下的协议参考文档
