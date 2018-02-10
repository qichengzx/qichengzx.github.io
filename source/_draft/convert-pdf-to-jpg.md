title: 转换 PDF 为 JPG
categories: golang
date: 2018-02-10 19:45:26
tags:  [go]
---

有时候需要用到解析 PDF 的功能，比如将 PDF 转换为 JPG 图片，下面通过一个示例来演示这个功能。

在这个示例中，将使用 (gographics/imagick)[https://github.com/gographics/imagick] 包来完成这个功能，目前最新版本为 3.2.0 。

通过设置分辨率，压缩级别和 alpha 通道，然后保存转换完的输出文件。同时需要注意调用 ```Terminate``` 和 ```Destroy``` 方法保证释放内存。

