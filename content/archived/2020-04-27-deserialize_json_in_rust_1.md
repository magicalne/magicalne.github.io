---
title: rust反序列化提升性能
layout: post
date: 2020-04-27
tag:
- rust
- deserialize
- serde_json
- nom

---

一个多月前入了rust的坑。当时想学学rust就是想尽可能的提升性能，但是又“不敢”写生产环境级别的C++。退而求其次，学学rust吧。

最近遇到了一个之前写java就遇到过的问题——deserialize慢。

client收到了server发送的response，把json反序列化。这是一个非常常见的场景，使用serde_json的derive可以简单干净的完成功能。
但是serde_json会使用vector，而vector是在heap上，allocation就是性能的第一块瓶颈。加上在业务逻辑处理的地方需要频繁的删除vector中的元素，dealloc也很慢。

咋办呢？

# 1. impl DeserializeSeed

第一个想法是向deserializer传递一个struct，遍历一遍input即可完成对struct的修改。对于数组字段，我可以声明一个统计意义不可越界的array，这样所有allocation都发生在stack上。DeserializeSeed可以接受参数传递，我可以实现这个trait。

但现实是，这个并不好实现。

```rust
pub trait DeserializeSeed<'de>: Sized {
    /// The type produced by using this seed.
    type Value;

    /// Equivalent to the more common `Deserialize::deserialize` method, except
    /// with some initial piece of data (the seed) passed in.
    fn deserialize<D>(self, deserializer: D) -> Result<Self::Value, D::Error>
    where
        D: Deserializer<'de>;
}
```

原因之一在于，deserialize这个method的self参数不是mutable。当然用wrapper可以解决。

第二个原因是不好解决nested object。想了两天没想明白，就放弃了。

# 2. Nom

这是第二次手写parser了，上次写java用的parboiled。这次的Nom给我打开了新世界。我一开始一直局限在自己的思想之下，一直想着怎么向parser传递变量。这是与Nom的设计相违背的。

Nom parser的返回签名很简单: **IResult<Input, Output, Error>**

parser执行成功，会返回<剩余input， 生成的输出>；如果失败，则会返回<生于input，Error>。典型的rust返回范式。而parser的输入永远只有一个，默认的实现支持&[u8]和&str。文档说也可以支持自定义的input。如果编写的parse方法需要传入其他参数，可以通过call!这个macro，实现原理就是currying，最多支持6层。

对于parser过程中需要识别的部分，则需要通过组合的方式把子parser合并起来。这也就是combinator的思想。各个combinator会byte by byte的消化input。

Nom对需要循环处理的地方也是用vec实现的，所以这里需要自己实现了。

## serde_json vs nom benchmark

是时候比一比了！

```
Deserialize OrderBookL2/serde_json
                        time:   [11.647 us 11.790 us 11.971 us]
                        change: [+7.7511% +10.118% +12.641%] (p = 0.00 < 0.05)
                        Performance has regressed.
Found 11 outliers among 100 measurements (11.00%)
  4 (4.00%) high mild
  7 (7.00%) high severe
Deserialize OrderBookL2/nom
                        time:   [7.4556 us 7.4677 us 7.4816 us]
                        change: [+3.0453% +3.3319% +3.6136%] (p = 0.00 < 0.05)
                        Performance has regressed.
Found 3 outliers among 100 measurements (3.00%)
  2 (2.00%) high mild
  1 (1.00%) high severe
```

尴尬，虽然是nom更快一点，但是没有想象中的好。还停留在微妙级别。

## 继续优化

nom本身的优化已经很极致了，[这里](https://www.youtube.com/watch?v=7VNsmlCAmHU)是nom作者做的一次分享。采用了多种优化方式，最终将http parser的性能从500MB/S提升到2GB/s。

后来我重写了parser，然后参考httparser的优化方式。httparser有多处空间换时间的操作，简单有效。

下面是我匹配字段的方法：

```rust
macro_rules! byte_map {
    ($($flag:expr,)*) => ([
        $($flag != 0,)*
    ])
}

static TOKEN_MAP: [bool; 256] = byte_map![
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 1, 0, 1, 1, 1, 1, 1, 0, 0, 1, 1, 0, 1, 1, 0,
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0,
    0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 0, 1, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
];

#[inline]
fn is_token(b: u8) -> bool {
    TOKEN_MAP[b as usize]
}

#[inline]
fn content(i: &[u8]) -> IResult<&[u8], &[u8]> {
    take_while(move |c| {
        is_token(c)
    })(i)
}
```

benchmark提升显著，两个json的速度都上来了：

```
Deserialize/nom_my_parser_order_book                                                                             
                        time:   [6.8123 us 6.8257 us 6.8408 us]
                        change: [-1.2373% -0.7937% -0.3625%] (p = 0.00 < 0.05)
                        Change within noise threshold.
Found 3 outliers among 100 measurements (3.00%)
  1 (1.00%) low mild
  1 (1.00%) high mild
  1 (1.00%) high severe
Deserialize/nom_my_parser_trade                                                                             
                        time:   [4.6656 us 4.6828 us 4.7036 us]
                        change: [-0.4447% +0.0002% +0.4730%] (p = 1.00 > 0.05)
                        No change in performance detected.
Found 8 outliers among 100 measurements (8.00%)
  4 (4.00%) high mild
  4 (4.00%) high severe
```

在这个版本中，byte array转具体类型（比如转u64）是用的lexical_core::parse，这里还有优化空间吗？

自己分别实现了to_u64、to_i32和to_f32。确实小有提升，其中to_f32的提升最明显。还尝试了古法优化unrolling，但是没有明显提升。
