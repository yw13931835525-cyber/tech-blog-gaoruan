---
layout: post
title: "GitHub Actions CI/CD完全指南：从自动化测试到生产部署"
subtitle: "手把手配置多环境部署、Docker镜像构建、SonarQube代码分析、K8s部署的完整流水线，支持monorepo和矩阵构建"
date: 2026-04-13
category: devtools
category_name: 🛠️ 开发工具
tags: [GitHub Actions, CI/CD, Docker, Kubernetes, DevOps, 持续集成, 自动化部署, DevSecOps]
excerpt: "本文是GitHub Actions CI/CD的全面教程，涵盖基础配置、矩阵构建、多环境部署、Docker镜像推送、Kubernetes部署等实战技能，提供可直接使用的Workflow模板。"
keywords: GitHub Actions教程, CI/CD配置, Docker CI, Kubernetes部署, DevOps, 自动化测试, GitHub Workflow
---

# GitHub Actions CI/CD完全指南：从自动化测试到生产部署

GitHub Actions 是 GitHub 内置的 CI/CD 工具，无需额外费用即可使用。本文将带你从零掌握 GitHub Actions 的完整配置。

## 基础配置

### 1. Workflow 结构

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # 每天凌晨2点执行
  workflow_dispatch:  # 支持手动触发

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run linter
        run: flake8 . --max-line-length=120

  test:
    needs: lint
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Run tests
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
        run: |
          pip install -r requirements.txt
          pytest tests/ -v --cov=src --cov-report=xml
```

### 2. 矩阵构建

```yaml
jobs:
  build:
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']
        operating-system: [ubuntu-latest, windows-latest]
        exclude:
          - python-version: '3.12'
            operating-system: windows-latest  # 排除不兼容组合
      fail-fast: false
    
    runs-on: ${{ matrix.operating-system }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Build
        run: pip install build && python -m build
```

### 3. Docker 多平台构建

```yaml
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### 4. K8s 部署

```yaml
jobs:
  deploy:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v4
      
      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kube_config
          echo "KUBECONFIG=$(pwd)/kube_config" >> $GITHUB_ENV
      
      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f k8s/ --namespace production
          kubectl rollout status deployment/myapp --namespace production
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            --namespace production
```

---

## 💼 需要DevOps服务？

**联系方式：13315868800（微信同号）**

---

## ❤️ 支持我们

如果你觉得我们的工具和教程有帮助，欢迎赞助开源项目：

[![Sponsor](https://github.com/sponsors/yw13931835525-cyber/widgets/badge.svg)](https://github.com/sponsors/yw13931835525-cyber)

**加密货币赞助：**
- **ETH/ERC-20：** `0x6FCBd5d14FB296933A4f5a515933B153bA24370E`
- **BTC：** `bc1pfvewxym02a4hwupn9f0d0x7h26uuf2jhjkcz3ysd9ef6acjttrmqwex66h`
