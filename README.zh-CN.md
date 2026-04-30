# Claude Code to Figma

[![Made for Claude Code](https://img.shields.io/badge/Made%20for-Claude%20Code-blueviolet?style=flat-square&logo=anthropic)](https://docs.anthropic.com/en/docs/claude-code)
[![Figma MCP](https://img.shields.io/badge/Figma-MCP%20Server-ff7262?style=flat-square&logo=figma)](https://www.npmjs.com/package/@anthropic-ai/figma-mcp)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

**一组 Claude Code Skills，让 AI 在 Figma 里按照 Design System 的规范构建设计——组件绑定 Master Component，视觉值绑定 Token，零裸值。**

[English Version](README.md)

---

## 效果对比

### Without Skills — 裸值、无绑定

<!-- 替换为实际截图 -->
![Without Skills](assets/without-skills.png)

> 组件从零构建，颜色/字体/间距是硬编码的数值，与 Design System 无关联。

### With Skills — 组件链接、Token 绑定

<!-- 替换为实际截图 -->
![With Skills](assets/with-skills.png)

> 组件是 Master Component 的 Instance，所有颜色、字体、间距、圆角都绑定了 Design System 的 Variable 和 Style。

---

## Preflight 预检系统

每次设计会话开始前，自动执行 7 项检查，确保一切连接就绪：

<!-- 替换为实际截图 -->
![Preflight Status](assets/preflight-status.png)

---

## 它能做什么

- **一键预检** — 验证 MCP 连接，加载 Styles / Variables / Components 完整清单
- **组件优先** — 从 Design System 查找并复用组件，创建的都是 Master Component 的 Instance
- **Token 强绑定** — 颜色、字体、间距、圆角全部绑定 Variable / Style，禁止裸值
- **自动 QA** — 每次写入 Figma 后自动检查所有节点的绑定状态
- **参考解读** — 给一张截图或 URL，输出结构化 Design Brief

## 适合 / 不适合

| 适合 | 不适合 |
| ---- | ------ |
| 基于现有 Design System 搭建页面 | 从零创建 Design System |
| 用自然语言描述 UI，Claude 在 Figma 里构建 | 精细的像素级插画 / 图标绘制 |
| 在 DS 文件或链接了 DS 的新文件中工作 | 没有 Design System 的自由设计 |
| 确保设计产出 100% 遵守 Token 规范 | FigJam / 白板类工作 |

---

## 安装

### 前置条件

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (CLI / Desktop / VS Code)
- [Figma MCP Server](https://www.npmjs.com/package/@anthropic-ai/figma-mcp) 已安装并认证
- 一个包含 Design System 的 Figma 文件（本地定义或通过 Library 链接）

### 安装步骤

```bash
# 1. 克隆仓库
git clone https://github.com/senlindesign/cc2figma.git

# 2. 将 skills 复制到你的项目
cp -r cc2figma/.claude/skills/* your-project/.claude/skills/

# 3. 复制配置文件
cp cc2figma/.claude/settings.json your-project/.claude/settings.json

# 4. 复制 CLAUDE.md 模板到项目根目录
cp cc2figma/CLAUDE.md.template your-project/CLAUDE.md
```

### 配置 CLAUDE.md

打开 `CLAUDE.md`，填入你的 Figma 文件信息：

```markdown
# Figma Design Project

- **Figma file:** <https://www.figma.com/design/YOUR_FILE_KEY/...>
- **Fonts:** Inter (primary) · Roboto Mono (code)
- **Session goal:** [今天要设计什么？]

## Rules

1. Every visual value must bind to a Style or Variable.
2. Always search connected libraries before building any component from scratch.
3. Never start designing before the Design Brief is confirmed.
```

> Fonts 字段可以留空——Preflight 会自动从 Design System 的 Variable 中读取并填充。

---

## 使用流程

```
1. 打开 Claude Code，进入你的项目目录
2. 说 "Let's start" 或 "开始"               → 触发 Preflight 预检
3. 等待 7 项检查全部通过                      → Styles / Variables / Components 已加载
4. 描述想要的设计                             → "做一个登录页，包含邮箱和密码输入框"
5. Claude 查找 DS 组件 → 创建 Instance → 绑定 Token
6. 每步完成后截图验证                          → 确认后继续下一个 section
7. 最终产出：100% Token 绑定、组件链接完整的 Figma 设计
```

---

## Skills 一览

| Skill | 触发条件 | 职责 |
| ----- | -------- | ---- |
| `figma-preflight` | "let's start"、首次分享 Figma URL | 7 项预检 + Token Map + Component Registry |
| `component-rules` | "创建 card"、"做一个按钮"、构建 UI | 组件库优先查找、Auto Layout、语义化命名 |
| `figma-style-binding` | 设置颜色、字体、间距 | 强制所有视觉值绑定 Style 或 Variable |
| `figma-qa-verifier` | 每次 `use_figma` 后自动触发 | 检查节点绑定状态，报告裸值 |
| `reference-interpreter` | 分享截图 / URL / 设计描述 | 输出结构化 Design Brief |

### 它们如何协作

```
Session start
    |
    v
+-----------------+
|  figma-preflight | <-- 加载 Token Map + Component Registry
+--------+--------+
         |
         v
+-----------------+     +---------------------+
| component-rules  | --> |  figma-style-binding |
| 查找组件、搭结构  |     |  绑定颜色/字体/间距    |
+--------+--------+     +----------+----------+
         |                        |
         v                        v
    +-------------------------------------+
    |       figma-qa-verifier             |
    |  检查所有节点的 Style/Variable 绑定    |
    +-------------------------------------+
```

---

## 适用场景

| 场景 | 工作原理 |
| ---- | -------- |
| **直接在 DS 文件中工作** | Preflight 读取所有本地 Styles、Variables 和 Components，完整 Token 绑定。 |
| **新文件链接了 DS Library** | 通过 `importComponentByKeyAsync` 导入的 Instance 自动继承 Master Component 的所有 Token 绑定。新建 frame 可通过 `importVariableByKeyAsync` 导入 Library 变量。 |

---

## 目录结构

```
your-project/
├── CLAUDE.md                          # 项目配置（Figma URL、字体、规则）
└── .claude/
    ├── settings.json                  # 权限 + QA Hook
    └── skills/
        ├── figma-preflight/
        │   └── SKILL.md               # 7 项预检 + Token Map + Component Registry
        ├── component-rules/
        │   └── SKILL.md               # 组件构建规则（Library-first、Auto Layout、命名）
        ├── figma-style-binding/
        │   └── SKILL.md               # 样式绑定规则（Color / Text / Spacing）
        ├── figma-qa-verifier/
        │   └── SKILL.md               # QA 验证规则
        └── reference-interpreter/
            └── SKILL.md               # 参考解读 → Design Brief
```

---

## 核心原则

1. **组件优先** — 先查 Design System，找不到才从零构建
2. **Token 强绑定** — 所有视觉值必须绑定 Variable 或 Style，不允许裸值
3. **增量构建** — 一次一个 section，截图验证后再继续
4. **语义命名** — 每个节点使用 `"Section / Role"` 格式
5. **透明 QA** — 每次操作返回节点 ID，自动验证绑定状态

## 已知限制

| 限制 | 说明 | 应对 |
| ---- | ---- | ---- |
| 嵌套实例修改 | 双重嵌套 Instance 无法直接修改内部属性 | 先 `detachInstance()` 外层 |
| Library 变量不可枚举 | `getLocalVariablesAsync()` 看不到 Library 变量 | Instance 自动继承 Token；新建 frame 用 `importVariableByKeyAsync` |
| Variant 属性值 | `setProperties` 传入无效值会回滚整个脚本 | 先读 `componentPropertyDefinitions` 确认有效值 |

---

## 贡献

欢迎提交 Issue 和 PR。如果你有新的 Figma 设计场景需要 Skill 支持，请在 Issue 中描述你的工作流程。

## License

MIT © 2025 Sen Lin
