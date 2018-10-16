---
title: Spring Boot 开发者工具
date: 2017-10-17 16:03:14
tags:
- Spring
- Spring Boot
categories:
- Tech
---
Spring Boot 自 1.3 版本起包含一组开发者工具，其中的 **自动重启** 和 **LiveReload 支持** 可以让开发过程更加便捷。

使用前需向工程中添加开发者工具，如 maven：

```xml
<dependency>
 	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-devtools</artifactId>
</dependency>
```

  程序以完整打包好的 JAR 或 WAR 文件形式运行时，开发者工具会被禁用，无需在生产部署前移除依赖。



<!--more-->



## 自动重启

添加开发者工具后，classpath 中对文件的任何修改都会触发程序的重启。为了速度够快，不会修改的类（如第三方 JAR）会加载到基础类加载器中，应用程序的代码会单独加载到重启类加载器里。检测到变更时只有重启类加载器会被重启。

默认情况下，Spring Boot 会排除掉如下目录：**/META-INF/resources、/resources、/static、/public 和 /templates**

通过 `spring.devtools.restart.exclude` 属性可以覆盖默认的重启排除目录，如 `spring.devtools.restart.exclude=/static/**, /templates/**`。

通过 `spring.devtools.restart.enabled=false` 可以彻底关闭自动重启。

需要注意的是，Intellij IDEA 默认关闭自动编辑功能，需在 **preferences -- Build, Execution, Deployment --Compiler** 中打开自动编辑，如图

![preferences](http://image.ulyssesss.com/201710171.png)



然后按 `Shift + Command + Alt + /`，选择 **Registry**，勾选 `compiler.automake.allow.when.app.running`，如图

![registry](http://image.ulyssesss.com/201710172.png)

设置完毕并重启后生效。





## LiveReload

在 Web 应用开发过程中，比较常见的步骤包含：

1.修改要呈现的内容，如image、css、template。

2.刷新浏览器。

3.重复第一步。

Spring Boot 的开发者工具会在启动一个内嵌的 LiveReload 服务器，在资源文件变化时自动触发浏览器刷新，唯一要做的就是在浏览器中安装 LiveReload (http://livereload.com) 插件。

通过配置 `spring.devtools.livereload.enable=false` 可禁用 LiveReload 功能。
