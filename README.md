# Linux-devops-handbook

Linux 运维实践手册：从 Linux 基础到 Docker / Kubernetes 的系统化学习笔记与脚本。

## 目录结构

```
Linux-devops-handbook/
├── assets/                     # 截图、拓扑图、配图
├── codes/                      # 配置片段，yaml、helm-values、compose 等
├── docs/                       # ✨ 所有学习笔记 md 文档（重点）
│   ├── linux/                  # Linux 基础、命令、调优
│   ├── shell/                  # shell 语法笔记
│   ├── docker/
│   └── k8s/
├── scripts/                    # ✅ 可直接运行的 shell/python 脚本，与 docs 分离
│   ├── shell/
│   └── python/
├── .gitignore
└── README.md
```

## 内容规划

- `docs/`：核心学习笔记，按主题分目录存放
- `codes/`：笔记引用的配置片段（yaml、helm-values、compose 等）
- `scripts/`：可直接运行的脚本，与文档解耦
- `assets/`：文档配图、截图、拓扑图

## 使用

```bash
# 克隆仓库
git clone https://github.com/lai000000/Linux-devops-handbook.git
```
