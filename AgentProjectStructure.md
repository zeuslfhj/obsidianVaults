# Code Base使用

## 文件说明

- **AGENTS.md** 这个是OpenAI(codex) 与 gmini 共同约定的协议，用于读取整体的内容
- **CLAUDE.md** 这个是CC用来读取的，如果希望和`AGENTS.md` 同步，可以直接使用 `@AGENTS.md` 来进行引用

## 文件夹说明

```
my-project/
├── AGENTS.md
├── CLAUDE.md
├── README.md
├── docs/                     # 用于存放项目架构文档
│   ├── architecture.md       # 项目架构文件
│   ├── api.md                # API描述
│   ├── coding_standards.md
│   └── ...（其他分模块文档）
├── schema/
│   └── metadata.yaml         # 项目元数据模式
├── src/                      # 源代码
│   ├── ...
│   └── (项目模块)
├── examples/                 # 示例代码
│   └── example_usage.py
├── tests/                    # 单元/集成测试
│   └── ...
└── .github/workflows/
    └── ci.yml               # CI 流程（构建、测试、文档检查）

```

## 工具说明

### Code Map工具

- **aider repo map**: aider 不走“整仓库全文打包”的路子，而是生成一个简洁的仓库地图，包含重要类、函数、签名和关键定义
	- aider需要主动控制添加的文件列表
- **Repomix**:  Repomix is a powerful tool that packs your entire repository into a single, AI-friendly file. 
  - Warn: 单文件尺寸是一个值得注意的点，且output是一个xml文件

## Claude Skill推荐
- **Planning with files**: 规划、进度和知识都写进 Markdown 文件 [git](https://github.com/OthmanAdi/planning-with-files)
- **superpowers**: 打包了 20 多个可组合的 Skill [git](https://github.com/obra/superpowers) ,可以尝试**brainstorming** worktress, test-driven-development
- claude.md manager: 维护claude 的a