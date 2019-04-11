---
layout:    post
title:     Markdown转PDF
subtitle:   "使用VSCode将md转为PDF"
date:       2019-04-10
author:     Jiangs
header-img: img/view.jpg
catalog:    true
tags:
     - VSCode
---

## 背景

经常使用VSCode写markdown文件，并不想使用其他软件甚至收费软件进行转换，并且也没有要求样式风格，故使用VSCode中的插件进行转换

## 过程

1. 安装Markdown PDF 插件

2. 调出设置页面(ctrl+,)

3. 在extension中找到markdown-pdf扩展，并点击*edit in settings.json*

4. 添加如下内容

```
{
    "window.zoomLevel": 0,
    "markdown-pdf.executablePath": "C:\\Users\\Jiangs\\AppData\\Local\\Google\\Chrome\\Application\\chrome.exe",
    "markdown-pdf.outputDirectory": "D:\\文档"
}
```

其中executablePath为chrome浏览器地址，outputDirectory为输出文件的目录

5. 在任意一个markdown文件中（以md为后缀），右键 *Markdown PDF：Export(pdf)*即可

## 问题

1. 在安装扩展之后VS会自动安装chrome(多半是不会成功的)，如果你本地已有chrome浏览器，可以取消按上述操作，如果自动安装成功，可以不配置executable。

2. 如果右键没有*Markdown PDF：Export(pdf)*，可以使用F1调出命令行输入*export*，此时应该就可以看到了。