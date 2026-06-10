# 测试向量提取与契约测试生成

配合 `protocol-integration-workflow` 和 `document-analysis-workflow.md` 使用。
本文档解决核心问题：**在没有真实硬件的条件下，如何验证组帧（pack）和解包（unpack）的字节级正确性。**

---

## 目录

1. [测试向量的定位与提取](#1-测试向量的定位与提取)
2. [测试向量结构化记录](#2-测试向量结构化记录)
3. [契约测试生成模板](#3-契约测试生成模板)
4. [CRC 参数反推技术](#4-crc-参数反推技术)
5. [往返测试（Round-Trip Test）](#5-往返测试round-trip-test)
6. [错误帧测试](#6-错误帧测试)
7. [无测试向量时的替代方案](#7-无测试向量时的替代方案)

---

## 1. 测试向量的定位与提取

### 1.1 在协议文档中搜索

**高优先级搜索关键词**（按命中率排序）：
- `示例` `举例` `例如` `Example` `Sample`
- hex dump 正则：`[0-9A-Fa-f]{16,}` （连续 16 个以上 hex 字符）
- 帧字段解析说明：`SOF=` `HEAD=` `CMD=` `TYPE=` `SEQ=` `LEN=` `CRC=`
- 附录中的 `报文示例` `通信示例` `数据帧示例` 章节

**搜索策略**：
1. 先用正则 `[0-9A-Fa-f]{20,}` 全局搜索，标记所有 hex dump 位置
2. 检查 hex dump 前后是否有字段解析说明
3. 如果只有 hex dump 没有解析说明 → 按帧格式模板自行解析
4. 如果只有解析说明没有 hex dump → 按解析说明反推 hex dump

### 1.2 从 hex dump 提取

```python
import re

def extract_hex_dumps(text: str) -> list[dict]:
    """从协议文档文本中提取所有 hex dump 及其上下文。"""
    results = []

    # 匹配连续的 hex 字符串（忽略空格和换行）
    pattern = re.compile(r'(?:[0-9A-Fa-f]{2}\s*){8,}')
    for match in pattern.finditer(text):
        hex_str = re.sub(r'\s+', '', match.group())

        # 取前后 500 个字符作为上下文
        start = max(0, match.start() - 500)
        end = min(len(text), match.end() + 500)
        context = text[start:end]

        results.append({
            "hex": hex_str,
            "context": context,
            "position": match.start(),
        })

    return results

# 使用示例
full_text = Path("protocol_extracted.txt").read_text(encoding="utf-8")
dumps = extract_hex_dumps(full_text)
for i, d in enumerate(dumps):
    print(f"--- Hex Dump #{i+1} ---")
    print(f"Hex: {d['hex']}")
    print(f"Context: ...{d['context'][:200]}...")
    print()
```

### 1.3 测试向量的优先级排序

提取到所有 hex dump 后，按以下优先级选择要纳入测试的向量：

| 优先级 | 向量特征 | 最少数量 | 原因 |
|--------|---------|---------|------|
| **P0 必须** | 文档有完整 hex dump + 逐字段解析说明 | 全部 | 这是唯一可验证的"基准真值" |
| **P0 必须** | 简单请求帧（无 payload 或 payload=最小） | ≥1 | 验证帧头/帧尾/CRC 的基准 |
| **P0 必须** | 带 payload 的请求帧 | ≥1 | 验证字段序列化 |
| **P0 必须** | 响应帧 | ≥1 | 验证解析逻辑 |
| **P1 推荐** | 错误响应帧（result_code ≠ 0x0000） | ≥1 | 验证错误码提取 |
| **P1 推荐** | 加密帧（如适用） | ≥1 | 验证安全层 |
| **P2 可选** | 有 payload 的复杂帧 | 尽可能 | 提高覆盖率 |

---

## 2. 测试向量结构化记录

从文档中提取的测试向量，必须按以下模板记录（以 HIL 协议为例）：

```markdown
### 向量编号: TV-001
### 名称: GET_INFO_REQ
### 方向: PC → 板卡 (下行)

### Hex Dump:
55AA010100010001000062D3

### 已知上下文（从文档提取）:
- SOF=55AA, VER=01, TYPE=01(REQ)
- SEQ=0001, CMD=0001(GET_INFO)
- LEN=0000 (无 payload)
- CRC16=62D3

### 预期 pack 验证点:
- 帧头: bytes 0-1 = 55AA
- VER:   byte 2 = 01
- TYPE:  byte 3 = 01
- SEQ:   bytes 4-5 = 0001
- CMD:   bytes 6-7 = 0001
- LEN:   bytes 8-9 = 0000
- CRC:   bytes 10-11 = 62D3
- 帧总长: 12 bytes

### 预期 unpack 验证点:
- command_id = 0x0001
- payload = b"" (空)
- is_uplink = False
```

### 2.1 当文档只有 hex dump 没有解析说明时

按帧格式模板逐字节手动解析，标注 `[推测]`：

```markdown
### 向量编号: TV-005
### 名称: 未知请求帧 [推测: 可能是设置参数命令]
### 来源: 协议文档第 15 页示例

### Hex Dump:
7A72191A069999999999990000000000000101F358EA69AFC41EF0F0AA

### 手动解析（按帧格式模板）:
- 帧头 7A72 (2 bytes)
- VER 19 (推测版本号)
- CMD 1A06 (command_id 推测 = 0x1A06)
- 地址 999999999999000000000000 (12 bytes)
- 数据域 01 (1 byte)
- 安全相关 F358EA69AFC41EF0 (8 bytes) [推测: MAC或加密数据]
- 帧尾 F0AA (2 bytes)

### 需要与固件方确认:
- [ ] CMD=0x1A06 的命令名是什么？
- [ ] 12 字节地址字段是否确实在 VER 之后？
- [ ] 8 字节安全域是 MAC 还是加密数据？
```

---

## 3. 契约测试生成模板

### 3.1 最小 pack 契约测试

参考项目中的 `tests/test_jk_protocol_codec.py` 的模式：

```python
"""<协议名> codec 契约测试 — 基于协议文档测试向量。

测试向量来源: <协议文档名称> 第 <X> 页
"""

from pathlib import Path
import yaml
import pytest

from app.infrastructure.protocol.abc import PackContext
from app.infrastructure.validators.formatters.binary import BinaryFormatter
from plugins.<protocol_key>.codec import Codec


# ===== 共享 fixture =====

@pytest.fixture(scope="module")
def codec():
    """加载协议配置并初始化 Codec。"""
    config = yaml.safe_load(
        Path("plugins/<protocol_key>/config.yaml").read_text(encoding="utf-8")
    ) or {}
    c = Codec()
    c.on_load(config)
    return c


# ===== Pack 契约测试 =====

def test_pack_get_info_req_matches_document_hex(codec):
    """TV-001: GET_INFO 请求帧必须与协议文档 hex dump 逐字节一致。"""
    payload = BinaryFormatter.format_fields(
        {},  # 无字段
        [],  # 无字段定义
    )

    frame = codec.pack(PackContext(
        command_id=0x0001,
        payload=payload,
        is_uplink=False,
        session={},
    ))

    expected = bytes.fromhex(
        "55AA010100010001000062D3"
    )
    assert frame == expected, (
        f"\n实际: {frame.hex().upper()}"
        f"\n期望: {expected.hex().upper()}"
    )


def test_pack_gpio_write_ch1_high_matches_document_hex(codec):
    """TV-004: GPIO_WRITE channel=1 level=1 请求帧必须与文档一致。"""
    payload = BinaryFormatter.format_fields(
        {"通道": 1, "电平": 1},
        [
            {"name": "通道", "type": "uint8", "length": "1"},
            {"name": "电平", "type": "uint8", "length": "1"},
        ],
    )

    frame = codec.pack(PackContext(
        command_id=0x0101,
        payload=payload,
        is_uplink=False,
        session={},
    ))

    expected = bytes.fromhex(
        "55AA01010010010100020101119C"
    )
    assert frame == expected


# ===== Unpack 契约测试 =====

def test_unpack_get_info_rsp_parses_correctly(codec):
    """TV-003: GET_INFO 响应帧解析后字段值必须与文档一致。"""
    raw_frame = bytes.fromhex(
        "55AA010200010001001A00000001"
        "0101312E302E302D646576000000"
        "5354333247340000000000000000"
        "0003F93F678C4E12XXXX"
    )
    # 注意：上面是示意，实际 CRC 需正确

    session_snapshot = {}
    result = codec.unpack(raw_frame, session_snapshot)

    assert result.command_id == 0x0001
    # payload 是解密后的业务字节，这里验证关键字段
    # 具体验证取决于 codec 的 decoded_fields_mode
```

### 3.2 参数化批量测试

当有多条测试向量时，用 `pytest.mark.parametrize` 批量测试：

```python
TEST_VECTORS_PACK = [
    # (名称, command_id, field_data, field_defs, session, expected_hex)
    (
        "GET_INFO_REQ",
        0x0001,
        {},
        [],
        {},
        "55AA010100010001000062D3",
    ),
    (
        "GPIO_WRITE_CH1_HIGH",
        0x0101,
        {"通道": 1, "电平": 1},
        [
            {"name": "通道", "type": "uint8", "length": "1"},
            {"name": "电平", "type": "uint8", "length": "1"},
        ],
        {},
        "55AA01010010010100020101119C",
    ),
    # ... 更多向量
]


@pytest.mark.parametrize(
    "name,cmd_id,field_data,field_defs,session,expected_hex",
    TEST_VECTORS_PACK,
)
def test_pack_vectors(name, cmd_id, field_data, field_defs, session, expected_hex, codec):
    """所有 pack 测试向量逐字节验证。"""
    payload = BinaryFormatter.format_fields(field_data, field_defs)
    frame = codec.pack(PackContext(
        command_id=cmd_id,
        payload=payload,
        is_uplink=False,
        session=session,
    ))
    expected = bytes.fromhex(expected_hex)
    assert frame == expected, f"[{name}] FAILED"
```

### 3.3 调试辅助：帧字节逐字段对比

当测试失败时，快速定位差异：

```python
def diff_frames(actual: bytes, expected: bytes, frame_layout: list[tuple[str, int, int]]):
    """逐字段对比两帧，定位差异。

    frame_layout: [(字段名, start_offset, length), ...]
    """
    print(f"{'Field':<20} {'Offset':>6} {'Len':>4} {'Expected':<24} {'Actual':<24} {'Match'}")
    print("-" * 90)

    all_match = True
    for name, offset, length in frame_layout:
        exp_slice = expected[offset:offset+length].hex().upper()
        act_slice = actual[offset:offset+length].hex().upper()
        match = "✓" if exp_slice == act_slice else "✗ ← MISMATCH"
        if exp_slice != act_slice:
            all_match = False
        print(f"{name:<20} {offset:>6} {length:>4} {exp_slice:<24} {act_slice:<24} {match}")

    return all_match


# 在测试中使用
def test_pack_with_diff(codec):
    frame_layout = [
        ("SOF",  0, 2),
        ("VER",  2, 1),
        ("TYPE", 3, 1),
        ("SEQ",  4, 2),
        ("CMD",  6, 2),
        ("LEN",  8, 2),
        ("PAYLOAD", 10, -1),  # -1 = 到 CRC 前
        ("CRC16", -2, 2),      # 最后 2 字节
    ]
    # ... pack 后调用 diff_frames(actual, expected, frame_layout)
```

---

## 4. CRC 参数反推技术

当协议文档未明确 CRC 参数时，利用多条测试向量反推。

### 4.1 穷举法

```python
"""
CRC 参数反推工具。
需要至少 2-3 条不同内容的测试向量（已知校验范围 hex + 帧尾 CRC 值）。
"""

import struct
from crccheck.crc import Crc16, Crc32, CrcXmodem, CrcCcitt


def brute_force_crc16(test_vectors: list[tuple[str, str]]):
    """逐对尝试常见 CRC16 参数，找到匹配所有向量的 CRC 配置。

    test_vectors: [(校验范围hex, 帧尾CRC hex), ...]
    """
    candidates = []

    configs = [
        ("CRC16-IBM",     Crc16(poly=0x8005, initvalue=0x0000, reflect_input=True,  reflect_output=True,  xor_output=0x0000)),
        ("CRC16-XMODEM",  Crc16(poly=0x1021, initvalue=0x0000, reflect_input=False, reflect_output=False, xor_output=0x0000)),
        ("CRC16-CCITT",   Crc16(poly=0x1021, initvalue=0xFFFF, reflect_input=False, reflect_output=False, xor_output=0x0000)),
        ("CRC16-CCITT-x", Crc16(poly=0x1021, initvalue=0x0000, reflect_input=True,  reflect_output=True,  xor_output=0x0000)),
        ("CRC16-DNP",     Crc16(poly=0x3D65, initvalue=0x0000, reflect_input=True,  reflect_output=True,  xor_output=0xFFFF)),
        ("CRC16-MODBUS",  Crc16(poly=0x8005, initvalue=0xFFFF, reflect_input=True,  reflect_output=True,  xor_output=0x0000)),
    ]

    for name, crc_impl in configs:
        all_match = True
        results = []
        for data_hex, expected_crc_hex in test_vectors:
            data = bytes.fromhex(data_hex)
            expected = bytes.fromhex(expected_crc_hex)

            crc_impl.reset()
            crc_impl.process(data)
            actual = struct.pack(">H", crc_impl.finalvalue())

            if actual != expected:
                all_match = False
            results.append(f"  {data_hex[:20]}... → 计算={actual.hex().upper()} 期望={expected.hex().upper()}")

        if all_match:
            candidates.append(f"✓ {name}: ALL MATCH\n" + "\n".join(results))

    return candidates


# 使用示例
vectors = [
    # GET_INFO_REQ: VER..PAYLOAD = 01000100010000, CRC=62D3
    ("01000100010000", "62D3"),
    # PING_REQ: VER..PAYLOAD = 010100020002000401020304, CRC=F9BD
    ("010100020002000401020304", "F9BD"),
]

for c in brute_force_crc16(vectors):
    print(c)
```

### 4.2 手工反推步进法

如果穷举法无匹配（可能是非标准 CRC），手工反推：

1. 计算校验范围内所有字节的 XOR → 与帧尾 CRC 对比，确认不是简单 XOR
2. 计算累加和取低 8/16 位 → 对比
3. 检查 CRC 字节序：帧尾 CRC 是 `[高字节 低字节]` 还是 `[低字节 高字节]`？
4. 检查校验范围：LEN 字段是否计入？CRC 自身是否计入？
5. 检查是否有字节需要先转义再 CRC？

---

## 5. 往返测试（Round-Trip Test）

往返测试是最强的正确性保证：pack 后 unpack，验证 payload 不变。

```python
def test_round_trip_pack_unpack(codec):
    """pack → unpack 往返后，command_id 和 payload 必须不变。"""
    payload = BinaryFormatter.format_fields(
        {"通道": 1, "电平": 1},
        [
            {"name": "通道", "type": "uint8", "length": "1"},
            {"name": "电平", "type": "uint8", "length": "1"},
        ],
    )
    session = {}

    # Pack
    frame = codec.pack(PackContext(
        command_id=0x0101,
        payload=payload,
        is_uplink=False,
        session=session,
    ))

    # Unpack
    result = codec.unpack(frame, session)

    assert result.command_id == 0x0101
    assert result.payload == payload, (
        f"payload 不一致!\n"
        f"  原始: {payload.hex().upper()}\n"
        f"  解析: {result.payload.hex().upper()}"
    )
```

### 5.1 带安全层的往返测试

如果协议有加密层，往返测试会自动覆盖加密/解密正确性：

```python
def test_round_trip_with_encryption(codec):
    """加密 pack → 解密 unpack 往返后 payload 必须不变。"""
    session = {"aes_key": "C83E7386FA4DB629C83E7386FA4DB629"}  # 256-bit key

    payload = bytes.fromhex("0102030405060708")
    frame = codec.pack(PackContext(
        command_id=0x1A06,
        payload=payload,
        is_uplink=False,
        session=session,
    ))

    # 验证：加密后的 frame 不应包含明文 payload
    assert payload not in frame, "payload 以明文形式出现在加密帧中！"

    result = codec.unpack(frame, session)
    assert result.payload == payload
```

---

## 6. 错误帧测试

验证 codec 对异常输入的正确处理：

```python
import pytest
from app.infrastructure.protocol.exceptions import ProtocolUnpackError

def test_unpack_crc_error_raises(codec):
    """CRC 校验失败的帧必须抛出 ProtocolUnpackError。"""
    # 取一条正常帧，修改最后一个字节破坏 CRC
    valid_frame = bytearray(bytes.fromhex(
        "55AA010100010001000062D3"
    ))
    valid_frame[-1] ^= 0xFF  # 翻转最后字节

    with pytest.raises(ProtocolUnpackError):
        codec.unpack(bytes(valid_frame), {})

def test_unpack_short_frame_raises(codec):
    """不完整的帧（长度不足）必须抛出异常。"""
    short_frame = bytes.fromhex("55AA01")  # 只有 3 字节

    with pytest.raises(ProtocolUnpackError):
        codec.unpack(short_frame, {})

def test_unpack_wrong_header_raises(codec):
    """帧头错误的帧必须抛出异常。"""
    wrong_header = bytes.fromhex(
        "FFFF010100010001000062D3"  # 帧头改为 FFFF
    )

    with pytest.raises(ProtocolUnpackError):
        codec.unpack(wrong_header, {})
```

---

## 7. 无测试向量时的替代方案

如果协议文档**完全没有**示例 hex dump，按以下优先级建立测试：

### 7.1 手工构造最小验证帧

```python
def test_manual_minimal_frame(codec):
    """手工构造并验证最小帧的结构正确性。"""
    # 构造无 payload 的简单请求
    payload = b""
    frame = codec.pack(PackContext(
        command_id=0x0001,
        payload=payload,
        is_uplink=False,
        session={},
    ))

    # 基本结构断言
    assert len(frame) >= 8, f"帧太短: {len(frame)} bytes"
    assert frame[:2] == b"\x55\xAA", f"帧头错误: {frame[:2].hex()}"
    # ... 按帧格式模板逐字段断言

    # 打印帧以便手工检查
    print(f"生成帧: {frame.hex().upper()}")
```

### 7.2 往返测试作为最低保障

即使没有基准 hex dump，往返测试至少能保证 pack/unpack 互逆：

```python
def test_all_commands_round_trip(codec, command_store):
    """所有定义过的命令都做往返测试。"""
    for cmd_name, cmd_def in command_store.all().items():
        # 构造默认值字段数据
        field_data = {}
        for field in cmd_def.get("request", {}).get("fields", []):
            field_data[field["name"]] = field.get("default", "0")

        payload = BinaryFormatter.format_fields(
            field_data,
            cmd_def["request"]["fields"],
        )
        cmd_id = int(cmd_def["request"]["command_id"], 16)

        # Pack + Unpack
        frame = codec.pack(PackContext(
            command_id=cmd_id,
            payload=payload,
            is_uplink=False,
            session={},
        ))
        result = codec.unpack(frame, {})

        assert result.command_id == cmd_id, f"[{cmd_name}] command_id 不匹配"
```

### 7.3 与固件开发同步获取测试向量

如果固件同步开发中：
1. 请求固件方提供 5-10 条典型帧的 hex dump（含字段解析）
2. 或者双方商定用 Python 脚本生成交叉验证用的测试向量

---

## 8. 测试文件放置规范

```
tests/
├── test_<protocol_key>_codec.py       # 契约测试（pack/unpack 字节验证）
├── test_<protocol_key>_channel.py     # 集成测试（MockTransport + Channel）
└── conftest.py                         # 共享 fixture（如需）
```

命名与 `plugins/<protocol_key>/` 对应，保持一一映射。
