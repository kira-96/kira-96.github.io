---
title: DICOM图像像素相关Tag说明
date: 2020-06-15T10:30:00+08:00
description: DICOM图像相关
tags: [ "DICOM" , "图像" ]
keywords: [ "DICOM" ]
categories: [ "DICOM" ]
isCJKLanguage: true
enableMathJax: true
enableDisqus: true
---

## 常用图像像素相关的一些Tag

|Tag|VR|Keyword|
|:---:|:---:|:---:|
|(0028,0002)|US|Samples Per Pixel|
|(0028,0004)|CS|Photometric Interpretation|
|(0028,0006)|US|Planar Configuration|
|(0028,0010)|US|Rows|
|(0028,0011)|US|Columns|
|(0028,0100)|US|Bits Allocated|
|(0028,0101)|US|Bits Stored|
|(0028,0102)|US|High Bit|
|(0028,0103)|US|Pixel Representation|
|(7FE0,0010)|OW|Pixel Data|

## 相关Tag说明

### Sample Per Pixel

 > Samples per Pixel (0028,0002) is the number of separate planes in this image. One and three image planes are defined. Other numbers of image planes are allowed, but their meaning is not defined by this Standard.

> For monochrome (gray scale) and palette color images, the number of planes is 1.
  For RGB and other three vector color models, the value of this Attribute is 3.

Samples Per Pixel 指此图像中平面的个数。对于灰度图像，它的值为**1**，对于RGB等彩色图像，它的值为**3**。
听起来可能比较拗口，简单解释一下，对于灰度的图像，它只有一个灰度值，所以是1，而彩色的图像通常是由RGB三个通道混合而成，它的值为3。

### Photometric Interpretation

指定解析图像像素的格式。个人比较习惯叫它图像类型，可以根据这个tag判断图像是灰度还是彩色图像。

- MONOCHROME1

  灰度图，最小值显示为白色，像素值越大就越黑。

- MONOCHROME2

  灰度图，最小值显示为黑色，像素值越大就越亮。这应该是最常用的格式了。

- PALETTE COLOR

  自带调色板的图像，显示出来是彩色的，所以属于彩图。当使用它时，Samples Per Pixel的值必须为**1**。并且必须要有RGB三种颜色的调色板查找表，它的像素值(Pixel Data)用于查找表。

  > The pixel value is used as an index into each of the Red, Blue, and Green Palette Color Lookup Tables (0028,1101-1103&1201-1203).

- RGB

  彩色图像，每个像素由RGB三种颜色组成，Samples Per Pixel值必须为**3**。

- YBR_FULL

  通过色度信号来表示颜色的格式，每个像素由一个亮度**Y**(luminance)和两个色度**Cb**(蓝色)、**Cr**(红色)组成。Samples Per Pixel值为**3**。
  $$Y=+0.2990R+0.5870G+0.1140B$$
  $$Cb=-0.1687R-0.3313G+0.5000B+128$$
  $$Cr=+0.5000R-0.4187G-0.0813B+128$$

- YBR_FULL_422

  类似于YBR_FULL，通过色度信号来表示颜色的格式，每个像素点都有对应的亮度**Y**(luminance)，**每两个像素点采集一次色度信号**，缺少的色度信息通过内插补点的方式运算得到。
  Samples Per Pixel的值应该为3，Planar Configuration的值必须是**0**，像素存储的格式为：`Y, Y, Cb, Cr, ...`

- YBR_PARTIAL_422(Retired)

  类似于YBR_FULL_422，通过色度信号来表示颜色的格式，不过亮度和色度的计算方式和YBR_FULL的计算方式不同。
  $$Y=+0.2568R+0.5041G+0.0979B+16$$
  $$Cb=-0.1482R-0.2910G+0.4392B+128$$
  $$Cr=+0.4392R-0.3678G-0.0714B+128$$

- YBR_PARTIAL_420

  类似于YBR_PARTIAL_422，通过色度信号来表示颜色的格式，不同的是，用4:2:2的采样方式时，行方向的色度信息会被丢掉一半，而4:2:0的采样方式，不仅会把行方向的色度信息丢掉一半，列方向的色度信息也会被丢掉一半。色度的采样（Cb,Cr）只有亮度Y(luminance)的$\frac{1}{4}$。
  Samples Per Pixel的值应该为3，Planar Configuration的值必须是**0**。

- YBR_ICT

  Irreversible Color Transformation.(不可逆颜色变换)

  YCbCr的计算方式和YBR_FULL一样。Y为0时表示黑色，Cb，Cr都为0时表示没有颜色。
  JPEG 2000有损压缩的彩色图像。Samples Per Pixel的值应该为3，Planar Configuration的值必须是**0**。

- YBR_RCT

  Reversible Color Transformation.(可逆颜色变换)

  JPEG 2000无损压缩的彩色图像。Samples Per Pixel的值应该为3。
  从RGB转换到YBR_RCT
  $$Y=floor(\frac{R+2G+B}{4})$$
  $$Cb=B-G$$
  $$Cr=R-G$$
  从YBR_RCT转换到RGB
  $$R=Cr+G$$
  $$G=Y-floor(\frac{Cb+Cr}{4})$$
  $$B=Cb+G$$

- 不再使用的格式

  HSV、ARGB、CMYK

### Planar Configuration

指定颜色是按照像素来排列的或是按平面（plane）来排列的。当Samples Per Pixel大于1时应设定此值。
当值为0时表示颜色按像素排列。对于RGB图像，像素的格式为：`R1,G1,B1,R2,G2,B2,...`
当值为1时表示颜色按平面排列。对于RGB图像，像素的格式为：`R1,R2,R3,...Rn,G1,G2,G3,...Gn,B1,B2,B3,...Bn`

### Rows

图像的行数量，即图像的高(Height)。

### Columns

图像的列数量，即图像的宽(Width)。

### Bits Allocated, Bits Stored, High Bit, Pixel Representation

Bits Allocated指定每个像素分配多少位(bit)。值应当为1或者8的倍数。而对于图像像素，Bits Allocated的值通常为8或者16，实际上可以理解为每个像素分配多少字节，因为是8的倍数。

Bits Stored指定存储每个像素占用了多少位(bit)，值不能大于Bits Allocated。

High Bit则指定了像素的最高位，通常应该是`Bits Stored - 1`。

Pixel Representation指定了像素数据的类型。值只能为0或者1，对于彩色图像，值只能为0。
值为0时，表示像素为**无符号整型**（unsigned integer）。
值为1时，表示像素为**2的补码**，其实就是有符号整型，即允许存在负数。
这里一定要注意，如果不能正确处理负数，全部按照无符号整型来计算的话，就会遇到符号位的问题，即一个负数会变成一个很大的正数，图像上原本是黑色的区域会变得很亮。

### Pixel Data

图像像素。一堆二进制数字，通常会存放在Dicom文件最后的位置。

## 最后

其实，在写这篇文章之前，一些东西我都还是一知半解的，在写的过程中我也是不断的在查阅资料和源码。其中一些东西难免会掺杂了自己的理解，如果由错误的地方欢迎指正。

**参考**

- [DICOM Standard Browser](https://dicom.innolitics.com/ciods/mr-image/image-pixel/00280004)
- [颜色空间](https://www.nmm-hd.org/doc/%E9%A2%9C%E8%89%B2%E7%A9%BA%E9%97%B4)
