---
title: mybatis连接池的实现
copyright: true
date: 2021-03-22 01:01:36
tags:
	- mybatis
	- 连接池
categories:
	- mybatis
---

## 前言

最近看了Mybatis自带的数据源连接池的实现，在对比其他数据源的复杂实现，mybatis比较轻量简单的实现了数据库连接池化管理。这里写下来记录下。