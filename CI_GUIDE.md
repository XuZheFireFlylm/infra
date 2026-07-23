# GitHub Actions CI/CD 配置指南

由于当前 PAT 缺少 `workflow` scope，workflow 文件需要手动创建。请按以下步骤操作：

## 方法：通过 GitHub Web 界面创建

1. 打开 https://github.com/XuZheFireFlylm/scheduler
2. 点击 **Add file** → **Create new file**
3. 文件路径填写：`.github/workflows/ci.yml`
4. 复制下方内容粘贴进去
5. 点击 **Commit changes...** → **Commit directly to the `master` branch** → **Commit changes**

## ci.yml 内容

```yaml
name: CI · Scheduler
on:
  push:
    branches: [main]
    paths: ["scheduler/**"]
  pull_request:
    branches: [main]
    paths: ["scheduler/**"]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"
          cache-dependency-path: scheduler/requirements.txt
      - name: Install deps
        run: pip install -r scheduler/requirements.txt
      - name: Lint
        run: pip install ruff && ruff check scheduler/app/
      - name: Test
        run: |
          docker compose -f scheduler/docker/docker-compose.yml up -d postgres redis
          sleep 5
          pytest scheduler/tests/ -v --tb=short

  docker-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        run: |
          docker build -f scheduler/docker/Dockerfile \
            -t ghcr.io/xuzhefireflylm/scheduler:${{ github.sha }} \
            -t ghcr.io/xuzhefireflylm/scheduler:latest \
            scheduler/
          docker push --all-tags ghcr.io/xuzhefireflylm/scheduler
```

## 永久解决方案：生成带 workflow scope 的 Token

在 GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens

1. 点击 **Generate new token (classic)**
2. 勾选 scopes: `repo` + **`workflow`**
3. 保存 token 后更新本地的 `infra/.env` 或 CI 配置

---
*本文件由 Firefly Infra Bot 自动生成于 2026-07-23*
