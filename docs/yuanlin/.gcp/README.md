# .gcp

## 概述

此目录包含 Google Cloud Platform (GCP) 相关的构建与部署配置文件，包括 Docker 容器镜像定义和 Cloud Build 工作流配置。用于支持项目在 GCP 环境中的开发构建和发布流程。

## 目录结构

```
.gcp/
├── Dockerfile.development              # 开发环境 Docker 镜像定义
├── Dockerfile.development.dockerignore  # 开发环境构建时的文件忽略规则
├── Dockerfile.gemini-code-builder       # Gemini Code Builder 专用 Docker 镜像定义
├── development-worker.yml               # 开发环境 Cloud Build 工作流配置
└── release-docker.yml                   # 发布阶段 Docker 镜像的 Cloud Build 工作流配置
```
