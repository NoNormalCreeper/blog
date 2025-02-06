---
abbrlink: ''
categories: []
date: '2025-02-06T14:30:45.434458+08:00'
tags: []
title: 图片测试
updated: '2025-02-06T14:44:27.621+08:00'
---
> 测试博客后台管理系统与图床的兼容性。

![](https://pics.r1kka.one/file/1738824520350_Screenshot%202024-08-24%20144220.png)

![R-C.bmp](https://pics.r1kka.one/file/1738823656749_R-C.bmp)

测的时候果然发现了问题，图层 API 返回的图片链接是没有经过 URL 编码的，所以遇到特殊字符可能会导致 markdown 渲染不出来，遂去改了下源代码。
