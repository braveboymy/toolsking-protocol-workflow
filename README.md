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
| `SKILL.md` | 完整协议接入方法论：分层架构、帧编解码、安全层、数据类型、参数模型、UI 契约、决策树、实现检查清单（含分阶段参考文档速查表） |

### 参考文档

| 文件 | 内容 | 阶段 |
|------|------|------|
| `references/document-analysis-workflow.md` | **Phase 0**：PDF/Word/MD 协议文档系统化分析流程 | 第一阶段 — 最先阅读 |
| `references/field-type-mapping.md` | 协议文档类型描述 → 22 种 FieldType 精确映射表 | 第二阶段 — 提取字段时对照 |
| `references/commands-yaml-template.md` | 从命令表到 `commands.yaml` 的完整转换模板。**第 12 节专讲 TLV 格式**，**第 13 节专讲表具计量数据** | 第二阶段 — 编写 commands.yaml |
| `references/data-domain-polymorphism.md` | **数据域多态**：表型决定字段集——决定性参数 → 会话状态 → 条件解析的完整模式 | 第二阶段 — 协议有表型差异时必读 |
| `references/test-vector-extraction.md` | 测试向量定位→提取→pytest 契约测试生成 | 第三阶段 — codec 完成后 |
| `references/conformance-test-matrix.md` | **Phase 3.5**：命令全覆盖 × 六类测试类型，含自动生成脚本、pytest 参数化框架、计量院检测映射 | 第三阶段 — 送检/发布前 |
| `references/technical-extension.md` | 代码级参考：Codec 分步实现指南、同构拷贝示例、异常处理、故障排查 | 第三阶段 — codec 实现时 |
| `references/anti-patterns.md` | **反模式清单**：7 种最常见错误及修正方法 | 全阶段 — 实现过程中随时对照 |

### 推荐使用顺序

> 详见 `SKILL.md` 头部的完整分阶段工作流。简要版本：
>
> **第一阶段：全局认知**
> 1. `references/document-analysis-workflow.md`
> 2. `SKILL.md` §1-§2（架构与 codec 契约）
>
> **第二阶段：配置与命令**
> 3. `references/field-type-mapping.md`
> 4. `references/commands-yaml-template.md`
> 5. `SKILL.md` §5-§6（参数模型与 UI 契约）
> 6. `references/data-domain-polymorphism.md`（协议有表型差异时）
>
> **第三阶段：实现与测试**
> 6. `SKILL.md` §3, §7（安全层与决策树）
> 7. `references/technical-extension.md`（代码级参考）
> 8. `references/test-vector-extraction.md`（契约测试）
> 9. `references/conformance-test-matrix.md`（一致性测试）

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
