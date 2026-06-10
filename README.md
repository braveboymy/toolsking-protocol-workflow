# ToolsKing 协议接入工作流

从第一性原理出发的通信协议接入方法论。涵盖分层架构、帧编解码契约、安全层管线、数据类型边界、参数模型和 UI 契约。

## 适用场景

为协议调试工具（如 ToolsKing）接入新的通信协议时使用。覆盖：
- 分析协议文档，判断与已有协议的同构性
- 理解分层架构中每一层的职责边界
- 处理无安全层 / 单层安全 / 双层安全三种复杂度
- 在类型系统边界内外选择正确的解析策略
- 管理持久化参数与运行时会话状态

## 内容

### 核心技能文件

| 文件 | 内容 |
|------|------|
| `SKILL.md` | 完整八维度分析：架构分层、帧编解码、安全层、数据类型、参数模型、UI 契约、决策树、实现检查清单 |

### 参考文档

| 文件 | 内容 | 使用阶段 |
|------|------|---------|
| `references/document-analysis-workflow.md` | **Phase 0**: PDF/Word/MD 协议文档系统化分析流程，帧格式/命令表/CRC/安全参数/测试向量的提取方法论 | 拿到新协议文档后**最先阅读** |
| `references/field-type-mapping.md` | 协议文档类型描述 → 22 种 FieldType 的精确映射表，含字节序决策和配置要点 | 提取命令字段时对照使用 |
| `references/commands-yaml-template.md` | 从协议文档命令表到可用的 `commands.yaml` 的完整转换模板。**第 12 节专讲 TLV/自描述格式处理** | 编写 commands.yaml 时对照使用 |
| `references/test-vector-extraction.md` | 测试向量定位→提取→pytest 契约测试生成，含 CRC 参数反推、往返测试、错误帧测试 | 实现 codec 后立即使用，**解决无法测试的问题** |
| `references/technical-extension.md` | 代码级细节：完整数据流、三种安全模式代码、同构拷贝、异常处理、测试策略、故障排查 | codec 实现时参考 |

### 推荐使用顺序

```
1. document-analysis-workflow.md  ← 先分析文档
2. field-type-mapping.md          ← 映射字段类型
3. commands-yaml-template.md      ← 编写命令定义
4. SKILL.md                       ← 理解架构、实现 codec
5. test-vector-extraction.md      ← 编写契约测试
6. technical-extension.md         ← 处理特殊情况
```

## 安装

### pi

```bash
pi install git:github.com/braveboymy/toolsking-protocol-workflow
```

### Codex

```bash
cd your-project
mkdir -p .codex/skills
git clone https://github.com/braveboymy/toolsking-protocol-workflow .codex/skills/toolsking-protocol-workflow
```

### Claude Code

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/braveboymy/toolsking-protocol-workflow ~/.claude/skills/toolsking-protocol-workflow
```

## 更新

```bash
cd ~/.claude/skills/toolsking-protocol-workflow
git pull
```

## 许可

MIT
