# 协议接入技术扩展参考

配合 protocol-integration-workflow skill 使用。

---

## A. PackContext 与 UnpackResult：完整数据流

### A.1 发送端

GUI -> Facade -> 剥离 codec_override -> Runtime.send -> Channel.send
  -> CommandStore -> DirectionDef
  -> BinaryFormatter -> payload bytes
  -> PackContext(command_id, payload, is_uplink, session, overrides)
  -> codec.pack(ctx) -> frame bytes -> transport.write

### A.2 接收端

transport callback -> Channel._on_raw_data
  -> codec.split_frames -> [frame]
  -> codec.unpack -> UnpackResult
  -> session.apply_updates
  -> CommandStore.get_by_id -> command_name
  -> _deserialise_payload -> field dict
  -> ParsedFrame -> callbacks -> Facade signal

---

## B. 安全层实现模式

### B.1 无安全层（HIL协议）

payload -> CRC16 -> 嵌入帧。codec内部完成CRC计算和校验。

### B.2 单层安全（新金卡协议）

payload -> DES-ECB加密前8字节 -> CRC16 -> CS校验。用construct的Subconstruct声明式定义安全层。

### B.3 双层安全（100协议/gh_protocol）

payload -> AES-ECB+PKCS7 -> HMAC-SHA256 MAC -> CRC16。
安全模式按功能码+方向自动决策（04H下行=none, 05H下行=cipher_mac, 09H=plain_mac）。
密钥派生：enc_key=HMAC-SHA256(主密钥,随机码)[:16], mac_key=AES(主密钥,随机码)[:16]。

### B.4 安全覆盖

```yaml
instruction_attributes:
  security_mode:
    values: [auto, plain_mac, cipher_mac, none]
    codec_override: true
```

---

## C. 同构协议：拷贝后独立运行

### C.1 核心原则

**每套协议目录是独立交付单元**。当新协议与已有协议帧结构/安全层一致时，从已有协议**拷贝**底层文件后修改。**禁止跨协议 import**（from plugins.other_protocol import ...）。

### C.2 正确做法

步骤1: 拷贝底层文件
  cp plugins/gh_protocol/frame_security.py plugins/jk_100_protocol/frame_security.py

步骤2: 修改拷贝后的文件
  - 类名/常量（HEAD/TAIL/DID范围）
  - 功能码映射
  - 默认安全策略

步骤3: codec.py同目录相对导入
  from .frame_security import FrameCodec    # 正确
  # from plugins.gh_protocol.frame_security import ...  # 禁止

步骤4: 验证独立性
  删除/重命名其他协议目录，确认新协议仍可加载

### C.3 完整示例

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

### C.4 拷贝后需修改

类名、帧头/帧尾、DID映射表、功能码映射、随机码偏移、MAC长度、默认安全模式。

### C.5 何时不应该拷贝

CRC/加密/MAC完全不同、安全层完全不同、帧结构差异太大 -> 全新编写。

---

## D. 异常路径处理

### D.1 codec层

pack失败抛ProtocolPackError。CRC失败抛ProtocolUnpackError。MAC失败抛ProtocolAuthError。

### D.2 Channel层传播

unpack异常 -> warning/error日志 -> error_callbacks -> 继续，永不中断接收循环。

### D.3 非致命容错

_allow_unverified_parse=True：MAC失败仍返回payload，标记mac_verified:false。

---

## E. 测试策略

### E.1 单元测试

test_pack_minimal, test_unpack_valid, test_unpack_crc_error, test_unpack_mac_error。

### E.2 集成测试

MockTransport + ProtocolChannel.create + send + simulate_receive。

### E.3 独立性测试

用ast解析codec.py的import，验证不引用其他plugins/*模块。

---

## F. 故障排查

| 症状 | 原因 | 排查 |
|------|------|------|
| GUI无协议 | config.yaml缺失 | 检查plugins/目录 |
| 命令为空 | commands.yaml路径错 | 显示名=目录名 |
| 帧长度错 | field length不匹配 | 逐字段验证 |
| 解析乱码 | 密钥未进session | 检查密钥路径 |
| CRC失败 | 算法或范围错 | 对照文档 |
| seq不递增 | session_updates缺失 | unpack返回 |
| override无效 | 未标记codec_override | 检查指令属性 |
| 跨协议import | 引用其他插件 | 拷贝+相对导入 |
