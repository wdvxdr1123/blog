---
title: "一些没什么用的API"
date: 2021-04-24T23:35:13+08:00
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

### hitokoto

一言， 数据源自 `hitokoto.cn`

    GET https://api.wdvxdr.com/hitokoto?category=anime

当 category 不存在时，会随机从数据多的库中获取。

|    类型    |   说明   |
|:----------:|:------:|
|   anime    |   动画   |
|   comic    |   漫画   |
|    game    |   游戏   |
| literature |   文学   |
|  original  |   原创   |
|  internet  | 来自网络 |
|   other    |   其他   |
|   video    |   影视   |
|    poem    |   诗词   |
|    ncm     |  网易云  |
| philosophy |   哲学   |
|   funny    |  抖机灵  |

返回数据和 `hitokoto.cn` 一致

| 返回参数名称 |                 描述                  |
|:------------:|:-----------------------------------:|
|      id      |               一言标识                |
|   hitokoto   | 一言正文。编码方式 unicode。使用 utf-8。 |
|     type     |      类型。请参考第三节参数的表格      |
|     from     |              一言的出处               |
|   from_who   |              一言的作者               |
|   creator    |                添加者                 |
| creator_uid  |            添加者用户标识             |
|   reviewer   |              审核员标识               |
|     uuid     |             一言唯一标识；             |
| commit_from  |               提交方式                |
|  created_at  |               添加时间                |
|    length    |               句子长度                |
