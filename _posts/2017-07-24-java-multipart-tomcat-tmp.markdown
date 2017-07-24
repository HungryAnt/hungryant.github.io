---
layout: post
title:  "Spring Boot上传文件路径错误解决"
date:   2017-07-24 13:39:18 +0800
categories: spring
---

## 文件上传错误

今天线上系统客户反馈上传文件出现问题，排查发现错误日志如下：

![错误日志](/image/20170724/01.png)

    The temporary upload location [/tmp/tomcat.2353700999131105369.8110/work/Tomcat/localhost/ROOT] is not valid

检测发现`/tmp`目录下不存在`tomcat.2353700999131105369.8110`子目录

`/tmp`目录中的文件有被系统自动清理的可能，因此最好是将次目录设置为当前应用的子目录

搜索针对该问题的解决方案，如下：

[wuzhaoyang的个人博客 - The temporary upload location is not valid](http://wuzhaoyang.me/2017/06/07/spring-multipartexception-location-not-valid.html){:target="_blank"}

[解决使用Spring Boot、Multipartfile上传文件路径错误问题](http://blog.csdn.net/daniel7443/article/details/51620308){:target="_blank"}

采用上述方案可以解决问题

延伸思考：

临时文件目录是否可以直接置于应用目录下？ 这样部署时直接创建好子目录，多应用互不干扰

最终采用如下解决方案

## 解决方案

### 一、应用获取所在目录

在我的项目中，有专门的sh文件用于服务启停，由于是使用spring boot，直接通过 java -jar xxx启动服务

```shell
SUPERVISE_CMD="java ${APP_PROPERTIES_ARG} -Djava.security.egd=file:/dev/urandom \
-Dfile.encoding=UTF-8 -Xmx$MAX_MEMORY \
-XX:MaxPermSize=$MAX_PERM_MEMORY -XX:+UseConcMarkSweepGC -XX:+PrintGCDetails -XX:+PrintGCDateStamps \
-Xloggc:gc.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=20M \
-DprojectPath=$PRO_HOME \
-jar $PRO_MODULE "
```

仅需关注参数`-DprojectPath=$PRO_HOME \`，用于增加系统变量`projectPath`，值为当前应用所在目录，其中`PRO_HOME`定义在脚本首部

```shell
#!/bin/bash
[ $# -lt 1 ] && exit -1

PRO_HOME=$(dirname $(readlink -f $0))/..
```

### 二、应用根目录中创建子目录

约定子目录为`data/tmp`

在项目的构建脚本中添加指令`mkdir ${PROJ_OUT_DIR}/data/tmp -p`用于创建子临时目录

### 三、设置Multipart文件上传临时目录路径

在项目Java代码中添加类`MultipartConfig`:

```Java
@Configuration
public class MultipartConfig {
    @Value("${projectPath:}")
    private String projectPath;

    @Bean
    MultipartConfigElement multipartConfigElement() {
        MultipartConfigFactory factory = new MultipartConfigFactory();
        String location = StringUtils.isNotBlank(projectPath) ? projectPath + "/data/tmp" : "/data/tmp";
        factory.setLocation(location);
        return factory.createMultipartConfig();
    }
}
```

