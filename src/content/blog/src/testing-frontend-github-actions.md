---
slug: testing-frontend-github-actions
title: 前端仓库如何测试与触发 GitHub Actions
description: 详细介绍前端项目中测试和触发 GitHub Actions 工作流的多种方式，包括手动触发、API 触发、本地测试以及最佳实践。
pubDate: 2026-08-15T00:00
image: /image/image4.webp
badge: DevOps
draft: false
categories:
- DevOps
tags:
- GitHub Actions
- CI/CD
- 前端
- DevOps
- 自动化
---

## 引言

在前端项目中，GitHub Actions 已经成为 CI/CD 的核心工具 —— 从代码提交后的自动构建、测试、lint 检查，到合并后的自动部署，几乎无处不在。但工作流写好后，如何高效地**测试**它是否按预期运行？如何在需要时**手动触发**一个特定的 workflow？本文将系统性地介绍前端仓库中测试和触发 GitHub Actions 的多种方式。

## 1. 手动触发：workflow_dispatch

这是最常用的手动触发方式。在 workflow 文件中定义 `workflow_dispatch` 事件，并可以添加自定义输入参数：

```yaml
# .github/workflows/deploy-preview.yml
name: Deploy Preview

on:
  workflow_dispatch:
    inputs:
      environment:
        description: '部署环境'
        required: true
        type: choice
        options:
          - staging
          - production
        default: 'staging'
      branch:
        description: '部署分支'
        required: false
        type: string
        default: 'main'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.inputs.branch }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install & Build
        run: |
          npm ci
          npm run build

      - name: Deploy to ${{ github.event.inputs.environment }}
        run: |
          echo "Deploying to ${{ github.event.inputs.environment }}..."
          # 实际部署脚本
```

配置完成后，在 GitHub 仓库的 **Actions** 标签页中，左侧选择该 workflow，就会出现 **"Run workflow"** 按钮，你可以选择分支并填写参数后手动运行。

### 适用场景

- 手动部署到预览 / 生产环境
- 触发数据迁移脚本
- 清理缓存或临时资源
- 需要在非 push/PR 事件时运行的一次性任务

## 2. API 触发：repository_dispatch

当你需要从**外部系统**（如 CMS、其他仓库、定时任务平台）触发 workflow 时，`repository_dispatch` 是最佳选择。

### 2.1 定义接收端 workflow

```yaml
# .github/workflows/content-sync.yml
name: Content Sync

on:
  repository_dispatch:
    types: [content-updated, asset-changed]

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Handle content update
        if: ${{ github.event.action == 'content-updated' }}
        run: |
          echo "Content type: ${{ github.event.client_payload.content_type }}"
          echo "Changed files: ${{ github.event.client_payload.files }}"
          # 执行内容同步逻辑

      - name: Handle asset change
        if: ${{ github.event.action == 'asset-changed' }}
        run: |
          echo "Asset: ${{ github.event.client_payload.asset_name }}"
          # 执行资源更新逻辑
```

### 2.2 从外部触发

使用 GitHub Personal Access Token (PAT) 调用 API：

```bash
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/OWNER/REPO/dispatches \
  -d '{
    "event_type": "content-updated",
    "client_payload": {
      "content_type": "blog",
      "files": ["post-1.md", "post-2.md"]
    }
  }'
```

在前端项目中，你也可以在 Node.js 脚本中触发：

```typescript
// scripts/trigger-build.ts
async function triggerWorkflow(eventType: string, payload: Record<string, unknown>) {
  const response = await fetch(
    `https://api.github.com/repos/${owner}/${repo}/dispatches`,
    {
      method: 'POST',
      headers: {
        'Authorization': `token ${process.env.GITHUB_TOKEN}`,
        'Accept': 'application/vnd.github.v3+json',
      },
      body: JSON.stringify({
        event_type: eventType,
        client_payload: payload,
      }),
    }
  );

  if (response.status === 204) {
    console.log('✅ Workflow triggered successfully');
  } else {
    console.error('❌ Failed to trigger workflow:', response.status);
  }
}

// 使用示例
triggerWorkflow('content-updated', {
  content_type: 'blog',
  files: ['post-1.md'],
});
```

### 适用场景

- 从 CMS / Headless CMS 触发构建
- 跨仓库联动（如设计系统更新后触发下游项目重建）
- 定时任务平台（Cronjob.org、自建调度器）触发
- Webhook 接收器

## 3. 本地测试：act

[nektos/act](https://github.com/nektos/act) 是一个在本地运行 GitHub Actions 的工具，可以极大加速 workflow 的开发调试。

### 3.1 安装

```bash
# macOS (Homebrew)
brew install act

# Windows (Chocolatey)
choco install act-cli

# Linux (curl)
curl -s https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash
```

### 3.2 基本使用

```bash
# 运行默认的 push 事件触发的工作流
act

# 运行指定 workflow
act -W .github/workflows/deploy-preview.yml

# 模拟 pull_request 事件
act pull_request

# 运行指定 job
act -j build

# 模拟 workflow_dispatch 事件并传入 inputs
act workflow_dispatch \
  -W .github/workflows/deploy-preview.yml \
  --input environment=staging
```

### 3.3 前端项目的典型测试流程

```bash
# 1. 测试 CI workflow（lint + test + build）
act push -W .github/workflows/ci.yml

# 2. 测试部署 workflow（模拟 workflow_dispatch）
act workflow_dispatch \
  -W .github/workflows/deploy.yml \
  --input environment=preview

# 3. 查看详细日志
act -v push
```

> **注意**：`act` 默认使用中型镜像，首次运行会下载 Docker 镜像。某些 GitHub 特定功能（如 `GITHUB_TOKEN` 自动注入）需要在本地模拟，可以通过 `-s GITHUB_TOKEN=your_token` 传入。

### 适用场景

- 本地开发 workflow 时快速验证
- 在提交前确认 workflow 语法正确
- 调试复杂的 job 依赖和条件逻辑

## 4. 前端项目中常见的 Workflow 测试策略

### 4.1 分层测试

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # 第一层：代码质量（快速失败）
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  # 第二层：单元测试（并行运行）
  test:
    needs: lint
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test -- --shard=${{ matrix.shard }}/${{ strategy.job-total }}

  # 第三层：构建验证
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - name: Check bundle size
        run: |
          npm run analyze-bundle
          # 如果包体积超过阈值则失败
```

### 4.2 使用 concurrency 避免重复运行

```yaml
name: Deploy Preview

on:
  pull_request:
    types: [opened, synchronize, reopened]

# 同一 PR 的新提交会取消旧的运行
concurrency:
  group: preview-${{ github.ref }}
  cancel-in-progress: true

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    # ...
```

### 4.3 条件触发与路径过滤

```yaml
name: Visual Regression

on:
  pull_request:
    paths:
      - 'src/**/*.css'
      - 'src/**/*.scss'
      - 'src/components/**'

jobs:
  visual-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run visual regression tests
        run: npm run test:visual
```

## 5. 调试技巧

### 5.1 启用 Debug Logging

在 workflow 运行时，可以通过设置 secret 来启用详细日志：

```bash
# 在仓库 Settings > Secrets and variables > Actions 中添加：
ACTIONS_RUNNER_DEBUG = true     # Runner 诊断日志
ACTIONS_STEP_DEBUG = true       # Step 诊断日志
```

### 5.2 使用 tmate 进行交互式调试

```yaml
- name: Setup tmate session
  uses: mxschmitt/action-tmate@v3
  # 仅在 workflow_dispatch 且 debug 模式启用时
  if: ${{ github.event_name == 'workflow_dispatch' && inputs.debug_enabled }}
```

这会在 workflow 运行到该步骤时启动一个 SSH 会话，你可以直接连接到 runner 环境进行调试。

### 5.3 缓存策略测试

```yaml
- name: Cache node_modules
  uses: actions/cache@v4
  with:
    path: |
      node_modules
      ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# 测试缓存是否命中：在日志中查找 "Cache hit"
```

## 6. 常见问题与解决方案

### Q: workflow_dispatch 的分支选项中看不到我的分支？

GitHub 的 workflow 文件必须存在于**默认分支**（通常是 `main`）上，才能在 Actions 页面中看到手动触发按钮。如果你在 feature 分支上定义了新的 workflow，需要先合并到默认分支。

### Q: repository_dispatch 返回 204 但没有触发 workflow？

检查以下几点：
1. token 是否有 `repo` 权限（对于私有仓库）
2. `event_type` 是否与 workflow 中定义的 `types` 匹配
3. workflow 文件是否已存在于默认分支

### Q: act 运行时缺少某些 GitHub 上下文？

`act` 不提供完整的 GitHub 上下文。可以使用 `--env` 或 `--secret` 手动注入：

```bash
act -s GITHUB_TOKEN=your_token \
    --env GITHUB_REPOSITORY=owner/repo \
    --env GITHUB_REF=refs/heads/main
```

## 总结

| 方式 | 触发者 | 适用场景 | 复杂度 |
|------|--------|----------|--------|
| `workflow_dispatch` | 手动（GitHub UI） | 手动部署、一次性任务 | ⭐ |
| `repository_dispatch` | API 调用 | 外部系统触发、跨仓库联动 | ⭐⭐ |
| `act` | 本地命令行 | 本地开发调试 | ⭐⭐ |
| `push` / `pull_request` | Git 事件 | 常规 CI/CD | ⭐ |

掌握这些触发与测试方式，可以让你在前端项目的 CI/CD 流程中更加游刃有余。建议优先使用 `act` 在本地验证 workflow 逻辑，再通过 `workflow_dispatch` 在 GitHub 上进行端到端测试，最后结合 `repository_dispatch` 实现自动化联动。

---

> **参考链接**
> - [GitHub Actions - events that trigger workflows](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows)
> - [nektos/act - Run your GitHub Actions locally](https://github.com/nektos/act)
> - [Repository Dispatch API](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event)