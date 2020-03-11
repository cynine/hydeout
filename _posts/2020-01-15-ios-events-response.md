---
layout: post
title: 深入理解 iOS 事件机制
excerpt_separator: "<!--more-->"
categories:
  - iOS
tags:
  - iOS
  - Objective-C
  - 知识体系
  - 事件机制
last_modified_at: 2020-01-15T14:09:01
---

#### 问题

- View 的事件是怎么传递和响应的？
- 事件传递过程中如何找到最合适的 View?
- View 如果响应事件？

## iOS中的事件

iOS 的事件分为 **Touch Events(触摸事件)**、**Motion Events(运动事件，比如重力感应和摇一摇等)**、**Remote Events(远程事件，比如用耳机上得按键来控制手机)**。日常开发中，最常用到的就是触摸事件，这里也是深入学习下iOS