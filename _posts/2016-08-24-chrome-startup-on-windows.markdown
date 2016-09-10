---
layout: post
title: Chrome在Windows的启动流程
date: 2016-08-24
categories: chrome
---

# `src/chrome/app/chrome_exe_main_win.cc` : `wWinMain`

1. `base::AtExitManager`初始化
2. 设置

# `src/chrome/app/main_dll_loader_win` : `MainDllLoader::Launch`

加载Chrome.dll并进入到下面的函数ChromeMain函数中

# `src/chrome/app/chrome_main.cc` : `ChromeMain`


# `src/content/app/content_main.cc` : `ContentMain`


# `src/content/app/content_main_runner.cc` :  `ContentMainRunner::Run'

根据命令行参数的中进程类型，进入不同的入口

- ""                      => `BrowserMain`
- "ppapi-plugin-launcher" => `PpapiPluginMain`
- "ppapi-broker"          => `PpapiBrokerMain`
- "utility"               => `UtilityMain`
- "renderer"              => `RendererMain`
- "gpu-process"           => `GpuMain`
- "download"              => `DownloadMain`
