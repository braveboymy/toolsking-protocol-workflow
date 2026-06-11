# Phase 3.5：协议一致性测试矩阵

> 📏 ~3,200 tokens | 必读等级: ★☆☆（送检/发布前需要，日常开发可选）| 前置: Phase 3 契约测试通过

配合 `protocol-integration-workflow` 和 `test-vector-extraction.md` 使用。

本文档解决核心问题：**在契约测试验证了“单条向量正确”之后，如何系统化地验证“整个协议实现正确”——覆盖全部命令、全部异常路径、全部安全模式组合、全部会话状态转换。**

---

## 目录

1. [一致性测试与契约测试的区别](#1-一致性测试与契约测试的区别)
2. [测试矩阵的两个维度](#2-测试矩阵的两个维度)
3. [六类测试详解](#3-六类测试详解)
4. [矩阵模板](#4-矩阵模板)
5. [从 commands.yaml 自动生成基础矩阵](#5-从-commandsyaml-自动生成基础矩阵)
6. [pytest 实现框架](#6-pytest-实现框架)
7. [与计量院检测项目的映射](#7-与计量院检测项目的映射)
8. [检查清单](#8-检查清单)

---

## 1. 一致性测试与契约测试的区别

| 维度 | 契约测试（Phase 3） | 一致性测试（Phase 3.5） |
|------|-------------------|----------------------|
| **目标** | 验证 codec 对**单条已知 hex dump** 的 pack/unpack 正确 | 验证协议实现对**全部命令空间**的正确性 |
| **输入来源** | 协议文档中的 hex dump 示例 | 从 `commands.yaml` 推导出的系统化用例 |
| **覆盖范围** | 文档中明确给出的示例（通常 < 10 条） | 全部命令 × 多维度测试类型（通常 100+ 条） |
| **验证重点** | 字节级精确匹配 | 行为级合规：字段解析、错误处理、状态机、边界值 |
| **执行频率** | 每次修改 codec 后必跑 | 发布前 / 送检前完整跑一遍 |

**关键认知**：契约测试确保“文档给出的例子我能跑对”；一致性测试确保“协议定义的全部命令空间我都能正确处理”。两者互补，不可互相替代。

---

## 2. 测试矩阵的两个维度

一致性测试矩阵是一个**命令 × 测试类型**的二维表：

### 2.1 命令维度（X 轴）

从 `commands.yaml` 的 `commands` 段提取全部命令，按功能域分组：

```
命令分组（表具行业典型）
├── 管理类（必测）
│   ├── 读取版本信息
│   ├── 读取设备地址
│   └── 设置设备地址
├── 计量类（必测，含金额/单价/余量）
│   ├── 读取累积流量
│   ├── 读取剩余金额
│   ├── 读取当前单价
│   └── 读取计量状态字
├── 控制类（必测）
│   ├── 阀门控制（开/关）
│   ├── 充值（金额/气量）
│   └── 调价
├── 参数类（必测）
│   ├── 读取工作参数
│   └── 设置工作参数
├── 事件类（必测）
│   ├── 事件记录查询
│   └── 事件主动上报
├── 安全类（必测）
│   ├── 注册/认证
│   ├── 密钥更新
│   └── 随机码获取
└── 维护类（选测）
    ├── 出厂初始化
    └── 生产测试命令
```

**优先级规则**：
- **P0（送检必过）**：管理类 + 计量类 + 控制类 + 安全类
- **P1（功能完整）**：参数类 + 事件类
- **P2（可选）**：维护类

### 2.2 测试类型维度（Y 轴）

| 类型编号 | 类型名称 | 优先级 | 说明 |
|---------|---------|--------|------|
| T1 | 正例覆盖 | P0 | 命令按标准字段定义 pack/unpack 正确 |
| T2 | 异常帧覆盖 | P1 | CRC 错 / MAC 错 / 帧头错 / 长度错 / 非法 command_id |
| T3 | 边界值覆盖 | P1 | 字段取最大值/最小值/临界值时的行为 |
| T4 | 安全模式排列 | P1 | 每条命令在不同 security_mode 下的组帧差异 |
| T5 | 会话状态机遍历 | P1 | 注册 → 认证 → 加密通信 → 密钥更新的状态转换 |
| T6 | 压力与稳定性 | P2 | 连续发送 1000 帧 / 粘包 / 乱序 / 超时 |

---

## 3. 六类测试详解

### 3.1 正例覆盖（T1）

**目标**：验证每条命令的 request 能正确 pack，response 能正确 unpack。

**测试策略**：
- 对 `commands.yaml` 中**每条**命令的 `request` 和 `response` 分别生成一个正例
- request 正例：用字段的 `default` 值填充，调用 `codec.pack()`，验证不抛异常且输出帧结构合规
- response 正例：构造符合帧格式的模拟响应字节流，调用 `codec.unpack()`，验证 `command_id` 正确且 `decoded_fields` 包含预期键

**最小验收标准**：
- P0 命令的 request + response 正例覆盖率 = 100%
- P1 命令的 request + response 正例覆盖率 = 100%

### 3.2 异常帧覆盖（T2）

**目标**：验证 codec 面对脏数据/恶意数据时的容错行为，确保不崩、不挂、不误判。

**异常帧构造模板**：

```python
# 从一条合法帧出发，系统性注入故障
FAULT_TEMPLATES = [
    # 帧结构级故障
    {"name": "wrong_sof",          "mutate": lambda f: b"\xDE\xAD" + f[2:]},
    {"name": "truncated_crc",      "mutate": lambda f: f[:-1]},
    {"name": "extra_tail",         "mutate": lambda f: f + b"\x00\x00"},
    {"name": "len_mismatch",       "mutate": lambda f: f[:8] + bytes([f[8]+1]) + f[9:]},

    # 校验级故障
    {"name": "crc_bitflip",        "mutate": lambda f: f[:-2] + bytes([f[-2] ^ 0xFF, f[-1]])},
    {"name": "mac_bitflip",        "mutate": lambda f: f[:-10] + bytes([f[-10] ^ 0x01]) + f[-9:]},

    # 语义级故障
    {"name": "unknown_cmd_id",     "mutate": lambda f: f[:6] + b"\xFF\xFF" + f[8:]},
    {"name": "zero_length_payload","mutate": lambda f: f[:8] + b"\x00\x00" + f[10:]},
]
```

**断言策略**：
- 帧头错误 / CRC 错误 / MAC 错误：必须抛 `ProtocolUnpackError` 或返回 `mac_verified=False`
- 未知 command_id：unpack 不抛异常（command_id 提取可能仍成功），但上层 `CommandStore.get_by_id()` 返回 `None`
- 长度不匹配：抛 `ProtocolUnpackError`，且错误信息包含 "length"

### 3.3 边界值覆盖（T3）

**目标**：验证字段在极限值时的序列化/反序列化正确性。

**表具行业典型边界值**：

| 字段类型 | 边界值 | 说明 |
|---------|--------|------|
| uint32 累积流量 | `0x00000000`, `0xFFFFFFFF` | 零值和满量程 |
| uint32 金额 | `0`, `99999999` | 零余额和最大充值额 |
| BCD 日期 | `000000`, `991231` | 世纪交界 |
| string 设备编号 | 空串, 最大长度, 含中文 | 长度边界和编码边界 |
| bitfield 状态字 | 全 0, 全 1 | 所有位独立验证 |
| raw_hex 密钥 | 16 字节, 24 字节, 32 字节 | 不同密钥长度 |

**测试方法**：
对 commands.yaml 中每个 `fields` 条目，按 `type` 自动推导边界值集合，生成对应的 pack/unpack 测试。

### 3.4 安全模式排列（T4）

**目标**：验证同一条命令在不同 `security_mode` 下的组帧差异。

**表具行业典型安全模式组合**（以 100 协议为例）：

| 功能码 | 方向 | 预期安全模式 | 验证点 |
|--------|------|------------|--------|
| 04H | 下行 | none | 无数据域，无 MAC |
| 05H | 下行 | cipher_mac | AES 加密 + HMAC |
| 05H | 上行 | plain_mac | 明文 + HMAC |
| 09H | 下行 | cipher_mac | AES 加密 + HMAC |
| 09H | 上行 | plain_mac | 明文 + HMAC |

**测试策略**：
- 对 P0 命令，遍历其所有可能的 `(command_id, is_uplink)` 组合
- 对每个组合，验证 `codec.pack()` 输出的安全域（加密/MAC/无）与预期一致
- 使用 `ctx.overrides` 强制覆盖 `security_mode`，验证 override 机制生效

### 3.5 会话状态机遍历（T5）

**目标**：验证涉及密钥协商/会话状态的命令序列能正确驱动状态转换。

**表具行业典型状态机**（以 100 协议为例）：

```
[初始]
  │
  ▼
[发送注册帧] ──► 收到随机码 ──► [已获取随机码]
                                     │
                                     ▼
                    [派生 enc_key / mac_key] ──► [已派生密钥]
                                     │
                                     ▼
                    [发送加密命令] ◄──────► [加密会话中]
                                     │
                    [密钥更新命令] ──► [密钥已更新]
```

**状态机测试用例**：

| 用例编号 | 状态转换路径 | 验证点 |
|---------|------------|--------|
| SM-001 | 初始 → 发送注册 → 收到随机码 | `session_updates` 包含 `random_000c` |
| SM-002 | 已获取随机码 → 派生密钥 | 派生后的 `enc_key` / `mac_key` 与文档示例一致 |
| SM-003 | 已派生密钥 → 发送加密命令 | pack 输出含密文 + MAC，unpack 能自洽验证 |
| SM-004 | 加密会话中 → 密钥更新 | 新旧密钥同时存在时的切换逻辑正确 |
| SM-005 | 异常路径：跳过注册直接发加密命令 | codec 应能处理（如从 session 取默认密钥），或抛出明确的 `ProtocolPackError` |

**实现要点**：
- 使用 `MockTransport + ProtocolChannel` 模拟完整收发闭环
- 通过 `session.apply_updates()` 模拟状态积累
- 每一步验证 `session.snapshot()` 中的状态键值

### 3.6 压力与稳定性（T6）

**目标**：验证 codec 在极端输入条件下的鲁棒性。

| 压力场景 | 构造方法 | 通过标准 |
|---------|---------|---------|
| 高频收发 | 循环 pack → unpack 1000 次 | 无内存泄漏，耗时线性增长 |
| 粘包 | 把 3 条完整帧拼接成 1 个 bytes 输入 `split_frames` | 正确拆出 3 条，不丢帧 |
| 断帧续传 | 先输入半条帧，超时后再输入后半条 | `split_frames` 正确处理（根据实现策略） |
| 全 0xFF / 全 0x00 输入 | `bytes(1024)` / `b'\xff'*1024` | 不抛未捕获异常，正常报解析失败 |
| 巨帧 | payload 长度 = 协议允许最大值 + 1 | 抛 `ProtocolUnpackError` 或截断处理（按协议定义） |

---

## 4. 矩阵模板

### 4.1 总览矩阵（Markdown）

```markdown
## 一致性测试总览矩阵 — <协议名>

| 命令名 | T1正例_REQ | T1正例_RSP | T2异常 | T3边界 | T4安全模式 | T5状态机 | T6压力 | 备注 |
|--------|-----------|-----------|--------|--------|-----------|---------|--------|------|
| 读取版本信息 | ✅ | ✅ | ✅ | ➖ | ✅ | ➖ | ➖ | 无安全层 |
| 读取累积流量 | ✅ | ✅ | ✅ | ✅ | ✅ | ➖ | ➖ | 金额边界关键 |
| 阀门控制 | ✅ | ✅ | ✅ | ➖ | ✅ | ➖ | ➖ | 下行命令 |
| 注册 | ✅ | ✅ | ✅ | ➖ | ✅ | SM-001 | ➖ | 状态机起点 |
| 充值 | ✅ | ✅ | ✅ | ✅ | ✅ | SM-003 | ✅ | 金额边界+压力 |
| ... | | | | | | | | |

图例: ✅=已覆盖 ➖=不适用 ❌=待补充
```

### 4.2 配置化矩阵（YAML）

```yaml
# conformance-matrix.yaml —— 与 commands.yaml 同目录
protocol_key: gh_protocol

# ===== 测试类型开关 =====
test_types:
  T1_positive: { enabled: true, priority: P0 }
  T2_fault_injection: { enabled: true, priority: P1 }
  T3_boundary: { enabled: true, priority: P1 }
  T4_security_mode: { enabled: true, priority: P1 }
  T5_state_machine: { enabled: true, priority: P1 }
  T6_stress: { enabled: false, priority: P2 }  # 默认关闭，送检前手动开启

# ===== 命令级测试配置 =====
commands:
  读取版本信息:
    T1_positive: { request: true, response: true }
    T2_fault_injection: { frames: [wrong_sof, crc_bitflip] }
    T3_boundary: { enabled: false }  # 无敏感字段
    T4_security_mode:
      modes: [none]  # 该命令仅支持明文

  读取累积流量:
    T1_positive: { request: true, response: true }
    T2_fault_injection: { frames: [wrong_sof, crc_bitflip, mac_bitflip, len_mismatch] }
    T3_boundary:
      enabled: true
      boundary_fields:
        - field: 累积流量
          values: ["0", "4294967295"]  # uint32 边界
        - field: 剩余金额
          values: ["0", "99999999"]
    T4_security_mode:
      modes: [auto, plain_mac, cipher_mac]

  注册:
    T1_positive: { request: true, response: true }
    T5_state_machine:
      transitions: [SM-001, SM-002]

# ===== 状态机定义（T5 用） =====
state_machine:
  initial_state: {}
  states:
    - name: has_random
      condition: "session.random_000c != ''"
    - name: has_derived_key
      condition: "session.enc_key != ''"
  transitions:
    - id: SM-001
      from: initial
      to: has_random
      trigger: 注册响应
      verify_session_keys: [random_000c]
    - id: SM-002
      from: has_random
      to: has_derived_key
      trigger: 密钥派生
      verify_session_keys: [enc_key, mac_key]
```

---

## 5. 从 commands.yaml 自动生成基础矩阵

### 5.1 生成脚本

```python
#!/usr/bin/env python3
"""从 commands.yaml 生成一致性测试矩阵框架。

用法:
    python generate_conformance_matrix.py \
        --commands config/protocols/100协议/commands.yaml \
        --output conformance-matrix.yaml
"""

import argparse
from pathlib import Path
import yaml


def auto_detect_boundary_values(field: dict) -> list[str]:
    """根据字段类型自动推导边界值。"""
    ft = field.get("type", "")
    length = int(field.get("length", 1))

    if ft in ("uint8", "uint16", "uint32"):
        max_val = (1 << (length * 8)) - 1
        return ["0", str(max_val), str(max_val // 2)]
    if ft in ("int16", "int32"):
        max_val = (1 << (length * 8 - 1)) - 1
        min_val = -(1 << (length * 8 - 1))
        return [str(min_val), "0", str(max_val)]
    if ft == "string":
        return ['""', '"A" * ' + str(length)]
    if ft in ("yymmddhhmmss", "yymmdd", "hhmmss"):
        return ["000000000000", "991231235959"]
    return []


def generate_matrix(commands_path: Path) -> dict:
    data = yaml.safe_load(commands_path.read_text(encoding="utf-8"))
    commands = data.get("commands", {})

    matrix = {
        "protocol_key": commands_path.parent.name,
        "test_types": {
            "T1_positive": {"enabled": True, "priority": "P0"},
            "T2_fault_injection": {"enabled": True, "priority": "P1"},
            "T3_boundary": {"enabled": True, "priority": "P1"},
            "T4_security_mode": {"enabled": True, "priority": "P1"},
            "T5_state_machine": {"enabled": True, "priority": "P1"},
            "T6_stress": {"enabled": False, "priority": "P2"},
        },
        "commands": {},
        "state_machine": {"initial_state": {}, "states": [], "transitions": []},
    }

    for cmd_name, cmd_def in commands.items():
        entry = {
            "T1_positive": {
                "request": "request" in cmd_def,
                "response": "response" in cmd_def,
            },
            "T2_fault_injection": {"frames": ["wrong_sof", "crc_bitflip"]},
            "T3_boundary": {"enabled": False, "boundary_fields": []},
            "T4_security_mode": {"modes": ["auto"]},
            "T5_state_machine": {"transitions": []},
        }

        # 自动识别边界值字段
        all_fields = []
        if "request" in cmd_def:
            all_fields.extend(cmd_def["request"].get("fields", []))
        if "response" in cmd_def:
            all_fields.extend(cmd_def["response"].get("fields", []))

        for f in all_fields:
            bvs = auto_detect_boundary_values(f)
            if bvs:
                entry["T3_boundary"]["enabled"] = True
                entry["T3_boundary"]["boundary_fields"].append(
                    {"field": f["name"], "values": bvs}
                )

        matrix["commands"][cmd_name] = entry

    return matrix


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--commands", required=True, type=Path)
    parser.add_argument("--output", required=True, type=Path)
    args = parser.parse_args()

    matrix = generate_matrix(args.commands)
    args.output.write_text(
        yaml.dump(matrix, sort_keys=False, allow_unicode=True),
        encoding="utf-8",
    )
    print(f"矩阵已生成: {args.output}")
    print(f"共 {len(matrix['commands'])} 条命令，请手动补充 T4/T5/T6 的配置")


if __name__ == "__main__":
    main()
```

### 5.2 人工填充指南

自动生成的矩阵只包含最保守的默认值，以下部分**必须人工补充**：

1. **T4 security_mode.modes**：根据协议文档，标注每条命令支持的安全模式（`none` / `plain_mac` / `cipher_mac`）
2. **T5 state_machine.transitions**：标注哪些命令是状态机转换的触发点（如注册、密钥更新）
3. **T2 fault_injection.frames**：对无安全层的命令删除 `mac_bitflip`；对定长帧增加 `len_mismatch`
4. **命令优先级标注**：在 YAML 注释中标记 P0/P1/P2，用于 CI 分级执行

---

## 6. pytest 实现框架

### 6.1 参数化测试

利用 `conformance-matrix.yaml` 驱动 pytest，实现"配置即测试"：

```python
"""一致性测试 —— 由 conformance-matrix.yaml 驱动。

运行方式:
    pytest tests/test_conformance.py -v
    pytest tests/test_conformance.py -v -k "T1 and 读取累积流量"   # 只跑指定命令的 T1
    pytest tests/test_conformance.py -v -m "P0"                     # 只跑 P0
"""

import pytest
import yaml
from pathlib import Path

from app.infrastructure.protocol.abc import PackContext
from app.infrastructure.validators.formatters.binary import BinaryFormatter
from plugins.<key>.codec import Codec


# ===== 加载矩阵 =====
MATRIX = yaml.safe_load(
    Path("config/protocols/<Display Name>/conformance-matrix.yaml").read_text()
)
CODEC = Codec()
CODEC.on_load(yaml.safe_load(Path("plugins/<key>/config.yaml").read_text()))


def _make_t1_params():
    """生成 T1 正例测试参数。"""
    for cmd_name, cfg in MATRIX["commands"].items():
        t1 = cfg.get("T1_positive", {})
        if t1.get("request"):
            yield pytest.param(cmd_name, "request", id=f"T1-{cmd_name}-REQ")
        if t1.get("response"):
            yield pytest.param(cmd_name, "response", id=f"T1-{cmd_name}-RSP")


@pytest.mark.parametrize("cmd_name,direction", list(_make_t1_params()))
def test_t1_positive(cmd_name, direction):
    """T1: 正例覆盖 —— 验证 pack/unpack 不抛异常且结构合规。"""
    # 从 commands.yaml 加载字段定义
    commands = yaml.safe_load(
        Path("config/protocols/<Display Name>/commands.yaml").read_text()
    )["commands"]
    cmd_def = commands[cmd_name][direction]
    fields_def = cmd_def.get("fields", [])
    command_id = int(cmd_def["command_id"], 16)

    # 用 default 值构造字段数据
    field_data = {
        f["name"]: f.get("default", "0")
        for f in fields_def
    }
    payload = BinaryFormatter.format_fields(field_data, fields_def)

    # Pack 测试
    is_uplink = direction == "response"
    frame = CODEC.pack(PackContext(
        command_id=command_id,
        payload=payload,
        is_uplink=is_uplink,
        session={},
    ))
    assert isinstance(frame, bytes) and len(frame) > 0

    # Unpack 测试（自环）
    result = CODEC.unpack(frame, {})
    assert result.command_id == command_id
    assert isinstance(result.payload, bytes)


# ===== T2: 异常帧注入 =====
FAULT_TEMPLATES = {
    "wrong_sof": lambda f: b"\xDE\xAD" + f[2:],
    "crc_bitflip": lambda f: f[:-2] + bytes([f[-2] ^ 0xFF, f[-1]]),
    "mac_bitflip": lambda f: f[:-10] + bytes([f[-10] ^ 0x01]) + f[-9:],
    "len_mismatch": lambda f: f[:8] + bytes([f[8] + 1]) + f[9:] if len(f) > 9 else f,
}


def _make_t2_params():
    for cmd_name, cfg in MATRIX["commands"].items():
        t2 = cfg.get("T2_fault_injection", {})
        for fault_name in t2.get("frames", []):
            yield pytest.param(cmd_name, fault_name, id=f"T2-{cmd_name}-{fault_name}")


@pytest.mark.parametrize("cmd_name,fault_name", list(_make_t2_params()))
def test_t2_fault_injection(cmd_name, fault_name):
    """T2: 异常帧 —— 验证 codec 面对脏数据时不崩、不误判。"""
    # 先 pack 一条合法帧
    commands = yaml.safe_load(
        Path("config/protocols/<Display Name>/commands.yaml").read_text()
    )["commands"]
    cmd_def = commands[cmd_name]["request"]
    fields_def = cmd_def.get("fields", [])
    command_id = int(cmd_def["command_id"], 16)

    field_data = {f["name"]: f.get("default", "0") for f in fields_def}
    payload = BinaryFormatter.format_fields(field_data, fields_def)
    valid_frame = CODEC.pack(PackContext(
        command_id=command_id, payload=payload, is_uplink=False, session={},
    ))

    # 注入故障
    mutator = FAULT_TEMPLATES[fault_name]
    bad_frame = mutator(valid_frame)

    # 异常断言：允许抛出 ProtocolUnpackError，或返回带错误标记的结果
    from app.infrastructure.protocol.exceptions import ProtocolUnpackError
    try:
        result = CODEC.unpack(bad_frame, {})
        # 如果 unpack 没抛异常，必须有某种错误指示
        if hasattr(result, 'meta') and result.meta:
            assert result.meta.get('error') or not result.meta.get('mac_verified', True)
    except ProtocolUnpackError:
        pass  # 预期路径


# ===== T3: 边界值 =====
def _make_t3_params():
    for cmd_name, cfg in MATRIX["commands"].items():
        t3 = cfg.get("T3_boundary", {})
        if not t3.get("enabled"):
            continue
        for bf in t3.get("boundary_fields", []):
            for val in bf["values"]:
                yield pytest.param(cmd_name, bf["field"], val,
                                   id=f"T3-{cmd_name}-{bf['field']}-{val[:20]}")


@pytest.mark.parametrize("cmd_name,field_name,value", list(_make_t3_params()))
def test_t3_boundary(cmd_name, field_name, value):
    """T3: 边界值 —— 验证极限值的 pack/unpack 一致性。"""
    commands = yaml.safe_load(
        Path("config/protocols/<Display Name>/commands.yaml").read_text()
    )["commands"]
    cmd_def = commands[cmd_name]["request"]
    fields_def = cmd_def.get("fields", [])
    command_id = int(cmd_def["command_id"], 16)

    field_data = {f["name"]: f.get("default", "0") for f in fields_def}
    field_data[field_name] = value

    payload = BinaryFormatter.format_fields(field_data, fields_def)
    frame = CODEC.pack(PackContext(
        command_id=command_id, payload=payload, is_uplink=False, session={},
    ))

    # 往返验证
    result = CODEC.unpack(frame, {})
    # 对于某些字段（如 string），unpack 后可能和输入不完全一致（补零等），
    # 这里只做不抛异常 + command_id 正确的弱验证
    assert result.command_id == command_id
```

### 6.2 跳过标记与分级执行

```python
# 在 pytest.ini 或 conftest.py 中注册标记
def pytest_configure(config):
    config.addinivalue_line("markers", "P0: 送检必过")
    config.addinivalue_line("markers", "P1: 功能完整")
    config.addinivalue_line("markers", "P2: 可选增强")

# 在生成参数时根据矩阵 priority 打标记（通过 pytest.param 的 marks 参数）
```

### 6.3 覆盖率报告

一致性测试跑完后，应输出命令覆盖率报告：

```python
# conftest.py 或测试收尾

def pytest_terminal_summary(terminalreporter, exitstatus, config):
    """输出一致性测试覆盖率摘要。"""
    total_commands = len(MATRIX["commands"])
    tested = set()
    for item in terminalreporter.stats.get("passed", []):
        # 从 nodeid 提取命令名
        if "test_t" in item.nodeid:
            # 简化处理：实际应从 matrix 或 item 的 user_properties 读取
            pass
    terminalreporter.write_sep("=", "一致性测试覆盖率摘要")
    terminalreporter.write_line(f"总命令数: {total_commands}")
```

---

## 7. 与计量院检测项目的映射

表具上位机软件送检时，计量院/检测站通常会按以下项目检测。本矩阵与检测项目的对应关系：

| 检测站项目 | 对应本文测试类型 | 关键验收标准 |
|-----------|---------------|------------|
| **通信功能试验** | T1 正例 | 全部管理/计量/控制命令收发正常 |
| **数据安全性试验** | T4 安全模式 + T5 状态机 | 加密帧不可被明文解析；密钥更新后旧密钥失效 |
| **异常处理能力** | T2 异常帧 | 异常帧不导致设备死机/数据紊乱 |
| **数据正确性试验** | T3 边界值 + T1 正例 | 累积量/金额/余量等计量数据与标准装置一致 |
| **连续工作试验** | T6 压力 | 连续通信 8h 无丢帧、无内存泄漏 |

**送检前检查点**：
- [ ] T1 正例覆盖率 100%（P0 命令）
- [ ] T4 安全模式遍历了文档声明的全部组合
- [ ] T5 状态机至少跑了：注册 → 派生密钥 → 加密通信 → 密钥更新 的完整闭环
- [ ] T3 边界值包含：金额零值、金额最大值、流量满量程、日期临界值

---

## 8. 检查清单

### 8.1 矩阵生成阶段

- [ ] 已运行 `generate_conformance_matrix.py` 生成基础矩阵
- [ ] 已人工补充每条命令的 `T4.security_mode.modes`
- [ ] 已人工标注状态机转换命令的 `T5.transitions`
- [ ] 已删除无安全层命令的 `mac_bitflip` 异常帧
- [ ] 已根据协议文档补充特殊边界值（如单价调整范围、阀门动作次数上限）

### 8.2 测试执行阶段

- [ ] `pytest tests/test_conformance.py -m P0` 全部通过
- [ ] T2 异常帧测试中，每条异常帧都有明确的错误指示（抛异常或 meta 标记）
- [ ] T3 边界值测试中，计量类字段（金额/流量/余量）的往返误差为 0
- [ ] T4 安全模式测试中，加密帧的 payload 经过 `unpack` 后与原始字段一致
- [ ] T5 状态机测试中，`session.snapshot()` 在每个状态节点包含预期键值

### 8.3 送检准备阶段

- [ ] 一致性测试报告已导出（含命令覆盖率统计）
- [ ] T6 压力测试已连续运行 ≥ 8 小时（或协议要求的时长）
- [ ] 全部测试用例可脱离真实硬件运行（MockTransport 闭环）
