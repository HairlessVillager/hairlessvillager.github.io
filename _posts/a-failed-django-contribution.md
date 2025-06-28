---
title: 记一次失败的 Django Contribution
date: 2023-12-23 12:03
tags:
---

## 问题场景

正在学习 Django 的测试模块，看到 fixture 可以用 `manage.py dumpdata` 指令生成，遂尝试使用，发现遇到编码问题，输出的中文为 gkb 编码。在官方文档上搜索 database encoding 关键字，发现我使用的 SQLite 数据库后端的默认编码为 UTF-8，其他的地方也没发现 gbk 编码。

## 分析过程


开始调试源码。通过 manage.py 获取命令行解析模块 `django.core.management`，进一步查看源码定位目标子模块 `django.core.management.commands.dumpdata`，在 line 265 发现一处对 `serializers.serialize` 的调用，猜测这里是程序的出口，定位到 `django.core.management.serializers`。

在 `django.core.management.serializers` 发现若干子模块，为不同格式的序列化模块。使用多个 dumpdata 的 `--format` 参数，发现问题均能复现，且均为 gbk 编码中文。进入 `json.py` 子模块，通过`print(__class__.__name__)`打断点定位到入口函数 `start_serialization()`，发现对 `self.stream.write()` 的调用。尝试增加一行 `self.stream.write("ABCDEFG甲乙丙丁GFEDCBA")`，在输出文件中成功发现这个字符串，且为 gbk 编码。遂开始分析 `self.stream`。

`dumpdata.py` line 271 发现传入 `stream` 参数，分析为输出流优先为文件输出流 `stream = open_method(file_path, "wt", **kwargs)`，若无文件输出流则换为标准输出 `self.stdout`。我的场景是文件输出。定位 `open_method()` 函数，利用断点辅助调试，定位到 line 253，发现为 Python 内置函数 `open()`。

开始分析 `open_method()` 的 `kwargs` 参数。发现其在相同的位置赋值为 `{}`。

查阅 `open()` 的文档，发现 `encoding` 参数默认为 `None`，具体值为 `locale.getencoding()`，查阅文档：

> locale.getencoding()
> 获取当前的 locale encoding:
>
> 在 Android 和 VxWorks 上，将返回 `"utf-8"`。
>
> 在 Unix 上，将返回当前, return the encoding of the current `LC_CTYPE` 语言区域的编码格式。 如果 `nl_langinfo(CODESET)` 返回空字符串则将返回 `"utf-8"`: 举例来说，如果当前 `LC_CTYPE` 语言区域不受支持的时候。
>
> 在 Windows 上，返回 ANSI 代码页。

在 Windows 环境上测试，`locale.getencoding()` 返回 cp936，查阅资料获知即 gbk。

## 调用追踪

gbk 的调用追踪：

1. `manage.py dumpdata`
2. `django.core.management.commands.dumpdata`
3. `open()`
4. `locale.getencoding()`

## 查看历史工单

在申请工单之前，以关键词 dumpdata encoding 搜索历史工单，发现 [#26721](https://code.djangoproject.com/ticket/26721) 与我所遇到的问题一致，已经关闭了。他们给出的解决方案是调整 Windows 的编码，如 [如何在 Windows 上安装 Django ](https://docs.djangoproject.com/zh-hans/5.0/howto/windows/#common-pitfalls) 的“常见失误”一节所述。

## 总结

本以为能搞个 Contribution 的，哪晓得这是前人早就研究过的东西。
