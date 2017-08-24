---
title: springboot+swagger集成细节补充（二）
date: 2017-08-24 14:53:33
categories:
- java进阶
tags:
  - java
  - springboot
  - swagger
---
通过上一章，我们可以搭建出一套比较好用的在线API文档了。这套文档刚开始使用起来非常方便，但是随着接口的增多，又会有问题暴露出来:文档显得越来越臃肿，查找会变得不方便。那么是不是可以根据业务的不同对文档进行分组?接口文档是不是可以按照一定的顺序进行排序？（譬如按照开发的顺序排列）
<!-- more -->
## 准备
本文中的样例代码可参考：[https://github.com/BestRumbleCN/springboot-swagger](https://github.com/BestRumbleCN/springboot-swagger) 中的boot-swagger2。
## 分组
swagger2中接口分组非常简单，在我们的配置类Swagger2Config.java中，一个Docket就对应着一个分组，我们可以通过创建多个Docket对象实现分组。
```java
@Configuration
@EnableSwagger2
public class Swagger2Config {

	@Bean
	public Docket createDocket() {
		return new Docket(DocumentationType.SWAGGER_2).groupName("用户信息").apiInfo(userApiInfo()).select()
				.apis(RequestHandlerSelectors.basePackage("pri.fly.leaning.controller"))
				.paths(PathSelectors.any()).build();
	}

	private ApiInfo userApiInfo() {
		return new ApiInfoBuilder().title("在线接口文档").description("用户相关接口").termsOfServiceUrl("http://localhost:8080")
				.contact(new Contact("fly", "https://github.com/BestRumbleCN", "flyxie2009@foxmail.com")).version("1.0")
				.build();
	}
	
	//配置多个Docket实例
	@Bean
	public Docket createDocket2() {
		...
	}

}

```
具体的分组方式主要有两种：1.通过路径分组 2.通过注解分组
#### 路径分组
**通过RequestHandlerSelectors.basePackage("")指定扫描的包路径。**
假设我们在做一个学校的管理系统，后台用户分为校长、老师、学生。我们可以按照如下的方式对controller进行细分。
![](http://palyboy.qiniudn.com/9.png)
在我们的Swagger2Config.java可以配置三个Docket
```java
	@Bean
	public Docket createPresidentDocket() {
		return new Docket(DocumentationType.SWAGGER_2).groupName("校长").apiInfo(presidentApiInfo()).select()
				.apis(RequestHandlerSelectors.basePackage("pri.fly.leaning.controller.president"))
				.paths(PathSelectors.any()).build();
	}
	
	@Bean
	public Docket createStudentDocket() {
		return new Docket(DocumentationType.SWAGGER_2).groupName("学生").apiInfo(studentApiInfo()).select()
				.apis(RequestHandlerSelectors.basePackage("pri.fly.leaning.controller.student"))
				.paths(PathSelectors.any()).build();
	}

	@Bean
	public Docket createTeacherDocket() {
		return new Docket(DocumentationType.SWAGGER_2).groupName("老师").apiInfo(teacherApiInfo()).select()
				.apis(RequestHandlerSelectors.basePackage("pri.fly.leaning.controller.teacher"))
				.paths(PathSelectors.any()).build();
	}
```

现在可以通过下拉列表切换不同的分组啦。
![](http://palyboy.qiniudn.com/10.jpg)
#### 注解分组
**通过RequestHandlerSelectors.withClassAnnotation()指定扫描的包路径。**
上文提到的路径分组是一种分组方式，但是这种分组的局限比较大，万一我们的代码中没有按照这种方式分包那就无法分组。下面介绍一个更灵活的方式：注解分组。
仍然以学校管理系统为例，校长和老师都属于学校管理层，我们希望将他们放到同一个分组中。首先定义我们自己的分组注解。
```java
/**
 * 学校管理层分组注解
 * @author fly
 *
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface Manager {

}
```
在对应的Controller上加上注解
```java
@RestController
@RequestMapping("/teacher/users")
@Api(description = "老师班级管理接口")
@Manager
public class ClassController {
	....
}
@RestController
@RequestMapping("/president/school")
@Api(description = "校长学校管理接口")
@Manager
public class SchoolController {
	....
}
```
配置Docket
**核心：RequestHandlerSelectors.withClassAnnotation(Manager.class)**
```java
@Bean
public Docket createManagerDocket() {
	return new Docket(DocumentationType.SWAGGER_2).groupName("管理者").apiInfo(managerApiInfo()).select()
			.apis(RequestHandlerSelectors.withClassAnnotation(Manager.class))
			.paths(PathSelectors.any()).build();
}
```
再次发布打开我们的在线文档，可以看到已经有管理员分组了。
![](http://palyboy.qiniudn.com/11.png)
注意：同一接口可以被包含在多个分组中。
## 排序
通过分组，我们的接口文档显得更加清晰了，但是还不够完美，同一分组下的接口仍然可能会有很多，我们希望能够对接口进行排序。举个例子，后台管理系统中，用户登陆后有一系列的菜单栏，前端开发希望我们的接口顺序能够和菜单栏一一匹配。
![](http://palyboy.qiniudn.com/12.png)
虽然Docket类中提供了一个排序方法 operationOrdering，但是实际上并不起作用！方法注释上也说了该方法并不起效果。多方搜索，在stackoverflow上看到了spring-swagger一个作者说可以通过tags进行排序（我现在找不到说这句话的具体地方了），几番实验终于成功。
仍以学校管理系统为例，给老师增加一个班级学生的管理接口。
![](http://palyboy.qiniudn.com/13.png)
此时的管理者分组接口如下图：
![](http://palyboy.qiniudn.com/14.png)
这里的接口排序是 1.class 2.school 3.student，假设我们希望按照由大到小的方式排列（1.school 2.class 3.student），那么需要修改@Api注解中的tags属性。以SchoolController为例
```java
@RestController
@RequestMapping("/president/school")
@Api(tags = {"1.0.1"},description = "校长-学校管理接口")
@Manager
public class SchoolController {
}
```
这时的管理者分组就达到我们期望的效果了。
![](http://palyboy.qiniudn.com/15.png)

注意：
- tags尽量使用数字字母组合，当使用中文或tags过长时，会导致文档无法打开。
- 当tags使用 1.0.1这种格式时，如果数字大于10,排序会有问题，如：10.0.1会排在1.0.1前面（因为tags排序是一位一位的比较），所以建议超过10的时候1用I代替，如下图![](http://palyboy.qiniudn.com/16.png)

至此，一个完善的在线接口文档完成啦。

