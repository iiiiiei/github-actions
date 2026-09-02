# 仓库同步调度中心

[简体中文](./README.md)｜[English](./README_EN.md)

本仓库存放可复用的 GitHub Actions 工作流，用于将源仓库自动镜像同步到目标仓库。

## 架构原理

```
源仓库/
├── .github/
│   └── workflows/
│       └── sync.yml          # 触发器：薄 wrapper（约 14 行）
│           ├── 监听 push → main
│           └── 调用调度中心的 sync-core.yml
│
调度中心仓库/
├── .github/
│   └── workflows/
│       └── sync-core.yml     # 核心逻辑（维护这一份即可）
│           ├── Job 1: 同步代码与标签
│           │   ├── 清理 sync.yml（避免目标残留触发器）
│           │   ├── fetch 目标 main
│           │   ├── 版本检查（merge-base 严格校验）
│           │   ├── git push --force-with-lease
│           │   └── git push --tags
│           ├── Job 2: 同步 Releases
│           │   ├── 获取源 Releases 列表
│           │   ├── 跳过目标已存在的 Release
│           │   ├── 创建 Release（元数据 + Assets）
│           │   └── 下载并重新上传 Assets
│           ├── Job 3: 同步 Wiki
│           │   ├── clone 源 Wiki
│           │   ├── clone 目标 Wiki（不存在则初始化）
│           │   ├── rsync --delete 完全镜像
│           │   └── git push
│           └── Job 4: 同步 Packages (GHCR)
│               ├── 查询源仓库 GHCR Packages
│               ├── docker pull 源镜像
│               ├── docker tag → 目标仓库
│               └── docker push
│
目标仓库/
├── main                      # 纯净镜像，无 sync.yml
├── Tags                      # 与源仓库完全一致
├── Releases                  # 与源仓库完全一致（含 Assets）
├── Wiki/                     # 与源仓库 Wiki 完全一致
└── Packages (GHCR)           # 与源仓库镜像一致
```

### 核心逻辑

1. **触发**：源仓库的 `main` 分支发生 `push`，或手动触发 `workflow_dispatch`
2. **调用**：源仓库通过 `uses` 引用本仓库的 Reusable Workflow
3. **清理**：推送前自动移除 `.github/workflows/sync.yml`，避免目标仓库残留触发器
4. **版本检查**：
   - 目标为空 → 直接推送
   - 目标与源相同 → 跳过
   - 目标严格落后于源 → 安全推送（`--force-with-lease`）
   - 目标领先或分叉 → **显式报错终止**，防止覆盖
5. **Tags 同步**：`git push --tags`，保持标签一致
6. **Releases 同步**：通过 GitHub API 复制 release 元数据，下载并重传 assets；**已存在的 Release 不会覆盖**
7. **Wiki 同步**：使用 `rsync --delete` 完全镜像；目标无 Wiki 时自动初始化
8. **Packages 同步**：通过 `docker pull/tag/push` 同步 GHCR 容器镜像；支持用户和组织两种 package 归属

### 安全机制

- **PAT 隔离**：Token 由调用方仓库的 Secret 注入，调度中心不存储任何凭证
- **防覆盖**：`--force-with-lease` + `merge-base` 双重校验，拒绝任何可能丢失提交的推送
- **Release 防覆盖**：同步前检查目标是否已有同名 Release，存在则跳过
- **失败即停**：各 Job 独立运行，任一环节失败不影响其他，但错误会显式上报

## 文件结构

```
.github/workflows/
└── sync-core.yml          # 核心同步逻辑（维护这一份即可）
```

## 前置条件

### PAT 权限要求

| 同步内容 | 所需 PAT 权限 |
|---------|--------------|
| 代码 + Tags + Wiki + Releases | `repo` |
| Packages (GHCR) | `repo` + `write:packages` + `read:packages` |

> 若仓库无 GHCR Packages，只需 `repo` 权限即可。

## 新增仓库配置步骤

### 1. 配置 Secret

在源仓库的 **Settings → Secrets and variables → Actions** 中添加：

| Name | Value |
|------|-------|
| `<PAT_SECRET名称>` | 你的 Personal Access Token（权限见上文"前置条件"） |

### 2. 在源仓库创建触发器

创建文件：`.github/workflows/sync.yml`

```yaml
# 自动同步工作流
# 作用：当 main 分支有更新时，自动镜像推送到目标仓库
# 注意：此文件仅在源仓库生效，目标仓库不会保留此文件

name: Sync to Target

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  sync:
    uses: <调度中心仓库>/.github/workflows/sync-core.yml@main
    with:
      target_repo: <目标仓库全名>
    secrets:
      TARGET_PAT: ${{ secrets.<PAT_SECRET名称> }}
```

> `<目标仓库全名>` 格式为 `所有者/仓库名`，例如 `someone/my-repo`。

### 3. 验证

推送一次 `main` 分支，或手动触发 Actions，观察日志确认同步成功。

## 维护说明

- **修改同步逻辑**：只需更新本仓库的 `sync-core.yml`，所有引用它的仓库自动生效
- **新增仓库**：仅需复制上述薄 wrapper，修改 `target_repo` 即可
- **目标仓库改名**：修改对应源仓库 `sync.yml` 中的 `target_repo`
- **源仓库改名**：无需任何改动，`${{ github.repository }}` 自动识别
- **排除某项同步**：若不需要 Releases/Packages/Wiki 中的某一项，当前版本仍会执行但会自动跳过（无资源时静默退出），不影响整体流程
