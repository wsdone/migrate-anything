# migrate-anything

AI 驱动的跨平台软件迁移工具。将软件源代码在 macOS、Windows、Linux 之间迁移。

**[English](README.md)**

> 灵感来自 [cli-anything](https://github.com/HKUDS/CLI-Anything) —— 一个为 GUI 应用构建 CLI 接口的 AI Agent 项目。migrate-anything 将同样的 Agent 驱动方法论应用于跨平台代码迁移。

## 功能

migrate-anything 分析软件源代码，识别平台特定依赖，并将代码改写为目标操作系统的版本。覆盖范围：

- **系统层代码**：文件 I/O、线程、进程、IPC、网络
- **图形 API**：Metal、DirectX、OpenGL、Vulkan
- **UI 框架**：AppKit、Win32、GTK、Qt
- **构建系统**：Xcode、Visual Studio → CMake/Meson
- **架构迁移**：平台抽象层、重写边界设计
- **语言迁移**：Swift/ObjC → C++/Rust（需要时）

## 安装

**一键安装**（推荐）：
```bash
/plugin marketplace add wsdone/migrate-anything
/plugin install migrate-anything@migrate-anything
```

**手动安装** — 添加到 `~/.claude/settings.json`：
```json
{
  "enabledPlugins": {
    "migrate-anything@migrate-anything": true
  },
  "extraKnownMarketplaces": {
    "migrate-anything": {
      "source": {
        "source": "github",
        "repo": "wsdone/migrate-anything"
      }
    }
  }
}
```

## 命令

| 命令 | 说明 | 修改代码？ |
|------|------|-----------|
| `/migrate-anything` | 完整迁移流程（阶段 0-7） | 是 |
| `/migrate-anything:analyze` | 只读分析平台依赖 | 否 |
| `/migrate-anything:test` | 运行测试并更新报告 | 否 |
| `/migrate-anything:validate` | 对照标准验证迁移结果 | 否 |
| `/migrate-anything:refine` | 迭代改进迁移覆盖率 | 是 |
| `/migrate-anything:map` | 显示平台间 API 对应关系 | 否 |

## 典型工作流

```
第 1 步：分析（可选，只读）
  /migrate-anything:analyze /path/to/source

第 2 步：迁移
  /migrate-anything:migrate-anything /path/to/source

第 3 步：精炼循环（反复执行，直到覆盖率达标）
  /migrate-anything:refine /path/to/source
  → Refine 自动完成编译、测试、提交，无需单独跑 test

第 4 步：验证
  /migrate-anything:validate /path/to/source
```

### 使用 Ralph 自动精炼

使用 [ralph-wiggum](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum) 自动循环执行 refine：

```bash
# 自动精炼，直到覆盖率达标
/ralph-wiggum:ralph-loop /migrate-anything:refine /path/to/source
```

### 精炼重点

refine 命令的第二个参数接受自然语言描述：

```bash
# 聚焦某个技术领域
/migrate-anything:refine /path/to/source "图形渲染管线"

# 设定覆盖率目标
/migrate-anything:refine /path/to/source "功能覆盖率不到90%，继续迭代"

# 报告 UI bug 让 agent 修复
/migrate-anything:refine /path/to/source "右侧有白边，无法拖拽调整窗口大小"
```

**语言迁移** — 加 `--lang <语言>` 强制指定目标语言（如 Swift → Rust）。仅当源语言在目标平台不可用时才需要。大多数迁移不需要此参数。

```bash
/migrate-anything:migrate-anything /path/to/source --lang rust
```

## 示例场景

**UI 迁移：**
| 迁移内容 | 源平台 | 目标平台 | 复杂度 |
|---------|--------|---------|--------|
| macOS 原生应用 | macOS（AppKit + CoreGraphics） | Linux（GTK 4） | 困难 |
| Windows 桌面应用 | Windows（WPF/WinForms） | Linux（Qt） | 困难 |
| Qt 应用 + 原生集成 | Linux（Qt + D-Bus + udev） | macOS（Qt + 原生 API） | 中等 |

**图形 API 迁移：**
| 迁移内容 | 源平台 | 目标平台 | 复杂度 |
|---------|--------|---------|--------|
| GPU 加速终端 | macOS（Metal + Swift） | Linux（Vulkan） | 困难 |
| 游戏引擎 | Windows（DirectX + 自定义平台层） | Linux（Vulkan） | 困难 |

**系统 / 架构迁移：**
| 迁移内容 | 源平台 | 目标平台 | 复杂度 |
|---------|--------|---------|--------|
| 系统工具 | Windows（Win32 API） | Linux（POSIX） | 简单 |
| 内核扩展 | macOS（kext） | Linux（eBPF） | 困难 |
| C 库平台 I/O | Windows（Win32 文件 API） | Linux（POSIX） | 简单 |
| 可移植运行时应用 | 任意（Electron/JVM/Python/Go） | 任意 | 简单 |

## 迁移方法论

迁移遵循 8 阶段方法论（详见 MIGRATE.md）：

1. **平台检测** — 识别源平台和目标平台
2. **依赖分析** — 构建平台依赖清单（PDI）+ 功能清单（FI）
3. **可行性评估** — 困难迁移的门控检查
4. **架构设计** — 规划每个依赖的迁移策略
5. **构建系统迁移** — 转换构建系统
6. **代码迁移** — 实现替换（简单 → 困难）
7. **测试** — 验证编译，编写迁移测试
8. **文档与验证** — 生成报告，构建并创建精炼清单

## 许可证

MIT
