---
kind: build_system
name: QwenPaw 构建系统
category: build_system
scope:
    - '**'
source_files:
    - Makefile
    - scripts/docker_build.sh
    - deploy/Dockerfile
    - console/package.json
    - console/vite.config.ts
    - console/src-tauri/Cargo.toml
    - console/src-tauri/tauri.conf.json
    - pyproject.toml
    - setup.py
    - scripts/pack/build_macos.sh
    - scripts/pack-tauri/build_pyinstaller.sh
    - scripts/pack-tauri/qwenpaw.spec
    - scripts/pack-tauri/stage_python_runtime.py
    - scripts/pack-tauri/generate_update_manifest.py
---

## 构建系统概述

QwenPaw 采用多平台、多目标的复杂构建系统，支持桌面应用（Tauri）、容器化部署（Docker）和传统分发（conda-pack）。构建系统主要由以下部分组成：

- **前端构建**：使用 Vite + React + TypeScript 构建控制台界面
- **后端构建**：使用 PyInstaller 将 Python 后端打包为独立可执行文件
- **桌面应用构建**：使用 Tauri 框架创建跨平台桌面应用
- **容器化构建**：使用多阶段 Dockerfile 构建容器镜像
- **传统分发**：使用 conda-pack 打包为可移植环境

## 核心构建流程

### 1. 桌面应用构建流程

```bash
# macOS 打包
bash scripts/pack/build_macos.sh

# Tauri 后端构建
bash scripts/pack-tauri/build_pyinstaller.sh
```

构建流程包括：
- 使用 Vite 构建前端控制台并输出到 `console/dist`
- 使用 PyInstaller 将 Python 后端打包为独立可执行文件
- 集成前端资源到后端包中
- 为 Windows 平台构建 NSIS 安装程序
- 为 macOS 平台构建 `.app` 应用包

### 2. 容器化构建流程

```bash
bash scripts/docker_build.sh [IMAGE_TAG]
```

Dockerfile 采用多阶段构建：
- `console-builder` 阶段：构建前端控制台
- `runtime` 阶段：安装 Python 运行时和依赖，复制前端构建产物
- 支持构建参数控制包含/排除特定渠道

### 3. 传统分发构建

使用 `conda-pack` 创建可移植的 Python 环境，打包为 `.tar.gz` 文件，可在不同环境中解压运行。

## 关键构建配置

### PyInstaller 规范文件 (qwenpaw.spec)

定义了后端可执行文件的打包规则，包括：
- 数据文件收集（技能、模型、安全规则等）
- 隐藏导入模块（动态加载的模块）
- 元数据收集（包版本信息）
- 生成两个可执行文件：`qwenpaw-backend`（GUI模式）和 `qwenpaw`（CLI模式）

### Tauri 配置 (tauri.conf.json)

定义了桌面应用的配置，包括：
- 窗口属性（尺寸、标题）
- 安全策略（CSP）
- 资源文件（后端可执行文件、Python运行时）
- 更新机制配置

### Vite 配置 (vite.config.ts)

定义了前端构建配置，包括：
- 代码分割策略（按功能模块分割）
- 测试配置（mock外部依赖）
- 开发服务器代理设置

## 构建约定和规则

### 开发者应遵循的规则

1. **前端构建**：
   - 修改前端代码后必须重新运行 `npm run build` 以更新 `console/dist`
   - 测试覆盖率阈值：语句 5%，分支 4%，函数 3%，行 5%
   - 避免在 `node_modules` 中直接修改第三方库

2. **后端打包**：
   - PyInstaller spec 文件中的 `hiddenimports` 必须包含所有动态导入的模块
   - 数据文件必须通过 `datas` 显式添加到打包中
   - 新增依赖需要检查是否需要元数据收集

3. **桌面应用**：
   - Tauri 资源目录包含三个关键二进制文件：`qwenpaw-backend`、`python-runtime`、`node-runtime`
   - 更新 `tauri.conf.json` 时需同步更新构建脚本中的路径

4. **容器化**：
   - Dockerfile 使用预设的基础镜像，可通过构建参数覆盖
   - 支持通过环境变量控制包含/排除特定渠道
   - 构建时自动安装前端构建产物

5. **版本管理**：
   - 版本号从 `src/qwenpaw/__version__.py` 统一获取
   - Tauri 更新清单使用语义化版本格式
   - 构建脚本会自动提取版本信息用于命名输出文件

### 构建输出位置

- 桌面应用：`dist/QwenPaw.app` (macOS) 或 `dist/qwenpaw-setup.exe` (Windows)
- Docker 镜像：根据构建命令标签命名
- PyInstaller 包：`dist/pyinstaller/qwenpaw-backend`
- conda-pack 包：`dist/qwenpaw-env.tar.gz`

该构建系统设计考虑了多平台分发需求，通过模块化的构建脚本实现了灵活的打包策略。