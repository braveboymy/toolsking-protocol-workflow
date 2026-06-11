# 协议接入反模式清单

> 📏 ~960 tokens | 必读等级: ★☆☆（实现过程中随时对照）| 前置: 无

配合 `protocol-integration-workflow` 使用。

以下列出表具协议接入过程中最常见的 7 种错误模式及其修正方法。

---

## 反模式 1：在 codec 中硬编码字段偏移量

**症状**：

```python
# 错！偏移量硬编码在 codec 里
def unpack(self, data, session):
    payload = self._decrypt(data[9:-3])
    return UnpackResult(
        payload=payload,
        command_id=int.from_bytes(payload[0:2], "big"),
        decoded_fields={
            "累计用气量": int.from_bytes(payload[2:6], "big"),
            "阀门状态": payload[6],
        },
        decoded_fields_mode="replace",
    )
```

**为什么错**：字段定义在 commands.yaml 中已经声明了 type/length。codec 硬编码偏移量意味着：
- 每次 commands.yaml 字段顺序调整，codec 也要改（两份真相源）
- 字段顺序和类型定义在 YAML 和 Python 之间可能漂移
- GUI 侧的字段元数据（unit/scale）来自 YAML，但 codec 的解析不经过 BinaryFormatter，元数据不会被应用

**正确做法**：只在 codec 中做"无法由 schema 表达的"解析（TLV Tag 提取、加密块解密、表型条件分支）。字段级别的序列化交给 BinaryFormatter：

```python
# 好 — codec 只负责剥离帧头帧尾/解密，字段解析由框架完成
def unpack(self, data, session):
    payload = self._decrypt(data[9:-3])
    return UnpackResult(command_id=cmd_id, payload=payload)
    # 不提供 decoded_fields → 框架用 schema 模式自动解析
```

---

## 反模式 2：把不同协议的逻辑写进同一个 codec

**症状**：

```python
# 错！通过 if-else 分支区分协议
class Codec(ProtocolCodec):
    def pack(self, ctx):
        if self._protocol_name == "gh":
            return self._pack_gh(ctx)
        elif self._protocol_name == "100":
            return self._pack_100(ctx)
        else:
            return self._pack_stcb(ctx)
```

**为什么错**：违反模块独立性原则。修改一个协议的逻辑时可能意外破坏另一个。测试一个协议时需要覆盖所有分支。

**正确做法**：每个协议独立一个目录（`plugins/<key>/`），共同底层代码通过同构拷贝独立维护（SKILL.md §7）。

---

## 反模式 3：session_updates 当作全局临时变量池

**症状**：

```python
# 错！每次 unpack 都往里塞临时数据
return UnpackResult(
    session_updates={
        "last_payload_len": len(payload),
        "last_func_code": func_code,
        "last_crc_ok": crc_ok,
        "last_frame_raw": data.hex(),
    },
)
```

**为什么错**：
- `session_updates` 每次 unpack 都会触发 `session.apply_updates()`，写入 session.state
- session.state 在内存中持续增长，临时数据永不清理
- 这些"上次解析结果"完全应该在 codec 的实例变量中维护，而不是污染全局 session

**正确做法**：`session_updates` 只写**需要跨帧持久化**的数据。临时状态放在 codec 实例变量中：

```python
class Codec(ProtocolCodec):
    def on_load(self, config):
        self._last_func_code = 0    # 实例变量，仅存活于当前 codec 对象
        self._last_crc_ok = True

    def unpack(self, data, session):
        ...
        self._last_func_code = func_code  # 放在实例变量
        return UnpackResult(
            session_updates={"seq": seq},  # 只写需要跨帧持久化的
        )
```

---

## 反模式 4：在 unpack 中修改除 session 外的全局状态

**症状**：

```python
# 错！修改类级别的全局状态
_global_seq = 0

class Codec(ProtocolCodec):
    def unpack(self, data, session):
        global _global_seq
        _global_seq += 1
```

**为什么错**：`unpack()` 在 I/O 线程中执行（SKILL.md §2.2）。类级别的全局变量没有线程安全保护。多个协议实例共享同一个全局变量会导致序列号跳变。

**正确做法**：所有需要跨调用持久化的状态通过 `session_updates` 写入 session，或使用 codec 实例变量（实例变量是单线程访问的，因为每个 Runtime 持有独立的 codec 实例）。

---

## 反模式 5：用 raw_hex 模糊化计量数据（"一票否决"级违规）

**症状**：

```yaml
# 错！累计量、金额等核心数据被压缩成 hex blob
response:
  fields:
    - name: 实时数据
      type: raw_hex
      length: 0
      dynamic: true
      description: "TLV 实时数据块，由 codec 解析"
```

**为什么错**：详见 `commands-yaml-template.md` §13.1。核心问题是：即使 codec 在 decoded_fields 中提供了正确的数值，commands.yaml 缺失了 type/unit/scale 元数据——GUI 无法格式化展示，数据库无法正确存储。

**正确做法**：每个计量字段独立定义。如果 codec 已经在 decoded_fields 中做了解析，用 `merge` 模式让 commands.yaml 提供元数据。

---

## 反模式 6：在 codec 中重复应用 scale

**症状**：

```python
# codec 里已经除了 scale
decoded_fields = {"累计用气量": raw_value / 100}
```

```yaml
# commands.yaml 里又声明了 scale
- name: 累计用气量
  type: uint32
  length: 4
  unit: m³
  scale: 0.01
```

**为什么错**：`merge` 模式下，框架 schema 解析不会执行（codec 已经提供了值）。但如果将来切换到 `schema` 模式或有人同时应用了 scale，值会被除了两次（raw/10000）。

**正确做法**：选择一种方式：
- **方案 A（推荐）**：codec 返回**原始传输值**（未除 scale），由 commands.yaml 的 scale 元数据让 GUI 层展示时除以 scale
- **方案 B**：codec 返回**最终展示值**（已除 scale），commands.yaml 不设 scale（或设 `scale: 1`），在 description 中标注"codec 已缩放"

---

## 反模式 7：忽略主动上报帧的发送时机

**症状**：将主动上报帧的解析逻辑和响应帧的解析逻辑混在一起，没有区分发送时机：

```python
def unpack(self, data, session):
    result = self._parse_frame(data, session)
    # 所有帧走同一条路径，没有区分主动上报 vs 请求响应
    return UnpackResult(command_id=result["did"], payload=result["raw_payload"])
```

**为什么错**：
- 主动上报帧可能在**任何时间**到达（包括正在发送命令时）
- 如果把主动上报帧当作对最近一次 send() 的响应，会导致字段匹配错乱
- 主动上报帧的 `session_updates`（如表型更新）可能在与 host 发送的注册帧做同样的事

**正确做法**：
- 在 commands.yaml 中明确标注哪些命令是主动上报（通常只有 response 侧）
- codec 内部区分主动上报帧和响应帧（通过功能码或 command_id 范围），分别处理
- 主动上报帧优先走"状态更新"路径（更新 `session_updates`），不走"响应匹配"路径
