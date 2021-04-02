---
title: "在Go1.16导出C文件中的函数"
date: 2021-04-02T15:23:02+08:00
draft: false
---

## 缘起

在群里聊天时，听到了群友说在更新了Go 1.16后，使用`c-shared`编译无法导出C文件中
定义的导出函数。处于兴趣，我便开始探索无法导出的缘由。

## 排查过程

```go
package main

import "C"

//export AddGo
func AddGo(a C.int,b C.int) C.int {
    return C.int(int(a) + int(b))
}

func main() {}
```

```c
extern int __stdcall AddC(int a,int b) {
    return a + b;
}
```

将这两文件放一个文件夹，使用 `go build -buildmode=c-shared -o a.dll` 进行编译，
然后使用`dumpbin`工具查看导出函数

Go 1.16

```text
    ordinal hint RVA      name

          1    0 0006F640 AddGo
          2    1 00138410 _cgo_dummy_export
```

发现确实只导出了Go文件中使用 `//export` 导出的函数。

## Go 1.16 在CGO中的改动

在go仓库中搜寻有无类似 issue 时，go1.16 修复的一个 bug 引起了我的注意 [#43591](https://github.com/golang/go/issues/43591)

其大致内容是，使用 `c-shared` 编译时，会导出很多很多函数，不仅仅是标注为 `//export` 的函数，很多标准库中的函数，包括运行时内部的
函数也被导出。

我查看了修复这个 bug 的 commit 记录[link](https://github.com/golang/go/commit/6f7b553c82b69b47becbe36d9115971d30fdab48).

{{< image src="/images/go1.16-cgo1.png" caption="修复的 commit 记录" >}}  

从这条 commit 中可以看出， cgo 工具在go 1.16中为所有Go文件中定义的导出函数加上了 `__declspec(dllexport)`，
这样没有使用 `//export` 的函数就不会被导出，但是我们的 C 文件中定义的函数也会被忽略。

由此，我们很容易得出解决方法——将c函数加上`__declspec(dllexport)`。

```c
extern int __stdcall __declspec(dllexport) AddC(int a,int b) {
    return a + b;
}
```

重新编译后，使用 `dumpbin` 查看导出函数

```text
    ordinal hint RVA      name

          1    0 0006F6A0 AddC
          2    1 0006F640 AddGo
          3    2 00138410 _cgo_dummy_export
```

不出所料，`AddC` 函数成功的被导出。问题成功解决!!!
