---
title: protocol buffers 编码规则(二)
date: 2022-09-02 09:29:33
categories: 技术
tags: gRPC
---

补充负数, 浮点数, 字符串, 嵌套结构的编码规则

## 负数

因为负数是用最高位表示符号位, 使用 varint 去编码会占用特别多的字节(10 字节), 所以 varint 不适合负数, 为了解决这个问题 pb 定义了 `sint32`, `sint64` 的类型, 先使用 ZigZag 编码, 再使用 varint 编码.

仅使用 varint 编码负数的结果:

`08 | ff fe ff ff ff ff ff ff ff 01`

具体编码过程有点复杂, 就跳过了, 总之除 tag 外会占用 10 字节, 对比上文的正数占用非常大的空间.

而 ZigZag 编码是将有符号的整数全部转换成正整数:

![](/img/20220902095738.png)

这样再使用 varint 编码就可以压缩大小.

## 非 varint 数字

使用一般编码, double 占用 8 字节(64 bit), float 占用 4 字节(32 bit), 此处没有优化.

## 字符串

字符串遵循 tag + length + data 的规则, length 使用 varint 编码

```
message HelloRequest {
  string name = 1;
}
```

编码为

`0a | 05 | 77 6f 72 6c 64`

`0a` 为 tag, `05` 为长度

## 嵌套消息体

```
message HelloRequestAgain {
	HelloRequest r = 1;
}
```

嵌套的消息体都会转成 string 编码:

`0a | 07 | 0a 05 77 6f 72 6c 64`

`0a` 为 tag, `07` 为长度
