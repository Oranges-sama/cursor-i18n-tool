<div align="center">

# Cursor 一键汉化工具

让 Cursor 编辑器显示中文界面。一键翻译，一键还原，不影响正常使用。

[![Node.js](https://img.shields.io/badge/Node.js-%3E%3D16-green?logo=node.js)](https://nodejs.org)
[![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20macOS-blue)](#)
[![License](https://img.shields.io/badge/License-MIT-yellow)](./LICENSE)
[![Version](https://img.shields.io/badge/Version-1.2.0-orange)](./package.json)

</div>

> 本仓库是 [Wuyf5275/cursor-i18n-tool](https://github.com/Wuyf5275/cursor-i18n-tool) 的 Fork（[@Caph-dev](https://github.com/Caph-dev/cursor-i18n-tool)）。在 upstream 基础上扩展了词典、修复了 Tab 键位误汉化问题，并增加了安装路径自定义等功能。

## 功能

- **一键汉化**：自动替换 Cursor 核心 JS 中的 UI 文案。
- **一键还原**：随时恢复英文原版，干净可逆。
- **智能定位安装路径**：自动搜索 Cursor 安装目录，支持手动指定与多版本选择。
- **路径记忆**：上次使用的安装路径保存到用户配置，下次自动沿用。
- **完整性修复**：自动重算 `workbench.desktop.main.js` 哈希，消除「安装已损坏」警告。
- **macOS 适配**：自动清除隔离属性并本地重签名，无需手动执行 `xattr`。
- **权限处理**：权限不足时自动请求管理员权限。
- **自动备份**：首次运行备份原文件，后续运行保留已有补丁。
- **可打包**：支持通过 `pkg` 构建为独立可执行文件。

## 效果预览

<table>
  <tr>
    <td><img src="./screenshots/preview-general.png" alt="通用设置" width="500"/></td>
    <td><img src="./screenshots/preview-agents.png" alt="智能体设置" width="500"/></td>
  </tr>
  <tr>
    <td align="center"><b>通用设置（General）</b></td>
    <td align="center"><b>智能体设置（Agents）</b></td>
  </tr>
</table>

## 快速开始

### 方式一：下载可执行文件

前往仓库 [`dist/`](https://github.com/Wuyf5275/cursor-i18n-tool/tree/main/dist) 目录，下载对应平台的可执行文件：

| 平台 | 文件 | 说明 |
|------|------|------|
| Windows x64 | `cursor-i18n-tool-win-x64.exe` | 双击运行 |
| macOS x64 | `cursor-i18n-tool-macos-x64` | Intel 芯片 |
| macOS ARM | `cursor-i18n-tool-macos-arm64` | Apple 芯片 |

macOS 首次运行可能需要：右键 → 打开，或在终端执行 `chmod +x ./cursor-i18n-tool-macos-arm64 && ./cursor-i18n-tool-macos-arm64`。

### 方式二：源码运行

环境要求：Node.js ≥ 16（推荐 18+），npm 随 Node.js 自带。

```bash
git clone https://github.com/Caph-dev/cursor-i18n-tool.git
cd cursor-i18n-tool
npm install
node index.js
```

### 指定 Cursor 安装路径

若 Cursor 安装在非默认位置（例如 `D:\Program Files\cursor\`），启动时可：

- 在交互菜单中选择「手动指定其他路径」或「重新自动搜索」。
- 使用命令行参数 `--cursor-path`（支持安装根目录、`resources/app`、`Cursor.exe` 或 macOS 的 `Cursor.app`）。

配置会保存到 `~/.cursor-i18n-tool/config.json`。

**自动搜索范围摘要：**

| 平台 | 默认探测位置 |
|------|--------------|
| Windows | `%LOCALAPPDATA%\Programs\cursor`、Program Files、注册表 `InstallLocation` |
| macOS | `/Applications/Cursor.app`、`~/Applications/Cursor.app` 及 `Cursor*.app` 扫描 |

### 命令行静默模式

```bash
# 直接汉化
node index.js --action=translate

# 指定安装路径后汉化（路径含空格请加引号）
node index.js --action=translate --cursor-path="D:\Program Files\cursor"

# 恢复英文
node index.js --action=restore

# 指定路径后恢复
node index.js --action=restore --cursor-path="D:\Program Files\cursor"
```

提权后的子进程会携带 `--cursor-path`，无需再次选择路径。

## 项目结构

```
cursor-i18n-tool/
├── index.js              # 入口：交互菜单 + 提权逻辑
├── src/
│   ├── i18n-core.js      # 核心引擎：正则替换 + 哈希修复 + Gatekeeper
│   ├── dict.js           # 翻译词典
│   └── platform.js       # 平台适配：路径探测/规范化/配置 + 权限检测 + 提权
├── package.json
└── README.md
```

## 技术原理

### 分层替换策略

| 层级 | 策略 | 目标 |
|------|------|------|
| L1 顽固词条 | `trickyReplacements` | 含特殊转义、模板字符串的复杂词条 |
| L2 安全长句 | `safeMegaRegex` | 被引号包裹的长句，按长度降序匹配 |
| L3 裸文本长句 | `longMegaRegex` | ≥20 字符的裸文本，不与代码变量冲突 |
| L4 危险短词 | `riskyRegexes` | 仅在 `children:`、`title:` 等 UI 属性上下文中替换；键位扫描表附近自动跳过 |
| L4.5 作用域替换 | `scopedReplacements` | 设置侧栏 ID、编译模板中的固定片段等 `dict` 无法覆盖的文案 |

### 文件完整性修复

Cursor 启动时会校验核心文件哈希。修改后工具会：

1. 使用内存中已汉化的 `workbench.desktop.main.js` 内容计算校验值，避免写回后立刻读盘失败（尤其在 `Program Files` 等系统目录）。
2. 重新计算哈希（自动检测 MD5/SHA256/SHA512）。
3. 更新 `product.json` 中对应的校验值。

核心 JS 写回采用「临时文件替换 + 直接覆盖回退」，在文件被占用时尽量保持可写行为。

### macOS Gatekeeper 处理

修改 `.app` 包内文件会破坏签名，工具会自动：

1. 清除隔离属性：`xattr -cr`。
2. 本地重签名：`codesign --force --deep --sign -`。

## 打包构建

```bash
npm install

# 全平台
npm run build

# 仅 Windows
npm run build:win

# 仅 macOS
npm run build:mac
```

产物输出到 `dist/` 目录。

## 与上游的差异

| 项目 | upstream `main` | 本 Fork |
|------|-----------------|---------|
| 版本 | 1.0.0 | 1.2.0 |
| Tab 键位修复 | 否 | 是 |
| 设置页词典扩充 | 基础 | 扩展 |
| `scopedReplacements` | 否 | 是 |
| 键位上下文保护 | 否 | 是 |
| 自定义安装路径 | 否 | 是 |
| 路径配置记忆 | 否 | 是 |

同步上游更新：

```bash
git remote add upstream https://github.com/Wuyf5275/cursor-i18n-tool.git
git fetch upstream
git merge upstream/main
```

合并时请保留本 Fork 的 `src/dict.js` 和 `src/i18n-core.js` 改动。

## 贡献指南

### 添加翻译词条

编辑 `src/dict.js`：

```javascript
// 安全长句（≥3 个单词或含特殊字符）→ safeGlobalDict
"Your english text here": "你的中文翻译",

// 危险短词（1-2 个单词，可能与代码变量冲突）→ riskyShortWords
"Settings": "设置",
```

**字典选择原则：**

- 长度 ≥ 20 字符或含 3 个以上单词 → `safeGlobalDict`。
- 短词（可能与代码变量名冲突）→ `riskyShortWords`（仅在 UI 属性上下文中替换；不要将 `Tab` 等键名放入此表）。
- 设置侧栏 key、编译模板中的固定片段 → `src/i18n-core.js` 的 `scopedReplacements`。

### 特殊格式词条

含模板字符串、转义字符或需要特殊处理的词条，请添加到 `src/i18n-core.js` 的 `trickyReplacements` 数组。

## 注意事项

- Cursor 每次更新后需重新运行汉化，更新会覆盖已修改的文件。
- 工具会自动备份原文件（`.backup` 后缀），可随时还原。
- 部分由服务器动态下发的文案无法通过本工具翻译。
- 汉化前请完全退出 Cursor（含系统托盘），避免文件占用。
- 安装在 `Program Files` 等系统目录时，若提示权限不足，请以管理员身份运行本工具。
- 写回失败或界面异常时，可尝试恢复英文或重装 Cursor 后再汉化。

## 开源许可

[MIT License](./LICENSE)
