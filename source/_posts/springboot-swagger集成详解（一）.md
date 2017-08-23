---
title: springboot+swagger集成细节补充（一）
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
## 准备
网上有一篇介绍springboot集成swagger的文章我觉得写的特别好：[http://www.jianshu.com/p/8033ef83a8ed](http://www.jianshu.com/p/8033ef83a8ed "Spring Boot中使用Swagger2构建强大的RESTful API文档") ，非常适合入门使用，本文主要是该篇文章的补充。
本文中的样例代码可参考：[https://github.com/BestRumbleCN/springboot-swagger](https://github.com/BestRumbleCN/springboot-swagger) 中的boot-swagger1。
## 使用详解
#### 类上注解 @Api
@Api 主要使用于接口controller类上，可以不加，但当不加时，生成的在线文档接口目录会使用默认的controller名称，如下图：
![](http://palyboy.qiniudn.com/1.png)
不够直观也不美观。推荐加上，@ApiModel中有两个属性建议设置值：description和tags。其中description作用于文档目录后缀名，tags作用于文档目录前缀名，都加上以后对应的文档如下图。
![](http://palyboy.qiniudn.com/3.png)
![](http://palyboy.qiniudn.com/2.png)
tags还有一个重要的排序功能，将在下篇文章中具体体现。

#### 方法上注解 @ApiImplicitParams 等

首先看一下常见的加了swagger注解的接口方法：
@ApiOperation 为接口的描述，@ApiImplicitParams 为多参数的数组聚合，@ApiImplicitParam为具体的各个参数的描述，@ApiImplicitParam中有5个常用的属性：
① name ： 参数名称
② value ： 参数描述
③ dataType: 参数类型 这里支持常用的基础数据类型 String long int boolean等。
④ required: 是否必填
⑤ defaultValue：文档中默认填写的值
⑥ paramType : 传参方式。支持：path query body header form，**注意这里的传参方式一定要和方法中的springmvc传参注解保持一致**，不然接口文档会生成两个名称相同但是传参方式不同的参数。
![](http://palyboy.qiniudn.com/5.png)

#### 入参对象及返回对象上注解
swagger针对返回值对象和入参（responseBody方式传入）对象，都会自动解析生成对应的json结构。
![](http://palyboy.qiniudn.com/6.png)
但是这还是不够的，我们希望能加上针对具体字段的描述。那么我们可以使用@ApiModel和@ApiModelProperty，以下是具体的代码。
![](http://palyboy.qiniudn.com/7.png)
对应的接口文档，我们点击model,可以看到
![](http://palyboy.qiniudn.com/8.png)
当然@ApiModelProperty还有许多使用的参数可以使用，可以在实践中慢慢探索，譬如sample，可以提供具体的参数样例。
ok,至此，一个比较完善的在线接口文档完成啦。