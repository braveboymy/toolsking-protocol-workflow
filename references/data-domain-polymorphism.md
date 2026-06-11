# 数据域多态：表型驱动的字段差异化解析

> 📏 ~2,100 tokens | 必读等级: ★☆☆（仅协议有表型差异时需要）| 前置: SKILL.md §2, §5
> ⏩ 如果你的协议所有表型返回相同字段集，可以跳过本文档

配合 `protocol-integration-workflow` 使用。

**核心问题**：同一个 `command_id`（如主动上报帧），不同表型（膜式表/超声波表/预结算仪表）返回的字段集不同。如何让 codec 和 commands.yaml 正确处理这种多态？

---

## 1. 什么是数据域多态

表具行业协议中，数据域多态指：

> **同一条命令的响应数据域，其字段的存在性、结构和语义，由设备的"决定性参数"决定。**

最典型的决定性参数是**表具规格（表型）**。例如：

| 场景 | 表型 | 差异字段 |
|------|------|---------|
| 膜式燃气表 (G2.5/G10) | `0` | 有"累计用气量_标况"，无"工况累积量"、无"介质温度"、无"介质压力"、无"声速" |
| 超声波燃气表/流量计 | `1` | 有"工况累积量"、"介质温度"、"介质压力"、"声速" |
| 预结算仪表 | `2` | 额外有"剩余金额"、"累计金额"、"当前单价" |

设备标识（IMEI/表号）也隐含表型信息，但**表具规格**是协议文档中明确独立定义的决定性字段。

---

## 2. 核心模式：决定性参数 → 会话状态 → 条件解析

```
注册帧 / 首帧上报
  │
  ├─ 1. codec.unpack() 从帧中提取 表型参数
  │     （如 0x01=膜式, 0x02=超声波, 0x03=预结算）
  │
  ├─ 2. 写入 session_updates
  │     return UnpackResult(
  │         ...,
  │         session_updates={"meter_type": "0x01"},
  │     )
  │
  ▼
session.state 持久化: {"meter_type": "0x01", ...}

后续每次 unpack()
  │
  ├─ 3. 读取 session["meter_type"]
  │     meter_type = int(session.get("meter_type", 0))
  │
  ├─ 4. 按表型决定 decoded_fields 构建逻辑
  │     decoded = _build_base_fields(payload)
  │     if meter_type == 0x02:  # 超声波表
  │         decoded.update(_build_ultrasonic_fields(payload))
  │     if meter_type == 0x03:  # 预结算
  │         decoded.update(_build_prepaid_fields(payload))
  │
  └─ 5. 仅返回实际出现的字段
        return UnpackResult(
            command_id=did, payload=data,
            decoded_fields=decoded,
            decoded_fields_mode="merge",  # ← merge 模式
        )
```

**关键原则**：
- `decoded_fields_mode` 使用 `"merge"`：commands.yaml 定义全量字段元数据（type/unit/scale/description），codec 只填入实际出现的字段
- codec 只返回**实际出现了的**字段，不返回空值占位
- 字段的 type/length/unit/scale 由 commands.yaml 提供（GUI 展示用的元数据），codec 只负责提供值

---

## 3. commands.yaml 的多态字段定义约定

### 3.1 全量列出原则

为同一个命令的 response 定义**所有可能出现的**字段，无论适用哪种表型。

```yaml
commands:
  数据上报:
    description: 设备主动上报。不同表型上报的字段集不同。
    response:
      command_id: "0x0606"
      fields:
        # ===== 所有表型通用的字段 =====
        - name: 表具规格          # ← 决定性参数本身也必须出现在字段列表中
          type: enum
          length: 1
          description: "Tag 0x01 ID=8. 决定性参数——后续字段的多态取决于此值"
          enum_values:
            "0": "G2.5"
            "1": "G10"
            "2": "G16"
            "3": "G100"
            "4": "G160"

        - name: 累计用气量_标况
          type: uint32
          length: 4
          unit: m³
          scale: 0.01
          description: "Tag 0x02 ID=1. 通用字段，所有表型均有"

        - name: 阀门状态
          type: enum
          length: 1
          description: "Tag 0x02 ID=11. 通用字段"
          enum_values: {"0": "开", "1": "关", "2": "异常"}

        # ===== 超声波表/流量计特有字段 =====
        - name: 工况累积量
          type: uint32
          length: 4
          unit: m³
          scale: 0.01
          description: "Tag 0x02 ID=14. ⚠️ 仅超声波表/流量计"

        - name: 介质温度
          type: int16
          length: 2
          unit: "℃"
          scale: 0.1
          description: "Tag 0x02 ID=19. ⚠️ 仅超声波表/流量计，有符号"

        - name: 声速
          type: uint16
          length: 2
          unit: m/s
          description: "Tag 0x02 ID=22. ⚠️ 仅超声波表"

        # ===== 预结算仪表特有字段 =====
        - name: 剩余金额
          type: uint32
          length: 4
          unit: 元
          scale: 0.01
          description: "Tag 0x02 ID=3. ⚠️ 仅预结算功能仪表"

        - name: 累计金额
          type: uint32
          length: 4
          unit: 元
          scale: 0.01
          description: "Tag 0x02 ID=4. ⚠️ 仅预结算功能仪表"
```

### 3.2 description 标注约定

每个**多态相关**的字段，其 `description` 必须标注：

| 标注方式 | 示例 | 适用场景 |
|---------|------|---------|
| `⚠️ 仅<表型描述>` | `⚠️ 仅超声波表/流量计` | 字段仅特定表型出现 |
| `⚠️ <表型描述> 不适用` | `⚠️ 膜式表不适用` | 字段特定表型不出现（逆向标注） |
| `决定性参数` | `决定性参数——后续字段的多态取决于此值` | 标注哪个字段是表型来源 |
| `⚡ 多态字段集源` | `⚡ 字段集随表型变化。超声波表追加: 工况累积量, 介质温度, 声速` | 在命令级 description 中做整体说明 |

### 3.3 命令级 description 的多态总览

在命令级别的 `description` 中，给出该命令的多态字段一览：

```yaml
commands:
  数据上报:
    description: >
      设备主动上报实时/累计/告警数据。

      字段集随表型变化：
      - 所有表型：表具规格、累计量_标况、阀门状态、表具状态字、事件记录
      - 超声波/流量计 追加：工况累积量、介质温度、介质压力、声速、压力传感器状态
      - 预结算仪表 追加：剩余金额、累计金额、当前单价、累计购气次数
    response:
      ...
```

---

## 4. Codec 实现模式

### 4.1 注册帧中提取表型（建立会话状态）

```python
def unpack(self, data: bytes, session: dict) -> UnpackResult:
    result = _parse_frame(data, session)
    did = result["did"]
    payload = result["raw_payload"]

    # ===== 注册帧 / 首帧数据上报：提取表型写入 session =====
    session_updates = {}
    if did == 0x3001:  # 注册帧 DID
        meter_type = self._extract_meter_type(payload)
        session_updates["meter_type"] = meter_type

        # 也可以同步提取其他影响解析的参数
        session_updates["random_code"] = self._extract_random_code(payload)

    # ===== 主动上报帧：根据 session 中的表型条件解析 =====
    decoded_fields = None
    decoded_mode = "schema"

    if did == 0x0606:  # 数据上报
        decoded_fields = self._parse_report_by_type(payload, session)
        decoded_mode = "merge"

    return UnpackResult(
        command_id=did,
        payload=payload,
        session_updates=session_updates,
        decoded_fields=decoded_fields,
        decoded_fields_mode=decoded_mode,
    )

def _extract_meter_type(self, payload: bytes) -> int:
    """从注册帧 payload 中提取表型（偏移量和编码方式按协议文档确定）。"""
    # 偏移量和编码方式取决于具体协议
    if len(payload) < 1:
        return 0
    return payload[0] & 0xFF  # 示例：第一个字节 = 表型
```

### 4.2 条件字段构建（多态解析核心）

```python
def _parse_report_by_type(self, payload: bytes, session: dict) -> dict:
    """根据 session 中的表型，解析主动上报帧的数据域。

    原则：只返回当前表型下实际出现的字段。不返回空值占位。
    """
    meter_type = int(session.get("meter_type", 0))

    # ===== 所有表型通用的字段（无条件解析） =====
    decoded: dict[str, Any] = {
        "表具规格": meter_type,
        "累计用气量_标况": self._read_uint32(payload, offset=2),
        "阀门状态": self._read_uint8(payload, offset=6),
    }

    # ===== 超声波表/流量计特有字段 =====
    if meter_type in (0x02, 0x03, 0x04):  # G16, G100, G160 = 超声波系
        decoded.update({
            "工况累积量": self._read_uint32(payload, offset=10),
            "介质温度": self._read_int16(payload, offset=14),
            "声速": self._read_uint16(payload, offset=18),
        })

    # ===== 预结算仪表特有字段 =====
    if self._has_prepaid_feature(payload):
        decoded.update({
            "剩余金额": self._read_uint32(payload, offset=20),
            "累计金额": self._read_uint32(payload, offset=24),
        })

    return decoded
```

**关键约束**：
- `decoded_fields` 中的 key 名**必须与 `commands.yaml` 的 `fields[].name` 完全一致**，否则 `merge` 模式无法匹配元数据
- codec 只返回实际出现的字段；commands.yaml 中有但 codec 未返回的字段不会被展示（这正是我们想要的）
- 不要在 codec 中重复定义 type/length/unit/scale——这些属于 commands.yaml 的职责

### 4.3 何时用 `merge`，何时用 `replace`

| 模式 | 何时使用 | 原因 |
|------|---------|------|
| `merge`（推荐） | 字段元数据（type/unit/scale/enum_values）可由 commands.yaml schema 提供 | GUI 能得到完整的展示格式信息 |
| `replace` | 字段多态极其复杂（>30 种字段组合），或字段结构本身也随表型变化 | 维护 schema 的成本超过收益 |

**判断方法**：如果你的多态仅仅是"字段有无"的差异（字段本身的结构不变），用 `merge`；如果"字段的结构也随表型变化"（如同一 Tag ID 在不同表型下长度不同），考虑 `replace`。

---

## 5. 多态的完整生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 1: 参数配置（软件启动 / 协议切换时）                            │
│                                                                     │
│   config.yaml 中可预置表型默认值（可选）:                              │
│     parameters:                                                     │
│       default_meter_type:                                           │
│         label: "默认表型"                                            │
│         default: "0"          # 0=自动从设备读取                     │
│         persistent: true                                            │
│                                                                     │
│   session:                                                          │
│     meter_type: ""            # 运行时从设备读取后填入                │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│ Phase 2: 表型获取（首次通信）                                         │
│                                                                     │
│   发送注册帧 / 读取表型命令                                           │
│     → codec.unpack() 提取 meter_type                                │
│     → session_updates["meter_type"] = 0x02                          │
│     → ProtocolSession 持久化                                         │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│ Phase 3: 多态解析（后续所有通信）                                      │
│                                                                     │
│   每次 unpack() 被调用:                                               │
│     → meter_type = int(session.get("meter_type", 0))                │
│     → 按 meter_type 决定解析哪些字段                                  │
│     → decoded_fields_mode = "merge"                                  │
│     → 框架 merge: commands.yaml 元数据 + codec 的条件字段值           │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│ Phase 4: 异常处理                                                     │
│                                                                     │
│   - 表型未知: session["meter_type"] 为未定义 → 解析通用字段，         │
│     记录 warning 日志，不中断解析                                     │
│   - 表型变化: 设备被更换 → 下次注册帧会更新 session_updates           │
│   - commands.yaml 中未定义的字段: codec 的 decoded_fields 中出现      │
│     了 schema 中没有的 key → 框架仍会展示（无元数据，显示原始值）       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. 表型之外的其他多态决定参数

表型是最常见的多态决定参数，但表具协议中也可能存在其他决定性参数：

| 决定性参数 | 影响范围 | 获取方式 | session key 示例 |
|-----------|---------|---------|-----------------|
| 表具规格 (meter_type) | 数据上报字段集、参数定义集 | 注册帧 / 首帧上报 | `meter_type` |
| 协议版本 (protocol_version) | 新版本新增字段或修改字段长度 | 注册帧 / 读版本命令 | `protocol_version` |
| 结算模式 (billing_mode) | 是否启用预结算相关字段 | 参数读取 / 注册帧标志位 | `billing_mode` |
| 通信模式 (comm_mode) | 影响上报周期和数据密度 | 注册帧 / 配置参数 | `comm_mode` |
| 安装地区 (region_code) | 地区性参数差异 | 配置参数 / 注册帧 | `region_code` |

**共同模式**：
1. 决定性参数 → 写入 `session_updates`
2. 后续 `unpack()` 中从 `session` 读取 → 条件化字段构建
3. `commands.yaml` 中全量列出所有可能字段 + description 标注适用范围

---

## 7. 验证检查清单

- [ ] 已识别协议中哪些命令的数据域存在多态（不同表型/版本返回不同字段集）
- [ ] 已确定每个多态命令的**决定性参数**是什么（通常是表型）
- [ ] 决定性参数已在 config.yaml 的 `session` 段声明初始值
- [ ] 注册帧/首帧上报的 `unpack()` 中提取决定性参数并写入 `session_updates`
- [ ] `commands.yaml` 中全量列出了所有可能出现的字段
- [ ] 每个多态相关字段的 `description` 标注了 `⚠️ 仅<表型>` 或 `⚠️ <表型>不适用`
- [ ] 命令级 `description` 中给出了多态字段一览
- [ ] codec 的 `decoded_fields` key 名与 `commands.yaml` 的 `fields[].name` 完全一致
- [ ] codec 使用 `decoded_fields_mode: "merge"`（字段元数据由 schema 提供）
- [ ] codec 只返回当前表型下实际出现的字段（不返回空值占位）
- [ ] 未知表型时有容错处理（解析通用字段 + warning 日志）

---

## 8. session_updates 的双重角色：表型与安全密钥

`session_updates` 是 codec 向会话状态写入数据的**唯一通道**。在一个典型的表具协议中，
它同时承载两个不同性质的负载：

| 用途 | 示例 key | 来源 | 消费方 | 生命周期 |
|------|---------|------|--------|---------|
| **表型信息**（本文档主题） | `meter_type` | 注册帧/首帧上报 | 后续所有 `unpack()` | 跨整个会话，变更是异常事件（换表） |
| **安全密钥材料**（SKILL.md §3） | `random_code`, `enc_key`, `mac_key` | 注册帧随机码 | 后续所有 `pack()` 和 `unpack()` 的加解密 | 跨整个会话，密钥协商后稳定 |
| **序列号/计数器** | `seq`, `mid` | 首帧或手动配置 | 后续所有 `pack()` | 跨整个会话，每次 pack 递增 |

### 8.1 共存机制

同一个 `unpack()` 可以同时返回多种更新：

```python
return UnpackResult(
    command_id=did,
    payload=payload,
    session_updates={
        "meter_type": 0x02,                          # 表型（多态）
        "random_code": payload[54:70].hex().upper(), # 随机码（安全）
    },
    decoded_fields=decoded,
    decoded_fields_mode="merge",
)
```

`ProtocolChannel._on_raw_data()` 对 `session_updates` 做的是 **dict.update()**——
所有 key 合并写入，不区分用途。这意味着：
- 写入顺序由帧到达顺序决定（通常是注册帧先到）
- 同一个 key 多次写入会覆盖（以最后一次为准）

### 8.2 降级策略

如果决定性参数获取失败（注册帧未到达、解析失败或设备不支持），`session_updates` 中
the 对应 key 不会被写入。后续解包时 `session.get("meter_type", 0)` 返回默认值。

**推荐降级行为**：
- `meter_type` 未知 → 仅解析所有表型通用的字段，对表型特有字段不做解析
- 每帧记录一次 `warning` 日志（不要每帧都打，用计数器控制频率）
- 允许用户通过 GUI 参数面板手动设置 `meter_type` 作为 fallback

```python
def unpack(self, data: bytes, session: dict) -> UnpackResult:
    meter_type = int(session.get("meter_type", 0))
    if meter_type == 0 and self._warn_count < 3:
        logger.warning("meter_type unknown — 仅解析通用字段。请先执行注册帧或手动设置表型。")
        self._warn_count += 1

    decoded = self._build_base_fields(payload)
    if meter_type in (0x02, 0x03):
        decoded.update(self._build_ultrasonic_fields(payload))
    return UnpackResult(
        command_id=did, payload=payload,
        decoded_fields=decoded,
        decoded_fields_mode="merge",
    )
```

### 8.3 与安全密钥的交互

表型信息和安全密钥走同一条通道（`session_updates` → `session.state`），
但它们之间**没有时序依赖**。无论表型先到还是密钥先到，两者分别被不同消费方读取：

```
注册帧到达
  │
  ├─ unpack() 提取 random_code  → session_updates["random_code"]
  ├─ unpack() 提取 meter_type   → session_updates["meter_type"]
  │
  ▼
  session.state = {"random_code": "...", "meter_type": 0x02}
  │
  ├─ 后续 pack():  派生 enc_key = HMAC(master_key, session["random_code"])
  ├─ 后续 unpack(): meter_type = session.get("meter_type") → 条件解析字段
  │
  └─ 两者互不阻塞、互不依赖
