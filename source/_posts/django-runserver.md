---
title: 首次运行 Django
date: 2022-06-20 16:09:32
tags: Django
categories: 技术
---

上文, 我们认识了一下 Django 项目下面的模块组成, 但目前这些模块只是一个个独立的 package 还没有组装, 下面我们来把上文示例的 `polls` package "**安装**" 到项目中, 并把项目运行起来.

首先, 我们需要让项目知道要 "**安装**" 哪些 app, 打开 `myproj/settings.py`, 找到 `INSTALLED_APPS` 选项:

```python
# Application definition

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```

先忽略 django 开头的行, 在这个 list 的最后, 添加一行:

```python
# Application definition

INSTALLED_APPS = [
    'django.contrib.admin',
	...


	'polls',  # 新增一行
]
```

如此就完成了 "**安装**"

但我们的 `polls` 里没有任何的业务处理逻辑, 所以看不到效果, 我们给 `polls` 添加一些简单的业务逻辑:

编辑 `polls/views.py`

```python
from django.http import HttpResponse

def index(request):
    return HttpResponse("Hello, world. You're at the polls index.")
```

> 上文有说过 views 是视图处理模块, 包含接收 http 请求和返回 http 响应的代码, 所以视图函数的约定形式为: 第一个入参接收 request, 返回值必然是一个 response

我们还需要把 http 的 url 和视图函数 **连接** 起来, 让程序知道, 当我们访问某个 url 地址时, 让它执行上面的 `index` 函数, 而起到 **连接** 作用的模块就是 `urls.py`:

编辑/创建 `pools/urls.py`

```python
from django.urls import path

from . import views   # 提一嘴这里是相对导入同包的 views 模块

urlpatterns = [
    path('', views.index, name='index'),
]
```

> `urlpatterns` 是一个 **约定** 的 list 变量, 它告诉 Django 从 urls 中导入这个值, 就可以加载到路由分发映射.

再来, 编辑 **根路由** 把 `polls` 里的路由模块 `include` 进来:

编辑 `myproj/urls.py`, 并在 `myproj/urls.py` 里的 `urlpatterns` 变量中添加一行:

```python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),  # 这个先忽略

    # 新增一行, include polls.urls, 注意 include 函数可能未导入,
    # 请注意将第二行修改为 from django.urls import include, path
    path('polls/', include('polls.urls')),
]
```

这样就完成了路由的挂载.

如果有些难理解, 可以把路由想像成一棵树结构, 顶层为项目的根路由, 下面挂载(include) app 的子路由或者直接加载视图函数. 当然 app 的子路由也可以挂载(include) 别的子路由和加载别的视图函数:

![](/img/20220620161533.png)


至此, 我们可以跑一下这个项目:

```shell
python manage.py runserver
```

> `runserver` 命令作用是启动一个 **web 开发服务器**, 开发服务器用于本地日常开发调试代码, 而不需要把服务部署到 web server 上运行.

如果一切正常, 你可以在浏览器上打开 [http://127.0.0.1:8000/](http://127.0.0.1:8000/) 看到如下界面:

![](/img/20220620161017.png)

这是个 `404` 页面, 并会显示路由表, 告诉你项目加载了哪些路由, 现在让我们修改浏览器地址为: [http://127.0.0.1:8000/pools/](http://127.0.0.1:8000/pools/)

应当可以看到:

![](/img/20220620162507.png)

为什么请求路径是 `/polls/` 而不是 `/` ?

让我们回过头看看 `myproj/urls.py` 和 `polls/urls.py` 就会发现每一个 `include` 的路由节点, 都会添加其前缀进去匹配:

```python
# myproj/urls.py
urlpatterns = [
	...
    path('polls/', include('polls.urls')),  # 注意 path('polls/', ...), 表示下面的子路由带有 polls/ 前缀
]

# polls/urls.py
urlpatterns = [
    path('', views.index, name='index'),    # 这里是 path('', ...), 等同于 '/'
]
```

| 127.0.0.1:8000 | /polls/ | / |
| -------------- | ------- | - |
|  | myproj.urls |  polls.urls |

> 这样做的好处是, app 只需要关注 **自身** 的路由表, 不用担心全局路由声明冲突, 由安装的 project 控制每个 app 路由表的转发入口
