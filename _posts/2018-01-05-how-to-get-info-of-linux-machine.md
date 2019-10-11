---
layout: post
title: "如何获取 Linix 机器信息"
date: "2018-01-05"
description: "如何获取 Linix 机器信息"
tag: [linux, centos]
---

### CPU
总核数 = 物理 CPU 个数 * 每颗物理 CPU 的核数
总逻辑 CPU 数 = 物理 CPU 个数 * 每颗物理 CPU 的核数 * 超线程数
#### 查看物理 CPU 个数
cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l
#### 查看每个物理 CPU 中 core 的个数(即核数)
cat /proc/cpuinfo| grep "cpu cores"| uniq
#### 查看逻辑 CPU 的个数
cat /proc/cpuinfo| grep "processor"| wc -l
#### 查看CPU型号
cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq
#### 查看CPU信息
cat /proc/cpuinfo

### Memory
#### 查看内存信息
cat /proc/meminfo
