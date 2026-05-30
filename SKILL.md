---
name: protocol-integration-workflow
description: First-principles guide for integrating a new communication protocol into ToolsKing. Covers the layered architecture (Transport/Codec/Channel/Runtime/Facade), frame encoding/decoding pipelines, security layer patterns, data type boundaries, parameter persistence, and UI contract. Use when analyzing, designing, or implementing any protocol plugin.
---

# ToolsKing 协议接入：第一性原理

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

`TypeRegistry` 是全局单例，存储 `type_name → FieldType 子类` 映射。所有类型在 `app/core/vali
