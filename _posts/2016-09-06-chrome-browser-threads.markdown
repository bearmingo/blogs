---
layout: post
title: "Chrome Browser进程线程管理"
date: 2016-09-06
categories: chrome
---

* 线程分类

Chrome的进程分类定义在`content\public\browser\browser_thread.h`的文件中。定义了以下几种线程。
- `UI`: 浏览器进程中主线程。
- `DB`: 与数据进行交互的线程。
- `FILE`: 与文件系统进行交互的线程。
- `FILE\_USER\_BLOCKING`: 用于文件系统操作，这些操作会阻碍用户交互。这线程的无响应会影响用户。
- `PROCESS_LAUNCHER`: 用于启动与终止Chrome的进程。
- `CACHE`: 用于处理慢http缓存的操作。
- `IO`: 用于处理非阻碍的IO操作, 例如`IPC`与网络。阻碍的IO操作应该发生在其它线程(根据使用方法，合理选择`DB`, `FILE`, `FILE\_USER_BLOCKING`与`CACHE`)
