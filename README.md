# Agent Tools Library

收集与分享 AI Agent 相关的工具集合，包括 Claude Code skills、辅助软件、自动化脚本等。

## 目录结构

```
Agent-Tools-Library/
├── skills/         # Claude Code / AI Agent skills
└── README.md
```

## Skills

### [windows-diag-bat](./skills/windows-diag-bat)

生成发给 Windows 同事运行的"单文件诊断/采集 bat"。脚本跑完会把报告路径自动复制到剪贴板、用资源管理器打开并选中文件、显示醒目横幅提示同事发回来。

**使用方法：** 把 `skills/windows-diag-bat/` 整个目录放到你的 Claude Code skills 目录下（如 `~/.claude/skills/`）即可。

## 贡献

欢迎提交 PR 添加新工具。每个工具放在对应分类目录下，并在 README 中追加简介。

## License

MIT
