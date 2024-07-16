---
title: gilde源码分析(2)-缓存设计
date: 2024-07-16 22:39:27
tags: [Android,glide,源码分析]
---

glide缓存设计

glide有内存缓存和磁盘缓存两种，内存缓存分为active和memory cache。

active表示正在被其他view使用的资源，通过ActiveResources进行管理。该对象随engine初始化。

activeResources通过一个map存储。从内存加载或者加载完毕时会放入，engine执行relesase时会释放。使用弱引用。

memorycache：lru

来源：active

磁盘缓存：两种，一种时资源缓存，一种是数据缓存

分别对应ResourceCacheGenerator和DataCacheGenerator

为什么有活动缓存？

确保正在使用的图片长时间未加载不被移除缓存