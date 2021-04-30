---
title: "Calculate Spacing Between Slices"
date: 2021-04-30T10:26:13+08:00
tags: [ "DICOM" ]
keywords: [ "DICOM" ]
categories: [ "DICOM" ]
draft: true
isCJKLanguage: true
---

计算切片间距的方法：

对于CT扫描出的断层图像，没有存储**Spacing Between Slices**信息，但可以利用位置信息计算得到。

需要先读取相邻两层切片的位置信息，假设为`pos1`和`pos2`，然后计算两个位置的距离即为切片间距。

``` csharp
// double pos1[3], pos2[3];
double spacing = sqrt(
    (pos1[0] - pos2[0]) * (pos1[0] - pos2[0]) +
    (pos1[1] - pos2[1]) * (pos1[1] - pos2[1]) +
    (pos1[2] - pos2[2]) * (pos1[2] - pos2[2]));
```
