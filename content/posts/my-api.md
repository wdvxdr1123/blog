---
title: "一些没什么用的API"
date: 2021-04-24T20:43:13+08:00
draft: false
---

## 前言

做这个 API 主要是为了学习 `Rust` ， 看 Rust 的书籍已经很久了，感觉基本语法都学了点，
是时候和 `rustc` 打打架了，所以一下的所有 API 都由Rust编写，也是我的第一个Rust项目。

## API

### silicon

将你的代码或文本变成一张精美的图片

{{< image src="/images/silicon_sample.png" caption="生成代码图片样例" width="774" >}}  

    POST https://api.wdvxdr.com/silicon

    Content-Type: application/json

| 字段名 |  类型  |    说明    |
|:------:|:------:|:--------:|
|  code  | string | 输入的代码 |
| format | object |   见下表   |

format格式具体内容

|   字段名    |  类型  |      说明      |
|:-----------:|:------:|:--------------:|
|  language   | string | 纯文本可填txt  |
|    theme    | string | 只能填 Dracula |
|  line_pad   |  int   |                |
| line_offset |  int   |                |
|  tab_with   |  int   |     tab宽      |

返回数据

| 字段名 |  类型  |       说明       |
|:------:|:------:|:--------------:|
|  code  |  int   | 状态码，200为成功 |
|  err   | string |     错误信息     |
|  url   | string |  生成图片的url   |

### nene

四斋蒸鹅心......

    GET https://api.wdvxdr.com/nene?words=呐

| 字段名 |  类型  |   说明   |
|:------:|:------:|:------:|
| words  | string | 一个句子 |

返回内容

| 字段名  | 类型  |      说明      |
|:-------:|:-----:|:------------:|
|  count  |  int  | 返回回复的数量 |
| replies | array |   返回的回复   |
