# Agent 指令：cursor-i18n-tool

本仓库是 Cursor IDE 的汉化工具。当用户要求为新版 Cursor 更新翻译时，请按以下工作流执行。

## 翻译更新工作流

1. **检测已安装的 Cursor**
   - 在 macOS 上读取 `/Applications/Cursor.app/Contents/Info.plist` 获取版本号。
   - 确认 `out/vs/workbench/workbench.desktop.main.js` 存在且可写。

2. **查找缺失字符串**
   - 在 `workbench.desktop.main.js` 中搜索新的 UI 字符串（通常来自截图或设置页）。
   - 提取上下文，确认字符串真实存在且归属同一设置分组。
   - 与 `src/dict.js` 对比（`safeGlobalDict` 与 `riskyShortWords`）。

3. **更新 `src/dict.js`**
   - 长句 / 设置标签 → `safeGlobalDict`。
   - 短下拉值、分组标签、可能与代码符号冲突的单词 → `riskyShortWords`。
   - 不要把 `"Tab"` 等键名放入 `riskyShortWords`，否则会破坏快捷键绑定。

4. **校验词典语法**
   - 运行 `node -e "require('./src/dict.js')"` 确认文件可正常解析。

5. **应用翻译到本地 Cursor**
   - 运行 `node index.js --action=translate`。
   - 工具会自动创建或复用 `.backup` 文件，patch `workbench.desktop.main.js`，更新 `product.json` 校验值，并修复 macOS Gatekeeper 签名。

6. **验证补丁**
   - 在已 patch 的 `workbench.desktop.main.js` 中 grep 新的中文字符串。
   - 确认对应的原始英文字符串已消失。
   - 提示用户重启 Cursor 查看效果。

## 项目结构说明

- `index.js` — CLI 入口（交互菜单 + 提权逻辑）。
- `src/dict.js` — 翻译词典（`safeGlobalDict`、`riskyShortWords`）。
- `src/i18n-core.js` — 补丁引擎、哈希修复、Gatekeeper 修复。
- `src/platform.js` — Cursor 路径探测与权限处理。

## 安全规则

- 始终保留已有的 `.backup` 文件，工具会自动处理。
- 不要提交用户已安装 Cursor 应用的改动，只提交 `src/dict.js`。
- 如果用户希望扩大覆盖范围，向其索要截图或 `workbench.desktop.main.js` 的相关片段。
