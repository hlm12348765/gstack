# gstack 中文翻译目录

本目录镜像 gstack 全部 skill 相关 Markdown 文件的简体中文版本，并尽量保持与原仓库一致的相对路径结构。

收录范围：
- 根目录 `AGENTS.md`、`SKILL.md`、`SKILL.md.tmpl`
- 根目录核心设计文档 `ARCHITECTURE.md`、`BROWSER.md`、`CLAUDE.md`、`ETHOS.md`
- 各 skill 子目录中的 `SKILL.md` 与 `SKILL.md.tmpl`
- `docs/skills.md`
- `review/` 与 `qa/` 下的 skill 支撑 Markdown 文档

翻译规则：
- 保留 Markdown 结构、frontmatter 键名、命令、路径、文件名、flag、代码标识符、模板变量与 slash command 名称
- 翻译自然语言正文、说明、提示、注释与用户可见文案
- 代码块中的命令和代码默认保持原样，只翻译其中的自然语言说明

说明：
- 若某些自动生成的 `SKILL.md` 未单独重译，则使用对应 `SKILL.md.tmpl` 的中文版本进行镜像补齐，以保证目录覆盖完整。
