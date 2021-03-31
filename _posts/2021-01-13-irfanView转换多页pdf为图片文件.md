---
title: irfanView转换多页pdf为图片文件
date: 2021-01-13
category: 软件使用 
tags: [irfanView]
---

### PDF文件导出为图片，类似于打印到图片

图片打印机不好找，irfanview能实现类似的功能，用irfanview打开pdf文件然后在**选项**菜单下面有“**提取所有帧**”即可提取为每页一张图（这个可以提取视频，gif等文件），或者在**选项**菜单下面的“**多页图像**”然后选“**提取所有页**”。

命令行下也可以实现，如下：(注意：/extract后面的".\test\目录只能和要转换的文件在同一个盘符之下，写盘符似乎就出错了。")

`./i_view64.exe "x:\test\test.pdf" /extract=".\test\","jpg"`

