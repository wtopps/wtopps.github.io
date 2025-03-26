---
layout: post
title: Protobuf Varint序列化原理
subtitle: Protobuf Varint序列化原理
excerpt_image: https://github.com/wtopps/wtopps.github.io/blob/master/images/Protobuf.jpeg?raw=true
categories: Others
tags: [Serialization]
top: 1
---

![banner](https://github.com/wtopps/wtopps.github.io/blob/master/images/Protobuf.jpeg?raw=true)

## 前言

Protocol Buffers（简称 protobuf）使用 varint（可变长整数）格式来序列化一些整数类型（如 int32, int64, uint32, uint64 等）。varint 的基本原理是将整数按照字节进行分段编码，以尽可能节省存储空间。这种编码方式使得对于较小的整数值，占用的字节数非常少，而对于较大的整数，则会根据需要动态增加字节。

## Varint 序列化的工作原理

**字节按位拆分**：Protobuf 将整数值分解为一组 7 位的块（字节），每个字节保存一个 7 位的数据

**高位标记**：每个字节的最高位（第 8 位）是一个标志位，用于指示该字节是否是最后一个字节。具体地：

- 如果字节的最高位为 1，表示该字节之后还有更多字节（即这个字节不是最后一个字节）
- 如果字节的最高位为 0，表示这是该整数的最后一个字节

**分块与压缩：**通过将整数分成多个 7 位的块，Protobuf 能够将小整数用一个字节表示，而对较大的整数则按需增加字节数

## Why Varint？

当我们日常处理的数字大多数都是比较小的数值时，比如1、2、3这样的整数，使用标准的32位二进制格式来存储这些值实际上是非常浪费资源的做法。

这是因为每个这样的小数都会占用整整32位的空间，而其中大部分位都被设置为了0，这导致了存储空间上的极大浪费。为了解决这个问题，人们提出了变长整数（varint）编码方案。

Varint是一种能够根据实际需要动态调整所占位数的数据表示方法。对于较小的整数来说，它可以在保证信息完整性的前提下大大减少所需的空间。具体而言，当一个整数足够小时，它可以被编码成较少的字节；随着数值增大，才逐渐增加额外的字节数以容纳更大的数值范围。这样做的好处是显著提高了小数值情况下的存储效率。

然而，虽然varint在处理小数字方面表现优异，但如果应用场景中频繁出现非常大的整数，则采用这种编码方式反而可能会消耗比固定长度编码更多的空间。这是因为在varint编码机制下，较大的数值需要通过多个字节来表达，并且每个字节还必须包含一些额外的信息（如标志位），用来指示当前字节是否属于该数值的一部分以及是否还有后续字节。因此，在某些特定条件下，与直接使用固定的32位或64位表示法相比，varint可能不是最优的选择。

## 具体编码流程

1. 假设要编码一个 32 位的整数 ( n )，下面是具体的编码步骤：

2. 将整数 ( n ) 拆分为多个 7 位块，按顺序存储在字节流中。

   对于每个字节：

   - 取整数的低 7 位（n & 0x7F），这部分将存储在字节的低 7 位中。

   - 如果整数还剩余部分，右移 7 位（n >>= 7），然后继续编码下一字节。

   - 每个字节的高位（第 8 位）标记是否是最后一个字节。如果是最后一个字节，设置高位为 0，否则设置高位为 1。

### 举个例子
假设我们要编码整数 300：

1. **300 转换为二进制：**

     300 的二进制表示是 100101100（9 位二进制）。

2. **将整数分解成 7 位块：**

     第一个 7 位块是 1001011（即 75，也就是 300 除以 128 的商，右移 7 位）。

     第二个 7 位块是 00（即 44，剩下的 44 位）。

3. **每个字节的高位：**

     对于第一个字节，1001011 后面还有字节，因此最高位是 1。

     对于第二个字节，00 是最后一个字节，最高位是 0。

4. **最终编码：**

     第一个字节（75）编码为 0x4B，高位为 1。

     第二个字节（44）编码为 0x2C，高位为 0。

     因此，300 被编码为两个字节：0x4B 0x2C。

## 解码过程

解码时，Protobuf 会根据高位标记是否为 1 来判断是否继续读取下一个字节。直到遇到高位为 0 的字节，表示所有数据已经读取完毕。

1. 从字节流中读取，每次读取一个字节。
2. 去掉高位，将每个字节的低 7 位合并到结果中。
3. 根据高位判断是否继续读取，如果高位是 1，则继续读取下一个字节；如果高位是 0，表示所有字节都已经读取完毕。

![serialize demo](https://github.com/wtopps/wtopps.github.io/blob/master/images/protobuf%20varint%20process.png?raw=true)



再看一个Google官方文档给出的示例：

> And here is 150, encoded as `9601` – this is a bit more complicated:

```proto
10010110 00000001
^ msb    ^ msb
```

> How do you figure out that this is 150? First you drop the MSB from each byte, as this is just there to tell us whether we’ve reached the end of the number (as you can see, it’s set in the first byte as there is more than one byte in the varint). These 7-bit payloads are in little-endian order. Convert to big-endian order, concatenate, and interpret as an unsigned 64-bit integer:

```proto
10010110 00000001        // Original inputs.
 0010110  0000001        // Drop continuation bits.
 0000001  0010110        // Convert to big-endian.
   00000010010110        // Concatenate.
 128 + 16 + 4 + 2 = 150  // Interpret as an unsigned 64-bit integer.
```



## 优势

节省空间：由于小数字只需要一个字节，因此在存储小的整数时，Protobuf 比其他固定长度编码方式（如 32 位或 64 位的整数）节省了大量空间。

灵活性：varint 可以动态调整字节数，使得编码更适合不同大小的整数值。

高效：对于大部分常见的数值，Protobuf 的 varint 编码方式相对高效，尤其是在数值较小的时候。

## 总结

Protobuf 的 varint 编码 通过将整数拆分为多个 7 位块，使用高位标记（0x80）来指示是否有更多字节，从而实现了压缩存储。

对于较小的整数，varint 格式非常节省空间，只有需要更多字节时才会增加开销。

这种编码方式非常适合序列化整数类型，尤其在处理大量小数字时能够显著减少存储空间。

参考文档：
https://protobuf.dev/programming-guides/encoding/



<script src="https://giscus.app/client.js"
        data-repo="wtopps/wtopps.github.io"
        data-repo-id="MDEwOlJlcG9zaXRvcnk2NzY3NTA3MA=="
        data-category="Comments"
        data-category-id="DIC_kwDOBAijvs4CizS6"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="preferred_color_scheme"
        data-lang="zh-CN"
        crossorigin="anonymous"
        async>
</script>