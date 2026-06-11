# 协议接入技术扩展参考

> 📏 ~1,200 tokens | 必读等级: ★★☆（Phase 2 对照用）| 前置: SKILL.md §2, §3
> ⏩ 同构拷贝 → 读 §B；全新开发 → 读 §A；排错 → 读 §E

配合 `protocol-integration-workflow` skill 使用。

本文档仅包含**代码级细节**。架构原理、安全模式、同构决策树等已在 SKILL.md 中完整覆盖：
- **PackContext / UnpackResult / decoded_fields_mode** → SKILL.md §2
- **安全层三种模式（加密管线、密钥派生）** → SKILL.md §3
- **同构 vs 异构决策树** → SKILL.md §7
- **参数模型与 UI 契约** → SKILL.md §5-§6

---

## A. Codec 分步实现指南

从零开始实现一个 codec，按复杂度递增分四步。每步完成后再进入下一步，每步可通过契约测试独立验证。

### A.1 第 1 步：定长帧（无校验、无安全层）

**目标**：验证帧格式理解正确，payload 字节级往返一致。

```python
from app.infrastructure.protocol.abc import PackContext, ProtocolCodec, UnpackResult
from app.infrastructure.protocol.exceptions import ProtocolPackError, ProtocolUnpackError

HEAD = b"\x68"
TAIL = b"\x16"

def _calc_xor(data: bytes) -> int:
    result = 0
    for b in data:
        result ^= b
    return result & 0xFF

class Codec(ProtocolCodec):
    def on_load(self, config: dict) -> None:
        pass  # 第 1 步不需要任何配置

    def pack(self, ctx: PackContext) -> bytes:
        # 帧结构: HEAD + LEN(1B) + CMD(2B) + PAYLOAD + CS(1B, XOR) + TAIL
        cmd_bytes = ctx.command_id.to_bytes(2, "big")
        payload = ctx.payload
        length = 2 + len(payload)  # CMD + PAYLOAD 的字节数
        body = bytes([length]) + cmd_bytes + payload
        cs = _calc_xor(body)
        return HEAD + body + bytes([cs]) + TAIL

    def unpack(self, data: bytes, session: dict) -> UnpackResult:
        if len(data) < 6 or data[0:1] != HEAD or data[-1:] != TAIL:
            raise ProtocolUnpackError("invalid frame")
        length = data[1]
        body = data[2:-2]  # 去掉 HEAD, CS, TAIL
        if _calc_xor(body) != data[-2]:
            raise ProtocolUnpackError("CS mismatch")
        cmd_id = int.from_bytes(body[1:3], "big")
        payload = body[3:]
        return UnpackResult(command_id=cmd_id, payload=payload)
```

**验证点**：
- `pack → unpack` 往返后 payload 不变
- 帧长度精确等于 HEAD + 1 + 2 + len(payload) + 1 + TAIL
- 用协议文档的 hex dump 测试 pack 输出逐字节匹配

### A.2 第 2 步：加 CRC

**目标**：在第 1 步基础上，将简单 XOR 校验替换为标准 CRC。

```python
# 追加到 codec.py
from crccheck.crc import Crc16Modbus  # 或其他 CRC 算法

def _calc_crc16(data: bytes) -> bytes:
    crc = Crc16Modbus.calc(data)
    return crc.to_bytes(2, "big")
```

修改 `pack()` 中 CS 计算为 `_calc_crc16(body)`，修改 `unpack()` 中的 CS 校验逻辑。

**CRC 算法选择**：如果协议文档未明确 CRC 参数（多项式、初始值），
可用穷举法确定（详见 `references/test-vector-extraction.md` §4）。

### A.3 第 3 步：加安全层

**目标**：在帧结构中嵌入加密和 MAC。

此时代码量显著增加。建议将安全逻辑抽到独立模块（如 `frame_security.py`），
codec.py 只做编排。详见 SKILL.md §3 的三种模式选择，以及本文档 §C 的同构拷贝方案
（如果安全层与已有协议一致，直接拷贝后修改）。

### A.4 第 4 步：加 session_updates 和 decoded_fields

**目标**：处理密钥协商、表型提取、TLV 解析等需要会话状态的逻辑。

```python
def unpack(self, data: bytes, session: dict) -> UnpackResult:
    result = self._parse_frame(data, session)
    did = result["did"]
    payload = result["raw_payload"]

    # ===== 密钥协商：从注册帧提取随机码 =====
    updates = {}
    if did == 0x3001:
        updates["random_code"] = payload[54:70].hex().upper()

    # ===== 数据域多态：从注册帧/首帧提取表型 =====
    if did == 0x0606:  # 数据上报
        meter_type = self._extract_meter_type(payload)
        updates["meter_type"] = meter_type

        # 按表型条件解析（详见 data-domain-polymorphism.md）
        decoded = self._parse_by_type(payload, meter_type)
        return UnpackResult(
            command_id=did, payload=payload,
            session_updates=updates,
            decoded_fields=decoded,
            decoded_fields_mode="merge",
        )

    return UnpackResult(command_id=did, payload=payload, session_updates=updates)
```

**关键原则**：
- `session_updates` 只写**需要跨帧持久化的数据**（密钥、表型、序列号）。
  不要把单次解析的临时值放进去——它们会让 session 膨胀且不可预期
- 如果同一个 `unpack()` 中同时更新密钥协商信息和表型信息，
  两者通过 `session_updates` 的 dict 合并自然共存，接收顺序由帧到达顺序决定
- 如果表型获取失败（注册帧未收到或解析失败），`meter_type` 在 session 中为空/默认值，
  后续帧解包时应*降级处理*（仅解析通用字段，日志记录 warning）

---

## B. 同构拷贝：完整 codec 示例

当新协议的帧结构/安全层与已有协议一致时（详见 SKILL.md §7 决策树判断），
从已有协议**拷贝**底层文件后修改。**禁止跨协议 import**。

以下是拷贝后 codec.py 的完整参考实现：

```python
# plugins/jk_100_protocol/codec.py
from app.core.protocol.abc import PackContext, ProtocolCodec, UnpackResult
from .frame_security import FrameCodec  # 拷贝后修改过的独立副本


class Codec(ProtocolCodec):
    def on_load(self, config):
        FrameCodec.configure(config)

    def pack(self, ctx):
        did = int(ctx.command_id) & 0xFFFF
        c = self._resolve_c(did, ctx.is_uplink, ctx.overrides)
        return FrameCodec.build({
            "command_id": ((c & 0xFF) << 16) | did,
            "raw_payload": ctx.payload,
            "is_uplink": ctx.is_uplink,
            "session_vars": ctx.session,
        })

    def unpack(self, data, session):
        result = FrameCodec.parse(data, context={"session_vars": session})
        func, did = result["c"] & 0x1F, result["did"]

        if func == 0x08 and result.get("is_uplink"):
            return UnpackResult(
                command_id=did, payload=result["raw_payload"],
                decoded_fields=self._parse_write_read(result["raw_payload"]),
                decoded_fields_mode="replace")

        updates = {}
        if did == 0x3001 and func == 0x09:
            updates["random_000c"] = result["raw_payload"][54:70].hex().upper()

        return UnpackResult(command_id=did, payload=result["raw_payload"],
                           session_updates=updates)
```

**拷贝后必须修改**：类名、帧头/帧尾、DID映射表、功能码映射、随机码偏移、MAC长度、默认安全模式。

**独立性验证**（将 codec.py 放在同目录运行）：
```python
import ast, sys
with open("plugins/new_protocol/codec.py") as f:
    tree = ast.parse(f.read())
for node in ast.walk(tree):
    if isinstance(node, ast.ImportFrom):
        if "plugins" in node.module and "new_protocol" not in node.module:
            print(f"违规跨协议import: {node.module}")
```

---

## C. 异常路径处理

### C.1 codec 层

- pack 失败 → 抛 `ProtocolPackError`
- CRC 失败 → 抛 `ProtocolUnpackError`
- MAC 失败 → 抛 `ProtocolAuthError`

### C.2 Channel 层传播

unpack 异常 → warning/error 日志 → error_callbacks → **继续**，永不中断接收循环。

### C.3 非致命容错

`_allow_unverified_parse=True`：MAC 失败仍返回 payload，标记 `mac_verified: false`。

---

## D. 测试策略

### D.1 单元测试

`test_pack_minimal`, `test_unpack_valid`, `test_unpack_crc_error`, `test_unpack_mac_error`。

### D.2 集成测试

`MockTransport + ProtocolChannel.create + send + simulate_receive`。

### D.3 独立性测试

用 ast 解析 codec.py 的 import，验证不引用其他 `plugins/*` 模块。

---

## E. 故障排查

| 症状 | 原因 | 排查 |
|------|------|------|
| GUI 无协议 | config.yaml 缺失 | 检查 plugins/ 目录 |
| 命令为空 | commands.yaml 路径错 | 显示名 = 目录名 |
| 帧长度错 | field length 不匹配 | 逐字段验证 |
| 解析乱码 | 密钥未进 session | 检查密钥路径 |
| CRC 失败 | 算法或范围错 | 对照文档 |
| seq 不递增 | session_updates 缺失 | unpack 返回 |
| override 无效 | 未标记 codec_override | 检查指令属性 |
| 跨协议 import | 引用其他插件 | 拷贝 + 相对导入 |
