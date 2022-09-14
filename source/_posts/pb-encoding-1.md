---
title: protocol buffers 编码规则(一)
date: 2022-09-01 14:29:33
categories: 技术
tags: gRPC
---

gRPC 使用的 protocol buffers 编码在序列化方面比传统的 json, xml 更小巧, 更快速, 那么到底是什么做的呢?

```
message HelloRequest {
  string name = 1;
}
```

使用 gRPC demo 中最简单的结构, 我们使 `name="world"` 那么通过 pb 编码后得到数据为:

`0a 05 77 6f 72 6c 64`

## 消息体结构

pb 中消息定义为 0 ~ n 个键值对的集合:

```
message   := (tag value)*     You can think of this as “key value”
```

tag 是由字段序以及被称为 wire type 的值组成, 例如上文的 `0a` 即是 tag, 它的最低 3 位表示 wire type, 高位表示字段序号:

```
// 0a
0 0 0 0 1 | 0 1 0  => 序号 1, string
```

因为是 string, 那么需要跟上长度 `05`, 剩下的 `77 6f 72 6c 64` 即是 `world` 这个词的字节码.

![](/img/20220901093811.png)

这是一个稍微复杂结构的例子:

```
message Person {
    required string user_name       = 1;  // "Martin"
    optional int64  favorite_number = 2;  // 1337
    repeated string interests       = 3;  // "daydreaming", "hacking"
}
```

![](/img/20220901094152.png)


## varint

从上面的例子中, 可以发现 pb 中对于 int, bool 等类型, 存储上使用的是 `varint` 类型, 这是一种紧凑的表示数字的方法.

varint 的除了最后一个字节外, 其余每个字节最高位, 都用于表示这个数字还有更多的字节 (most significant bit), 所以每个字节用于表示数字只有 7 位, 且最终每个字节需要按逆序组合, 即数字的高位在后, 低位在前.

我们拿上图示例的 1337 (`b9 0a`) 作为例子:

```
// b9
1 | 0 1 1 1 0 0 1  => 1 为 msb, 表示下一个字节也属于该数字, 取剩下 7 位
// 0a
0 | 0 0 0 1 0 1 0  => 0 为 msb, 表示已到结束, 取剩下 7 位


// 因为高位在后, 低位在前, 所以需要逆序排, 合并两个 7 位并得到数字
0 0 0 1 0 1 0 | 0 1 1 1 0 0 1

// (1 << 10) + (1 << 8) + (1 << 5) + (1 << 4) + (1 << 3) + 1
// 1337
```

使用 varint 的优势: 数字越小, 占用的字节数越少

> 字段序号建议在 15 内, 因为去掉 3 位 wire type, 一个字节只剩下 4 位 (除去第一位 MSB), 只能表示 1 ~ 15, 15 以内只占用 1 个字节, 16 ~ 2047 占用 2 个字节
