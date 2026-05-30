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

| 文件 | 内容 |
|------|------|
| `SKILL.md` | 八维度分析：架构分层、帧编解码、安全层、数据类型、参数模型、UI契约、决策树、检查清单 |
| `references/technical-extension.md` | 代码级细节：完整数据流、三种安全模式代码、同构拷贝、异常处理、测试策略、故障排查 |

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
