---
name: protocol-integration-workflow
description: First-principles guide for integrating a new communication protocol into ToolsKing. Covers the layered architecture (Transport/Codec/Channel/Runtime/Facade), frame encoding/decoding pipelines, security layer patterns, data type boundaries, parameter persistence, and UI contract. Use when analyzing, designing, or implementing any protocol plugin.
---

# ToolsKing 协议接入：架构契约与实现参考

> **⏱️ 最小上下文路径**：回答以下 3 个问题，确定你需要的文档子集。
>
> ```
> Q1: 数据域是固定字段顺序，还是 TLV 自描述？
>   ├─ 固定 schema → 继续 Q2
>   └─ TLV/自描述 → 跳读 commands-yaml-template.md §12（可跳过 §1-11）
>
> Q2: 协议是否有加密/认证层？
>   ├─ 无安全层   → 本文档 §3.3 模式 A 即可
>   ├─ 仅加密     → 本文档 §3.3 模式 B + §3.4
>   └─ 加密+MAC   → 本文档 §3 全读 + data-domain-polymorphism.md（如有表型差异）
>
> Q3: 帧格式/安全层与已有协议同构？
>   ├─ 同构 → 走本文档 §7 决策树 + technical-extension.md §B（同构拷贝）
>   └─ 异构 → 走 technical-extension.md §A（分步实现指南）
> ```
>
> **🏃 专家路径**：如果你已接入过至少一套协议，可以跳过 §1（分层架构）和 §4（类型系统），
> 直接进入 §2（接口契约）→ 选择 §3（安全层，按 Q2 决定读多少）→ §7（同构决策，按 Q3）。
>
> ---
>
> **完整工作流**：本文件是协议接入的架构与实现参考。如果你刚拿到一份新的 PDF/Word/MD 协议文档，请**分阶段阅读**：
>
> **第一阶段：建立全局认知**
> 1. `references/document-analysis-workflow.md` — Phase 0: 从文档中系统提取所有实现所需信息
> 2. **本文档 §1-§2** — 理解分层架构与 codec 契约（`PackContext` / `UnpackResult` / `decoded_fields_mode` 是 commands.yaml 设计的基础）
>
> **第二阶段：配置与命令定义**
> 3. `references/field-type-mapping.md` — 把文档中的类型描述映射到框架的 FieldType
> 4. `references/commands-yaml-template.md` — 编写 commands.yaml
> 5. **本文档 §5-§6** — 参数模型与 UI 契约（`overrides` / `session` / `params` 直接影响 commands.yaml 的参数段设计）
> 6. `references/data-domain-polymorphism.md` — 如果协议存在表型差异（同一命令不同表型返回不同字段），在此阶段设计参数→会话→条件解析链路
>
> **第三阶段：实现与测试**
> 6. **本文档 §3, §7** — 安全层模式 + 同构 vs 异构决策树
> 7. `references/technical-extension.md` — 代码级参考（完整 codec 示例、异常处理、故障排查）
> 8. `references/test-vector-extraction.md` — 用文档中的 hex dump 编写契约测试
> 9. `references/conformance-test-matrix.md` — Phase 3.5: 系统化覆盖全部命令、异常路径、安全模式、状态机

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

> **Phase 3.5 补充**：完成 Phase 3 契约测试后，如需系统化验证全部命令空间、异常路径、安全模式组合和会话状态机，请参阅 `references/conformance-test-matrix.md`。
>
> **参考文档速查**：
> | 文件 | 内容 | 阶段 |
> |------|------|------|
> | `references/document-analysis-workflow.md` | Phase 0：PDF/Word/MD 协议文档系统化分析流程 | 第一阶段 — 最先阅读 |
> | 本文档 §1-§2 | 分层架构、PackContext/UnpackResult/decoded_fields_mode 契约 | 第一阶段 — 必须，影响 commands.yaml 设计 |
> | `references/field-type-mapping.md` | 协议文档类型 → 22 种 FieldType 精确映射表 | 第二阶段 — 提取命令字段时对照 |
> | `references/commands-yaml-template.md` | 从命令表到 commands.yaml 的完整转换模板。第 12 节 TLV/自描述格式，第 13 节表具计量数据强制解析规则 | 第二阶段 — 编写 commands.yaml 时对照 |
> | 本文档 §5-§6 | 参数模型（params/session/overrides 合并规则）与 UI 契约 | 第二阶段 — 影响 commands.yaml 的参数段设计 |
> | `references/data-domain-polymorphism.md` | **数据域多态**：表型决定字段集，决定性参数 → 会话状态 → 条件解析的完整模式 | 第二阶段 — 协议存在表型差异时必读 |
> | 本文档 §3, §7 | 安全层三种模式 + 同构 vs 异构决策树 | 第三阶段 — codec 实现前必读 |
> | `references/technical-extension.md` | 代码级参考：完整 codec 示例、异常处理、故障排查 | 第三阶段 — codec 实现时参考 |
> | `references/test-vector-extraction.md` | 测试向量定位→提取→pytest 契约测试生成 | 第三阶段 — 实现 codec 后编写单条向量测试 |
> | `references/conformance-test-matrix.md` | **Phase 3.5**：命令全覆盖 × 六类测试类型，含自动生成脚本、pytest 参数化框架、计量院检测映射 | 第三阶段 — 送检前 / 发布前完整执行 |
> | `references/anti-patterns.md` | **反模式清单**：7 种最常见错误（硬编码偏移量、session_updates 滥用、raw_hex 模糊化计量数据等）及修正方法 | 全阶段 — 实现过程中随时对照 |

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

### 2.2 接收循环的驱动机制

在讨论接收管线之前，必须先理解**谁驱动了接收**——这直接影响 codec 的实现约束。

```
              ┌──────────────────────────┐
              │ Transport 层              │
              │                           │
              │ SerialTransport:          │
              │  独立读线程 → data_cb()   │  ← 线程回调（异步）
              │                           │
              │ TcpTransport:             │
              │  独立读线程 → data_cb()   │  ← 线程回调（异步）
              │                           │
              │ MockTransport:            │
              │  测试代码直接调用          │  ← 同步调用
              └──────────┬───────────────┘
                         │ data_cb(data: bytes)
                         ▼
              ProtocolChannel._on_raw_data(data)
```

**关键约束**：
- 串口/TCP 的 `data_cb` 在**独立 I/O 线程**中调用，不是主线程
- 这意味着 `codec.unpack()` 在 I/O 线程中执行
- **codec 实现者必须注意**：如果 `unpack()` 中有耗时操作（如 AES 解密大 payload），会阻塞后续帧的接收
- 推荐：codec 内部只做轻量级字节操作（切片、查表、位运算）。如需耗时密码运算，将 `unpack()` 设计为剥离安全层后的 payload 不做深入解析，把结构化解析延迟给 BinaryFormatter

**主动上报帧的特殊性**：
- 主动上报帧**不由 host 的 send() 触发**，而是由设备主动推送
- 接收管线对主动上报帧和响应帧的处理路径完全相同——都走 `_on_raw_data → codec.unpack → CommandStore 匹配`
- 唯一的区别：主动上报帧在 commands.yaml 中可能只定义了 `response` 侧（没有 `request`），但不影响 Channel 的匹配逻辑

### 2.3 接收管线（上行）

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
       │
       │  ═══ 以下步骤由框架自动完成，codec 无需关心 ═══
       │
       ├─ 4. CommandStore 查 command_id → command_name + direction
       ├─ 5. 按 decoded_fields_mode 构建最终字段:
       │     schema: 用 BinaryFormatter 反向解析 payload
       │     replace: 直接用 codec 返回的 decoded_fields
       │     merge: schema 打底，decoded_fields 覆盖
       ├─ 6. 构造 ParsedFrame → 入 _rx_queue
       └─ 7. 通知 _parsed_callbacks（GUI 通过此路径拿到解析结果）
```

**对 codec 实现者而言，只需关注步骤 1-3。**

**UnpackResult 的 6 个字段是 codec 能输出的全部**：

| 字段 | 类型 | 含义 |
|------|------|------|
| `command_id` | `int` | 从帧中提取的命令 ID（整数） |
| `payload` | `bytes` | 解密/解包后的业务字节 |
| `session_updates` | `dict` | 需要写回 session.state 的键值。**关键用途**：存储表型等决定性参数，供后续多态解析使用（详见 `references/data-domain-polymorphism.md`） |
| `decoded_fields` | `dict\|None` | codec 自行解析的结构化结果 |
| `decoded_fields_mode` | `str` | "schema" / "replace" / "merge" |
| `meta` | `dict` | 仅调试/日志用，不参与业务 |

**decoded_fields_mode 的三种模式**：

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `schema` | 忽略 decoded_fields，仅用 commands.yaml schema 反向解析 | 标准命令，字段结构与定义一致 |
| `replace` | 用 decoded_fields 完全替代 schema 解析 | codec 自行完成所有解析（加密嵌套、复杂TLV、记录数据等） |
| `merge` | schema 解析打底，decoded_fields 覆盖同名键 | 部分字段需要 codec 特殊处理 |

> **如何选择模式**：如果你的协议使用 TLV / Tag-Length-Value 自描述格式，
> 请参考 `references/commands-yaml-template.md` **第 12 节**，其中包含完整的决策树（§12.2）、
> 各模式下的字段定义粒度指南（§12.3–12.4）、以及 codec ↔ commands.yaml 的 key 名契约（§12.5）。

### 2.4 split_frames：多帧拆分

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

### 3.3 三种安全层级

协议的安全复杂度大致分为三档（以下用抽象模式描述，不绑定特定协议）：

**模式 A：无安全层**
```
payload → CRC → 嵌入帧 → 返回
```
只需计算和校验 CRC。无加密、无 MAC。适用场景：调试协议、内网封闭协议。

**模式 B：单层加密**
```
payload → 对称加密 → CRC → 嵌入帧 → 帧头帧尾
```
在 CRC 之前对 payload（或 payload 的一部分）做加密。密钥通常是预设的固定值。
适用场景：有保密需求但不需要防篡改的协议。

**模式 C：加密 + 消息认证（MAC）**
```
payload → 加密 → 追加 MAC → 嵌入帧 → CRC → 帧头帧尾
```
最完整的管道。安全策略通常需要**按命令/方向自动决策**（例如某些命令无数据域则跳过加密、
某些命令只需 MAC 不需要加密）。密钥可能来自固定预设值，也可能来自**注册帧密钥协商**
（设备返回随机码，主机用主密钥派生会话密钥）。

你的协议属于哪一档，决定了 codec 的复杂度上限。从模式 A 开始实现，逐步叠加。

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

所有 FieldType 通过 `TypeRegistry.register()` 全局注册。新增的类型（见 §4.4）在 codec 的 `on_load()` 中导入即可自动完成注册。

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
| `bcd` | `BCDField` | 配置决定 | 有符号 BCD（支持 `+/-`、小数点和符号字节） |
| `bcd_u` | `BCDUField` | 配置决定 | 无符号 BCD 编码 |
| `bcd_u_array` | `BCDUArrayField` | 配置决定 | BCD 数组 |
| `yymmddhhmmss` | `YYMMDDHHMMSSField` | 6 | 年月日时分秒（默认 BCD，可设 `encoding: hex`） |
| `yymmddhhmm` | `YYMMDDHHMMField` | 5 | 年月日时分（默认 BCD，可设 `encoding: hex`） |
| `yymmddhh` | `YYMMDDHHField` | 4 | 年月日时（默认 BCD，可设 `encoding: hex`） |
| `yymmdd` | `YYMMDDField` | 3 | 年月日（默认 BCD，可设 `encoding: hex`） |
| `hhmmss` | `HHMMSSField` | 3 | 时分秒（默认 BCD，可设 `encoding: hex`） |
| `hhmm` | `HHMMField` | 2 | 时分（默认 BCD，可设 `encoding: hex`） |
| `yymm` | `YYMMField` | 2 | 年月（默认 BCD，可设 `encoding: hex`） |
| `record` | `RecordField` | 配置决定 | 嵌套记录结构 |
| `record_array` | `RecordArrayField` | 配置决定 | 记录数组 |
| `bitfield` | `BitField` | 配置决定 | 位域，需配置 `bits` 列表 |
| `enum` | `EnumField` | 配置决定 | 枚举值，需配置 `enum_values` 映射 |

### 4.3 类型选择决策树

从协议文档中的类型描述到 FieldType 注册名的映射规则。

**完整决策矩阵（表格版）** 见 `references/field-type-mapping.md` 第 9 节"快速决策矩阵"。
以下为快速速查：

- 十六进制固定长度 → `hex`，变长 → `raw_hex` + `dynamic: true`
- 无符号整数 (1/2/4 字节) → `uint8` / `uint16` / `uint32`
- 有符号整数 (2/4 字节) → `int16` / `int32`
- 无符号 BCD（纯数字）→ `bcd_u`；有符号 BCD（含 +/- 和小数点）→ `bcd`
- BCD 数组 → `bcd_u_array`
- 日期时间 → 按格式选 `yymmddhhmmss` ~ `yymm`，默认 BCD 编码，可设 `encoding: hex`
- 字符串 → `string`
- 每个 bit 有独立含义 → `bitfield`（需列出 `bits` 配置）
- 有限个预定义取值 → `enum`（需列出 `enum_values`）
- 嵌套结构体 → `record`；结构体数组 → `record_array`

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

`session` 段的 key 在协议切换时作为 state 初始值注入。运行时 codec 通过 `session_updates` 写回新值。

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

### 8.1 Phase 0：文档分析

> ⛔ **门控条件**：进入本 Phase 的前提——已拿到可搜索的协议文档文本

从协议文档（PDF/Word/MD）中系统提取实现所需全部信息。

> **详细检查清单** → `references/document-analysis-workflow.md` 第 9 节
>
> **核心产出**：帧格式表、命令表、CRC/安全参数、错误码表、测试向量（≥3条）
>
> **常见错误** → `references/anti-patterns.md` 无直接对应（Phase 0 错误通常在 Phase 1-2 暴露）

### 8.2 Phase 1：配置与命令定义

> ⛔ **门控条件**：进入本 Phase 前，确认 Phase 0 以下产出已就绪：
> - [ ] 帧格式已精确文档化（每个字节的偏移/长度/含义）
> - [ ] 所有命令的 command_id 已提取并转为 hex 格式
> - [ ] CRC/校验算法参数已确定（多项式/初始值/范围）
> - [ ] 安全层参数已确认（加密算法/密钥来源）
> - [ ] 至少 3 条测试向量的 hex dump 已提取
>
> 如果以上任一项未完成，**回到 Phase 0**。

创建 `config.yaml` 和 `commands.yaml`，将 Phase 0 产出转化为框架可用的结构化定义。

> **通用检查清单** → `references/commands-yaml-template.md` 第 11 节
>
> **TLV/自描述格式专项** → `references/commands-yaml-template.md` 第 12.7 节
>
> **表具行业计量数据专项** → `references/commands-yaml-template.md` 第 13.7 节
>
> **类型映射参考** → `references/field-type-mapping.md` 第 9 节决策矩阵
>
> **核心产出**：可加载的 `commands.yaml`、通过字段验证的 `config.yaml`
>
> **常见错误** → `references/anti-patterns.md` §5（raw_hex 模糊化计量数据）、§7（忽略主动上报帧时机）

### 8.3 Phase 2：Codec 实现

> ⛔ **门控条件**：进入本 Phase 前，确认 Phase 1 以下产出已就绪：
> - [ ] `config/protocols/<协议名>/commands.yaml` 已创建且可正常加载
> - [ ] `plugins/<key>/config.yaml` 已创建且参数段完整
> - [ ] 已确定走同构拷贝还是全新开发（§7.1 决策树）
>
> 如果以上任一项未完成，**回到 Phase 1**。

实现 `plugins/<key>/codec.py`，继承 `ProtocolCodec`，完成帧编解码和异常处理。

> **代码级参考** → `references/technical-extension.md`（完整 codec 示例、异常处理、故障排查）
>
> **架构契约** → 本文档 §2（PackContext/UnpackResult） + §3（安全层） + §7（同构拷贝决策）
>
> **核心产出**：`pack()`/`unpack()` 通过契约测试，安全层正确处理，独立性验证通过
>
> **常见错误** → `references/anti-patterns.md` §1（硬编码偏移量）、§3（session_updates 滥用）、§4（全局状态）、§6（重复应用 scale）

### 8.4 Phase 3：契约测试

> ⛔ **门控条件**：进入本 Phase 前，确认 codec 的 `pack()` 和 `unpack()` 方法已实现且无语法错误

用 Phase 0 提取的 hex dump 测试向量验证 codec 字节级正确性。

> **完整指南** → `references/test-vector-extraction.md`
>
> **核心产出**：pack 输出与 hex dump 逐字节一致、unpack 解析结果与文档一致、往返测试通过

### 8.5 Phase 3.5：一致性测试（送检前必做）

> ⛔ **门控条件**：进入本 Phase 前，确认 Phase 3 契约测试全部通过

系统化覆盖全部命令空间 × 六类测试类型，验证协议实现的整体正确性。

> **完整指南与检查清单** → `references/conformance-test-matrix.md`（含自动生成脚本、pytest 框架、计量院检测映射）
>
> **核心产出**：一致性测试矩阵、覆盖率报告、T1-T6 全量通过

### 8.6 Phase 4：验证与交付

- [ ] 在 GUI 中协议列表可看到新协议
- [ ] 命令列表正确加载
- [ ] 参数面板正确展示
- [ ] 发送简单命令可看到帧出现在日志中
- [ ] 如有可能，与真实设备联调验证
- [ ] 更新 `docs/references/` 下的协议参考文档
