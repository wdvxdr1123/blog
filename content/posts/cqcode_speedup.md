---
title: "加速CQ码解析背后原理"
date: 2021-03-22T22:39:00+08:00
draft: false
---

## 前言

在开源项目 [go-cqhttp](https://github.com/Mrs4s/go-cqhttp) 中，我提交了两个 pr([#659](https://github.com/Mrs4s/go-cqhttp/pull/660),[#660](https://github.com/Mrs4s/go-cqhttp/pull/660)) 使得CQ码
(一种用字符串表示IM消息的方式)效率变为了原来的`400%`。 这篇文章简单介绍了加速解析的原理，里面的某些优化方法对 Go 语言是通用的，
希望能对读者优化 Go 程序有一定启发。

## CQ码解析的原方案

### CQ码的语法规则

CQ码的表示可以为[[1]](https://github.com/howmanybots/onebot/blob/master/v11/specs/message/string.md#cq-%E7%A0%81%E6%A0%BC%E5%BC%8F)

```antlr
CQCode
    : "[CQ:" Type (',' ParameterList)? ']'
    ;

Type
    : IDENTIFIER
    ;

ParameterList
    : (IDENTIFIER  '=' IDENTIFIER )*
    ;
```

其中`Type`是消息类型，`ParameterList`是参数列表

### 正则表达式解析

在最初解析方案是采用正则表达式，使用标准库中的`regex`包进行解析

```go
var matchReg = regexp.MustCompile(`\[CQ:\w+?.*?]`)
var typeReg = regexp.MustCompile(`\[CQ:(\w+)`)
var paramReg = regexp.MustCompile(`,([\w\-.]+?)=([^,\]]+)`)
```

### 自动机解析

使用正则表达式可以获得比较高的通用性，可读性，可维护性，但是为了压榨性能，手写自动机就是优化解析的重要途经。
Mrs4s大佬给出了一份自动机解析的实现[[2]](https://github.com/Mrs4s/go-cqhttp/blob/93d0d87fbb58adc3183622b096559f607e12ab23/coolq/cqcode.go#L320)

## 优化加速

在上一版自动机中，效率比正则表达式提高了 `30%` ，但这对于性能压榨还不够彻底，于是我便开始了优化之路。

优化后的初稿如下

```go
func ParseCQCode(s string) Message {
    var d = map[string]string{} // parameters
    var key, Type = "", ""
    l := len(s)
    i, j := 0, 0
S1: // Plain Text
    for ; i < l; i++ {
        if s[i] == '[' && s[i:i+4] == "[CQ:" {
            if i > j {
                saveText(s[j:i])
            }
            i += 4
            j = i
            goto S2
        }
    }
    goto End
S2: // CQCode Type
    d = map[string]string{}
    for ; i < l; i++ {
        switch s[i] {
        case ',': // CQ Code with params
            Type = s[j:i]
            i++
            j = i
            goto S3
        case ']': // CQ Code without params
            Type = s[j:i]
            i++
            j = i
            saveCQCode()
            goto S1
        }
    }
    goto End
S3: // CQCode param key
    for ; i < l; i++ {
        if s[i] == '=' {
            key = s[j:i]
            i++
            j = i
            goto S4
        }
    }
    goto End
S4: // CQCode param value
    for ; i < l; i++ {
        switch s[i] {
        case ',': // more param
            d[key] = CQCodeUnescapeValue(s[j:i])
            i++
            j = i
            goto S3
        case ']':
            d[key] = CQCodeUnescapeValue(s[j:i])
            i++
            j = i
            saveCQCode()
            goto S1
        }
    }
    goto End
End:
    if i > j {
        saveText(s[j:i])
    }
}
```

### 去除不必要的转换

在上一版自动机中，将整个字符串转换成了 `[]rune` ，观察到 CQ 码的结构中只含有 ASCII 字符，根据 `UTF-8` 的性质，
所有的非 ASCII 字符的个个字节都大于128，所以对于解析CQ码，我们并不需要将字符串转换成 `[]rune` 。

### 设计一个更全面的状态机

在上一版状态机中,状态数目较少，只能解析出整个CQ码的字符串，将Type和参数取出来，需要使用Split分割，
于是我设计了一个更加全面的状态机。

{{< mermaid >}}
stateDiagram-v2
    [*] --> PlainText
    note left of PlainText: 纯文本
    PlainText --> CQCode: 遇到 [CQ：
    state CQCode {
        [*] --> Type
        Type  --> ParameterList : 遇到 ,
        ParameterList -->[*]
        Type  --> [*] : ] (无参数)
        state ParameterList {
            [*] --> key
            key --> value : 遇到 =
            value --> key : 遇到 ,
            value --> [*] : 遇到 ]
        }
    }
    CQCode --> PlainText
{{< /mermaid >}}

相比之前的状态机，可以更加精细的解析CQ码的内容，避免了重复操作，通过一次遍历就完成了所有
CQ码的解析。

### 避免拷贝，使用切片进行内存复用

对于CQ码，我们只需要从字符串中提取一部分，而且后续的操作中，我们不会对字符串进行修改，所以我是用了切片操作来
截取字符串，尽可能的减少内存分配

### 消除边界检查

Go语言是一门内存安全的语言，程序运行过程中会进行边界检查，这对程序的稳定避免发生更严重的错误很重要，但是有些情况下，
我们可以保证一定不会发生数组(切片)越界，这时边界检查就变得多余了。

在 Go 编译器中，实现了对某些情况的边界检查消除，例如标准库`encoding/binary`中

```go
func (littleEndian) PutUint32(b []byte, v uint32) {
    _ = b[3] // early bounds check to guarantee safety of writes below
    b[0] = byte(v)
    b[1] = byte(v >> 8)
    b[2] = byte(v >> 16)
    b[3] = byte(v >> 24)
}
```

在第二行中，提前检查了边界，Go编译器就会对后续的4行操作不进行边界检查。
Go对边界检查消除优化还不够全面，往往需要我们手动提示编译器进行边界检查消除。

例如下面代码中，对`is`的范围进行暗示

```go
func foo(is []int, bs []byte) {
    if len(is) >= 256 {
        is = is[:256] // 给编译器一个暗示
        for _, n := range bs {
            _ = is[n] // 边界检查消除了！
        }
    }
}
```

虽然我重写了一遍状态机，
为了探究性能并无很大提升的原因，我使用了

```shell
go build -gcflags="-d=ssa/check_bce/debug=1"
```

查找了代码中所有的边界检查，结果发现代码中几乎所有切片操作都进行了边界检查，
同时，在我的自动机实现代码中，保证了索引一定不会越界 (`i < l`)，于是我尝试给编译器一定
提示，尝试消除边界检查。

不幸的是，我尝试了已知的所有 Hint 手段，编译器始终不能进行边界检查消除，这时候我选择
另一条道路—— `unsafe`。

通过 `unsafe` 操作，实现边界消除。

{{< admonition note "unsafe" false >}}

unsafe (×)

I say it is safe (√)

{{< /admonition >}}

使用了unsafe操作后的代码如下

```go
// add 指针运算
add := func(ptr unsafe.Pointer, offset uintptr) unsafe.Pointer {
    return unsafe.Pointer(uintptr(ptr) + offset)
}

S1: // Plain Text
    for ; i < l; i++ {
        if *(*byte)(add(ptr, uintptr(i))) == '[' && i+4 < l &&
            *(*uint32)(add(ptr, uintptr(i))) == magicCQ { // Magic :uint32([]byte("[CQ:"))
            if i > j {
                saveText(s[j:i])
            }
            i += 4
            j = i
            goto S2
        }
    }
    goto End
S2: // CQCode Type
    d = map[string]string{}
    for ; i < l; i++ {
        switch *(*byte)(add(ptr, uintptr(i))) {
        case ',': // CQ Code with params
            Type = s[j:i]
            i++
            j = i
            goto S3
        case ']': // CQ Code without params
            Type = s[j:i]
            i++
            j = i
            saveCQCode()
            goto S1
        }
    }
    goto End
S3: // CQCode param key
    for ; i < l; i++ {
        if *(*byte)(add(ptr, uintptr(i))) == '=' {
            key = s[j:i]
            i++
            j = i
            goto S4
        }
    }
    goto End
S4: // CQCode param value
    for ; i < l; i++ {
        switch *(*byte)(add(ptr, uintptr(i))) {
        case ',': // more param
            d[key] = CQCodeUnescapeValue(s[j:i])
            i++
            j = i
            goto S3
        case ']':
            d[key] = CQCodeUnescapeValue(s[j:i])
            i++
            j = i
            saveCQCode()
            goto S1
        }
    }
    goto End
End:
    if i > j {
        saveText(s[j:i])
    }
    return
```

同时，我将判断是否是CQ码头 `"[CQ:"` 改成了用 `uint32` 进行比较(与大小端有关),进一步优化了性能. 
最终的结果是十分的 Amazing 啊，比重写前的状态机快了几倍。

### 减少map的创建

目前的实现已经十分快了，正当我觉得没有任何可以优化的地方时，我突然发现了用于保存CQ码参数的map可以进行内存复用。

Go 编译器对map的清空进行了优化，下面这样的代码，Go 编译器会将它优化成一个runtime内部函数，对整个map一次清空。

```go
var a map[type1]type2

for k := range a {
    delete(a, k)
}
```

虽然并不能释放已使用的内存，但是我们可以复用内存，减少map的扩容，减少`alloc`，大大提高运行速度。

{{< admonition tip "内存复用" true >}}

在并发的场景下，可以使用 `sync` 包中的 `sync.Pool` 实现线程安全的内存复用。

{{< /admonition >}}

于是我做了一个小改动

```go
S2: // CQCode Type
    // d = map[string]string{} // old 
    for k := range a {
        delete(a, k)
    }
    for ; i < l; i++ {
        // ignore
    }
    goto End
```

经过这次改动后，CQ码的解析速度又提高了`33%`。

## 总结

最后总结下来，优化Go程序的性能可以从下面几个角度入手

1. 善于内存复用，使用切片或 `sync.Pool` 减少内存分配。
2. 尽量避免不必要的类型转换带来的开销，例如 `[]byte` 与 `string` 之间转换，
`[]rune` 与 `string` 之间的转换。
3. 合理暗示编译器，避免不必要的越界检查。
