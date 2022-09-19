---
title: Django 项目结构
date: 2022-06-9 11:36:20
tags: Django
---

让我们来学习一下 Django 的项目结构: Django 的项目并不需要手动创建, 官方文档会推荐你使用 Django 提供的 **脚手架** 创建:

```shell
# 在当前目录下创建一个名为 myproj 的 Django 项目
django-admin startproject myproj
```

激活了 **虚拟环境**  的情况, 建议使用 "python -m django" 来替代 "django-admin":

```shell
# python -m django 等同于 django-admin
python -m django startproject myproj
```

> 能够使用 django-admin 命令是需要 Python / 虚拟环境安装时, 成功注册 `$PATH` 指向正确环境的 `bin` 目录, 这个注册并不一定会成功. `python -m django` 则确保一定能正确执行

对于已经提前创建好工作空间(比如初始化虚拟环境), 再使用 `startproject` 就有个小问题, 默认执行会产生 **多一级** 目录:

```
mysite/              # 已创建好的工作空间
	.venv/
	myproj/          # 多了这一级
		- myproj/
		- manage.py
```

我们可以在 `startproject` 时追加 `directory` 参数:

```shell
python -m django startproject myproj /path/to/mysite
```

那么项目结构应当是这样的:

```dir
mysite/              # 已创建好的工作空间
	- .venv/         # 虚拟环境
	- myproj/
		- asgi.py
		- settings.py
		- urls.py
		- wsgi.py
	- manage.py
```

> 如果执行 startproject 会发生错误, 提示 **"xxx conflicts with the name of an existing Python module and cannot be used as a project name. Please try another name."** 是因为项目名会被作为 Python 的模块导入, 即代码中会 **"import myproj.urls"** 所以请确保项目名不会跟模块和包冲突.

那么, 通过 **脚手架**, 我们得到了一个 Django 项目的标准结构, 现在来看看这个项目的基本组成有什么:

```dir
- mysite
	- .venv/
	- myproj/
		- asgi.py
		- settings.py
		- urls.py
		- wsgi.py
	- manage.py
```

**mysite/**: 你的项目根目录, 存放项目所有的代码, 可以是任意合法字符

**.venv/**: 虚拟环境

**myproj/**: Django 项目的 Python package, 此目录的代码会通过 "目录名.模块名" 在代码中导入, **请注意不能与环境中的其他包冲突**

**myproj/settings.py**: 项目的全局配置, 如: 数据库, 缓存等等

**myproj/urls.py**: 项目的根路由声明, 是 web 程序路由的分发入口

**myproj/asgi.py**: ASGI(异步网关服务器) 接入实现

**myproj/wsgi.py**: WSGI(Web 网关服务器) 接入实现

**./manage.py**: 命令行工具入口, 之后文档里使用的本地命令, 都基于这个入口文件, 而不是使用 `django-admin`

----

也许你会疑惑, 业务逻辑代码应该写在哪里? 别急, 让我们再运行一个命令, 这里就直接引用 [官方文档] 的例子:

```shell
python manage.py startapp polls
```

这句命令的作用是在 **当前目录下创建一个 Django 的 app(Application)**: Django 的 app 被定义为**可复用的 Python package**, 真实的 Django 项目即由 project + 数个 app 组成, 每个 app 各自封装业务逻辑, 再 **安装** 到项目中.

> app 并不限定一定要在项目目录, 它可以是任意能够通过 `import` 导入的地方, 这意味着你可以使用 `pip` 安装别的开发者写的 app.
>
> 当然, 业务逻辑内聚的 app, 建议放在项目目录下

> app 既然是 Python package, 也需要注意包名冲突的问题.

那么执行 `startapp` 命令后, 现在项目结构如下:

```dir
- mysite
	- .venv/
	- myproj/
		- asgi.py
		- settings.py
		- urls.py
		- wsgi.py
	- polls/               # 新增的 polls package
	    - __init__.py
	    - admin.py
	    - apps.py
	    - migrations/
		    - __init__.py
	    - models.py
	    - tests.py
	    - urls.py          # 这个文件一开始可能并没有
	    - views.py
	- manage.py
```

虽然看起来东西有点多, 但我们一开始只需要关注下面几个模块:

**polls/views.py**: 视图模块, 处理 http 请求和返回 http 响应的代码

**polls/models.py**: 模型模块, 处理数据持久层的代码

**polls/migrations/\***: 迁移文件存放的目录

**polls/urls.py**: 路由模块, 决定 http 路由分发的代码.

> 不知道为什么, urls.py 一直都不会通过脚手架创建, 需要手动创建, 但目前我们只要知道这个模块的作用就好.

> app 里的大部分模块并不是必须的, 当你熟悉整个框架后, 可以自由的移除/添加模块. 而使用这些命名是因为 Django 设计理念是 **约定大于配置**, 即约定这样的结构, 开发者按结构编码, 就可以略去很多繁琐的配置

----

总结:

Django 的项目结构应该划分为下面几块:

- project package: 存放项目运行/配置相关模块, 根路由模块
- app package: 1 ~ n 个内聚的业务逻辑 package, package 内有对应的业务逻辑处理模块
- ./manage.py: 命令行程序入口

那么后面, 会讲解 Django 是如何让这些模块安装并 run 起来
