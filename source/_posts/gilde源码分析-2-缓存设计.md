---
title: gilde源码分析(2)-缓存设计
date: 2024-07-16 22:39:27
tags: [Android,glide,源码分析]
categories: Android三方库源码分析
---

### glide缓存设计

glide有内存缓存和磁盘缓存两种

### 内存缓存

#### active

active表示**正在被其他view使用的资源**，通过ActiveResources进行管理。该对象随engine初始化。
activeResources通过一个map存储。从内存加载或者加载完毕时会放入，engine执行relesase时会释放。使用弱引用。

#### memory cache

**策略**：LRU
**来源**：active resources

问：为什么有活动缓存？
答：确保正在使用的图片长时间未加载不被移除缓存

### 磁盘缓存

两种，一种时资源缓存，一种是数据缓存

分别对应ResourceCacheGenerator和DataCacheGenerator
<!--more-->
