---
title: springboot+swagger集成详解（一）
date: 2017-08-22 20:04:53
categories:
- java进阶
tags:
  - java
  - springboot
  - swagger
---
网上有很多springboot集成swagger的例子，但基本都是简单的demo，在实际的开发中无法给出更多的参考价值。特别是当接口数量越来越多的时候，会碰到以下几个问题：
① 针对各种不同的接口传参方式及返回格式，该如何去写？
② 当接口文档中的接口越来越多时，能不能做分组，方便更好地查？
③ 是否可以对各个接口进行排序？
本系列博文将针对以上问题分为三章：
第一章介绍详细的swagger使用及解决问题①；
第二章解决问题②问题③；
第三章介绍下基于springboot-starter封装我们自己的swagger-starter。
<!-- more -->
1

