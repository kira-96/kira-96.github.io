---
title: "Private Dicom Element"
date: 2021-01-21T15:31:46+08:00
tags: [ "DICOM" ]
keywords: [ "DICOM" ]
categories: [ "DICOM" ]
draft: true
isCJKLanguage: true
---

**规则**

私有 Tag (gggg, xxxx)

1. Group number (gggg) 必须为奇数 (odd)，并且 (0001, xxxx)，(0003, xxxx)，(0005, xxxx)，(0007, xxxx)，(FFFF, xxxx) 不允许使用。
2. (gggg, 0000) were Group Length Elements, which have been retired.（已弃用）
3. (gggg, 0001-000F)，(gggg, 0100-0FFF) 不允许使用。
4. (gggg, 0010-00FF) 供私有tag创建者（Private Creator）使用，用于在该group中插入一个未使用的标识码（identification code），私有标识码的VR应该为LO (Long String)，VM应该为1。
5. (gggg, 1000-FFFF) 为 Data Element。
6. Private Creator 和 Data Element 的对应关系为：

    例：

    Data Element (0029, 1000-10FF) 的 Private Creator 是 (0029, 0010)

    Data Element (0029, 1100-11FF) 的 Private Creator 是 (0029, 0011)

    Data Element (0029, 1200-12FF) 的 Private Creator 是 (0029, 0012)

    ……

    Data Element (0029, FF00-FFFF) 的 Private Creator 是 (0029, 00FF)

**标准**

http://dicom.nema.org/medical/dicom/current/output/chtml/part05/sect_7.8.html
