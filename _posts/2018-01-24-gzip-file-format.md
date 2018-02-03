---
layout: post
title: "GZIP 文件格式"
date: "2018-01-24"
description: "GZIP 文件格式"
tag: [compression, gzip]
---

### GZIP 文件格式
```
+---+---+---+---+---+---+---+---+---+---+--------//--------+--------//--------+---+---+---+---+---+---+---+---+
|ID1|ID2|CM |FLG|     MTIME     |XFL|OS |   额外的头字段    |    压缩数据块     |     CRC32     |     ISIZE     |
+---+---+---+---+---+---+---+---+---+---+--------//--------+--------//--------+---+---+---+---+---+---+---+---+
```

### 头部分 (共十个字节)
- ID1 (IDentification 1)
- ID2 (IDentification 2)
固定值, ID1 = 31 (0x1F), ID2 = 139(0x8B), 标识文件为 GZIP 格式
- CM (Compression Method)
标识文件使用的压缩方法; 0 - 7 是保留值, CM = 8 标识为 DEFLATE 压缩方法, 这是 gzip 通常使用的一种
- FLG (FLaGs)
这个标识字节每个比特被单独划分为如下标识
  - bit 0    FTEXT    指示文本数据
  - bit 1    FHCRC    指示存在CRC16头校验字段
  - bit 2    FEXTRA   指示存在可选项字段
  - bit 3    FNAME    指示存在原文件名字段
  - bit 4    FCOMMENT 指示存在注释字段
  - bit 5-7           保留
- MTIME (Modification TIME)
被压缩的原始文件最近的修改时间, 这个时间是 Unix 格式的, 自 00:00:00 GMT, Jan. 1, 1970 开始的秒数 (注意, 对于使用本地时间而不是世界时的 MS-DOS 或其他系统可能会引起问题); 如果压缩数据不是来自于文件, MTIME 将会被设置为压缩开始的时间, MTIME = 0 意味着没有可得到的时间戳
- XFL (eXtra FLags)
对于使用指定的压缩方法, 这个标识可以被用到; deflate (CM = 8) 方法将此标识设置为如下
  - XFL = 2: 压缩器使用最大压缩但最慢算法
  - XFL = 4: 压缩器使用最快压缩算法
- OS (Operating System)
此标识压缩在那个文件系统类型下压缩的, 这有助于文本文件确定行尾 (EOF) 约定; 目前定义的值如下
  - 0 - FAT 文件系统 (MS-DOS, OS/2, NT/Win32)
  - 1 - Amiga
  - 2 - VMS/OpenVMS
  - 3 - Unix
  - 4 - VM/CMS
  - 5 - Atari TOS
  - 6 - HPFS 文件系统 (OS/2, NT)
  - 7 - Macintosh
  - 8 - Z-System
  - 9 - CP/M
  - 10 - TOPS-20
  - 11 - NTFS 文件系统 (NT)
  - 12 - QDOS
  - 13 - Acorn RISCOS
  - 255 - 未知

### 额外头字段
- 如果有设置 FLG.FEXTRA

### 压缩数据块

### 尾部分 (共八个字节)
- CRC32 (CRC-32)
此标识包含未压缩数据计算出的循环冗校验值, 是根据 ISO 3309 标准和 ITU-T 推荐 V.42 的 8.1.1.6.2 章节使用的 CRC-32 算法 (参阅 http://www.iso.ch 订购ISO文件, 参阅 gopher://info.itu.ch 在线版本的 ITU-T V.42)
- ISIZE (Input SIZE)
此标识包含原始输入数据的大小, 模 2^32
