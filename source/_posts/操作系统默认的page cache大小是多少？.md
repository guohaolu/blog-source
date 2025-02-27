---
layout: post
title: 操作系统默认的page cache大小是多少？
date: 2024-03-19 21:43:47
categories:
  - 操作系统
tags:
  - Linux
---

1. **Page Size大小**
- Linux默认页大小为4KB (4096字节)
- 可以通过命令查看：`getconf PAGE_SIZE`
- 某些系统支持大页(Huge Page)，如2MB或1GB

2. **Page Cache总大小**
- 不是固定值，是动态调节的
- 默认最大可使用所有可用物理内存
- Linux通过vm.swappiness参数调节内存与swap的权衡
- 通过/proc/sys/vm/drop_caches手动释放

3. **查看方式**
```bash
# 查看当前page cache使用情况
cat /proc/meminfo | grep -i cache

# 查看系统页大小
getconf PAGE_SIZE
```

4. **影响因素**
- 系统总内存大小
- 当前内存使用压力
- 系统IO负载情况
- 内核参数配置