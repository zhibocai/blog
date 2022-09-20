---
title: 一种 http 1.1 转 gRPC 调用的网关
date: 2022-09-19 23:46:18
tags: gRPC
categories: 技术
---

因为项目中需要对接多个 gRPC 微服务, 但 gRPC 写得太烦了, 不像 http 提供方法(method), 路径(path), 数据(data) 即可:

```python
import grpc
# 需要导入对应的模块
import helloworld_pb2
import helloworld_pb2_grpc

with grpc.insecure_channel('localhost:50051') as channel:
	# 创建 stub, 实际上就是 client
	stub = helloworld_pb2_grpc.GreeterStub(channel)
	# 创建输入对象
	request = helloworld_pb2.HelloRequest(name='you')
	# 调用并返回输出
	response = stub.SayHello(request)
	print("Greeter client received: " + response.message)
```

所以整了一个网关组件, 使用一种通用化的 http 调用方式自动的转成 gRPC 调用.

## 开源方案

在这之前, 调研过两个开源组件:

- envoy: 从它的文档上看 [gRPC-JSON 转码器](https://cloudnative.to/envoy/configuration/http/http_filters/grpc_json_transcoder_filter.html), 每个 gRPC 调用都需要逐一配置, 成本太高, 不合适
- grpc-gateway(go):  这个东西看起来满足, 但是不够通用化.
	- 需要 go 的技术栈, 需要 proto 指定 **go_package**, 如果是别的语言的 proto 就必须自己添加
	- 会额外生成一些代码
	- 附带的转 restful 风格没多大用处, 不能成为加分项


> 我认为这种网关最需要的就是简单化, 把 gRPC 的 proto 丢到指定的目录里, 什么都不用做就能用, 才做到开箱


## 自研组件

我认为对于 proto 不需要额外添加 option 的 python 版本, 最适合做这种通用网关, 而且 python 的动态性, 比较容易做动态加载和动态更新.

主要问题是如何把 http 调用映射到 gRPC 调用上:

|    | http | gRPC |
|----|------|------|
| method | http 使用 GET / POST / ... 作为方法 | gRPC 是 proto 定义的方法名 |
| path | http 使用 /path/to 作为请求路径 | gRPC 是 package.Service |
| data | http 可使用 json 传输数据 | gRPC 是 protocol buffers |

这里设计了一套转换规则: **统一使用 POST 请求, 路径由 proto 的 Service  + Method 组成, 数据使用 json, 到达网关后, 由网关根据 proto 对应的 input type + json data 创建出 gRPC 需要的入参**

实现:

1. 编译 .proto 文件, 会得到 xxx_pb2.py 以及 xxx_pb2_grpc.py 文件
2. 导入模块, 通过代码 pb2 模块中 DESCRIPTOR , 分析 proto 包含的 Service 和 Method
```python
import importlib

protos = {}

services = {}

packages = {}

package_namespaces = {}

def setup(proto_dir: str = None):
    proto_dir = proto_dir or settings.proto_dir


    # 安装 sys.path, 下面需要预导入 pb 模块
    sys.path.append(proto_dir)

    # 将每个服务的 proto 分别放置 proto_dir 的子目录下, 都是以一个 package 方式导入
    for package in os.listdir(proto_dir):
        if package.startswith('_'):
            continue

        package_dir = os.path.abspath(os.path.join(proto_dir, package))
        if os.path.isdir(package_dir):
            package_services = {}

            for filename in os.listdir(package_dir):
                if filename.endswith('.proto'):
                    # 导入 xxx_pb2 和 xxx_pb2_grpc 模块, 并保存引用
                    proto_module, service_module = import_proto_module(package, os.path.splitext(filename)[0])

                    # 利用 DESCRIPTOR 可以查找 service 的全路径名, 方法, 以及入参出参的类型
                    proto_descriptor = proto_module.DESCRIPTOR
                    protos[proto_descriptor.name] = (proto_module, service_module)

                    for service_descriptor in proto_descriptor.services_by_name.values():

                        package_services[service_descriptor.full_name] = service_descriptor
                        package_namespaces[service_descriptor.full_name] = package


            # cache packages service
            packages[package] = package_services
            # cache global service
            services.update(package_services)
```

3. 客户端

```python
from google.protobuf.json_format import MessageToDict


class Resolver(object):

    def __init__(self, service_descriptor, method_descriptor):
        self.service_descriptor = service_descriptor
        self.method_descriptor = method_descriptor

    @property
    def service_name(self):
        return self.service_descriptor.name

    @property
    def service_file_name(self):
        return self.service_descriptor.file.name

    @property
    def service_method_name(self):
        return self.method_descriptor.name

    @property
    def service_stub_name(self):
        return f'{self.service_descriptor.name}Stub'

    @property
    def service_stub(self):
        """
        获得 Stub 类型, 通过 descriptor 在对应的 service 模块中反查
        """
        # protos 记录了 proro 文件对应的 (proto 模块, service 模块)
        # stub 从 service 模块获得
        return getattr(protos[self.service_file_name][1], self.service_stub_name)

    @property
    def input_type(self):
        """
        获得输入类型, 通过 descriptor 在 db._classes 反查
        """
        return db._classes[self.method_descriptor.input_type]

    def get_service_method(self, channel):
        """
        获得对应的服务方法 Callable
        """
        stub = self.service_stub(channel)
        return getattr(stub, self.service_method_name)

    def get_service_input(self, data):
        """
        获得对应的服务入参实例
        """
        try:
            return self.input_type(**data)
        except (TypeError, ValueError) as exc:
            # 传入错误的类型, 缺少/未知字段, 会抛出 TypeError, ValueError
            raise SchemaError(str(exc))



class Client(object):

    def __init__(self, endpoint):
        self.endpoint = endpoint
        self._channel = None

    def channel(self):
        """
        return a grpc aio channel
        """
        if self._channel is None:
            self._channel = grpc.insecure_channel(self.endpoint, options=[
                                ('grpc.max_send_message_length', settings.grpc_max_message_length),
                                ('grpc.max_receive_message_length', settings.grpc_max_message_length),
                            ])
        return self._channel

	def invoke(self, ervice: str, method: str, data: dict) -> dict:
	    """
	    调用 gRPC
	    """
	    channel = self.channel()       
	    resolver = resolve(service, method)
	    input = resolver.get_service_input(data)
	    method = resolver.get_service_method(channel)
	    message = method(input, timeout=timeout)
	    return MessageToDict(message)
```

4. 调用, 只需要传入 proto 完整的 Service + Method 就可以了
```python
client = Client(endpoint)    
# call rpc method
return await client.invoke(service, method, data)
```

> 总的来说效果不错, 基本实现了只要丢 proto 就可以跑(需要跑一下编译生成 pb 模块), 之后利用 asyncio 优化请求性能, 额外增加了重试 / 在线文档.
