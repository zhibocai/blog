---
title: Django 开发环境搭建
date: 2022-06-05 15:29:36
tags: Django
categories: 技术
---

在开始开发 Django 项目前, 需要安装好开发环境:

- 安装 Python
- 安装虚拟环境(可选)
- 安装 Django


## 安装 Python

在 [python.org](https://www.python.org/downloads/) 上下载对应操作系统的安装包

> Q: 如何确定安装哪一个 Python 版本?
> A: 通过 [此页面](https://docs.djangoproject.com/en/4.1/faq/install/#faq-python-version-support) 确认 Django 支持的 Python 版本


## 创建虚拟环境

(可选的): 当我们在做不同的项目开发时, 会在 Python 环境中安装各种依赖, 而不同项目的之前可能会有依赖间版本冲突, 如果希望开发的每个项目都有一个独立且干净, 就需要使用 "虚拟环境".

创建虚拟环境很简单:

```bash
# Linux
python -m venv /path/to/mysite/.venv

# Windows
python -m venv c:\path\to\mysite\.venv
```

> 通常虚拟环境会创建在项目目录下, 并以 `.venv` 命名, 这并不是强制要求的, 而是项目工程化的建议

创建后, 需要激活虚拟环境:

```bash
# Linux
source /path/to/mysite/.venv/bin/activate

# Windows
c:\path\to\mysite\.venv\Scripts\activate.bat
```

交互控制台能看到虚拟环境名在路径上时, 即表示激活成功:

```bash
# Linux
(.venv) [# /path/to/mysite]:

# Windows
(.venv) C:\path\to\mysite>
```

退出虚拟环境, 使用 deactivate 命令:

```bash
# Exit venv
deactivate
```

更多 venv 的使用, 可以参考 [官方文档](https://docs.python.org/zh-cn/3/library/venv.html#module-venv)

## 安装 Django

> 如果使用虚拟环境, 请记得先 **激活** 虚拟环境

Django 使用 pip 命令进行安装:

```bash
# Install latest version
pip install Django
```

建议先查看 [Django 支持的 Python 版本](https://docs.djangoproject.com/en/4.1/faq/install/#faq-python-version-support) 从而选择合适的版本进行安装:

```bash
# Install 4.1 version
pip install "Django==4.1"
```

----

后面, 我们来了解下一个 Django 项目的模块组成
