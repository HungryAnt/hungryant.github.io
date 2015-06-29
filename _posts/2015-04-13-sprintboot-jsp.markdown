---
layout: post
title:  "spring boot 框架构建jsp web应用"
date:   2015-04-13 00:35:00
categories: java
---


### 说明 ###
**Spring boot**支持将web项目打包成一个可执行的jar包，内嵌tomcat服务器，独立部署

为支持jsp，则必须将项目打包为**war包**

pom.xml中设置打包方式

	<packaging>war</packaging>

### 依赖包导入 ###
Srping boot web项目原本会包含依赖项（starter-web模块内部依赖包含了spring-boot-starter-tomcat模块）

	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>


增加如下依赖

	<dependency>
		<groupId>org.apache.tomcat.embed</groupId>
		<artifactId>tomcat-embed-jasper</artifactId>
		<scope>provided</scope>
	</dependency>
	<dependency>
		<groupId>javax.servlet</groupId>
		<artifactId>jstl</artifactId>
		<scope>provided</scope>
	</dependency>

注意：若使用**intellij**作为集成开发环境，则scope需要设置为compile `<scope>compile</scope>`

[参考](https://github.com/spring-projects/spring-boot/issues/1194)

>I can confirm that the IntelliJ error happens for me when I try to run a @SpringBootApplication class directly (without using maven/gradle). It happens in both versions on IntelliJ.
>
>It is fixed temporarily if you follow the instructions @xilin, but these changes are overridden anytime the Gradle project is reimported (and it's annoying to have to tell new devs to do this).

### 建立jsp文件目录 ###
在main目录下与resources同级别，建立目录webapp/WEB-INF/jsp/，所有jsp文件将放置与此目录中
配置ViewResolver，指定jsp目录

	/**
	 * Created by Ant on 2015/4/11.
	 * QQ:517377100
	 */
	@Configuration
	public class JspConfiguration {
		@Bean
		InternalResourceViewResolver internalResourceViewResolver () {
			InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
			viewResolver.setPrefix("/WEB-INF/jsp/");
			viewResolver.setSuffix(".jsp");
			return viewResolver;
		}
	}


### 资源目录 ###
资源目录直接位于webapp目录下，如图片目录为 webapp/imgs/

假设目录结构如下，welcome.jsp中以 `<img src="imgs/spring.jpg" />` 引用图片资源
	
	webapp/
	imgs/
		spring.jpg
	WEB-INF/
		jsp/
		welcome.jsp
	

### 启动服务 ###
通过 mvn package 完成编译打包，target目录中将生成可执行的xx.war文件

通过 java -jar xx.war 启动服务