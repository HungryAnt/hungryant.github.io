---
layout: post
title:  "Spring Boot集成beetl模板引擎 个人总结"
date:   2017-07-24 13:53:50 +0800
categories: spring boot
---

## 1. Spring boot快速集成beetl模板引擎

> 查看官方文档：<http://ibeetl.com/guide/#beetl>{:target="_blank"}
> 可参看官方文档 4.6. Spring Boot集成

### 增加beetl包依赖

```xml
<dependency>
    <groupId>com.ibeetl</groupId>
    <artifactId>beetl</artifactId>
    <version>2.7.13</version>
</dependency>
```

### 最基础配置类编写

```java
@Configuration
public class BeetlConf {
    @Value("${beetl.RESOURCE.root:/btl}")
    private String resourceRoot;

    @Bean(name = "beetlConfig")
    public BeetlGroupUtilConfiguration getBeetlGroupUtilConfiguration() {
        BeetlGroupUtilConfiguration beetlGroupUtilConfiguration = new BeetlGroupUtilConfiguration();
        try {
            ClasspathResourceLoader resourceLoader = new ClasspathResourceLoader(
                    this.getClass().getClassLoader(), resourceRoot);
            beetlGroupUtilConfiguration.setResourceLoader(resourceLoader);
            return beetlGroupUtilConfiguration;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    @Bean(name = "beetlViewResolver")
    public BeetlSpringViewResolver getBeetlSpringViewResolver(
            @Qualifier("beetlConfig") BeetlGroupUtilConfiguration beetlGroupUtilConfiguration) {
        BeetlSpringViewResolver beetlSpringViewResolver = new BeetlSpringViewResolver();
        beetlSpringViewResolver.setPrefix("/");
        beetlSpringViewResolver.setSuffix(".btl");
        beetlSpringViewResolver.setContentType("text/html;charset=UTF-8");
        beetlSpringViewResolver.setOrder(0);
        beetlSpringViewResolver.setConfig(beetlGroupUtilConfiguration);
        return beetlSpringViewResolver;
    }
}
```

上述配置类并没有包含自定义方法及自定义类型，请继续阅读！

完成配置后即可添加开始btl文档编写，上述配置中将`resourceRoot`字段默认设置为`/btl`（可以更改为其他值），意味着btl模板文件都会位于classpath:/btl路径下

btl基本模板的编写不在此文介绍，请各位查阅官方文档

## beetl Java8 日期时间类型支持

beetl引擎提供了对Date对象的格式化支持，但对于使用Java8的项目来说Date类早已过时可以彻底丢弃

在Java8中推荐使用`LocalDate`、`LocalDateTime`类型，beetl出于兼容性考虑并没有内置对这两个类的支持

beelt的对象格式化工具支持自定义扩展，我们可以针对指定类型，实现其格式化类型

因此我对Java8日期时间类型进行如下实现：

```java
public class LocalDateFormat implements Format {

    private Map<String, DateTimeFormatter> formatterMap = new ConcurrentHashMap<>();

    @Override
    public Object format(Object data, String pattern) {
        if (data == null) {
            return null;
        }
        if (!LocalDate.class.isAssignableFrom(data.getClass())) {
            throw new RuntimeException("format failed, expectedClass:" + LocalDate.class
                    + " actualClass:" + data.getClass());
        }
        LocalDate localDate = (LocalDate) data;
        DateTimeFormatter dateTimeFormatter = genDateTimeFormatter(pattern);
        return localDate.format(dateTimeFormatter);
    }

    private DateTimeFormatter genDateTimeFormatter(String pattern) {
        if (StringUtils.isBlank(pattern)) {
            return DateTimeFormatter.ISO_LOCAL_DATE;
        }

        DateTimeFormatter formatter = formatterMap.get(pattern);
        if (formatter != null) {
            return formatter;
        }
        formatter = DateTimeFormatter.ofPattern(pattern);
        formatterMap.put(pattern, formatter);
        return formatter;
    }
}
```

```java
public class LocalDateTimeFormat implements Format {

    private Map<String, DateTimeFormatter> formatterMap = new ConcurrentHashMap<>();

    @Override
    public Object format(Object data, String pattern) {
        if (data == null) {
            return null;
        }
        if (!LocalDateTime.class.isAssignableFrom(data.getClass())) {
            throw new RuntimeException("format failed, expectedClass:" + LocalDateTime.class
                    + " actualClass:" + data.getClass());
        }
        LocalDateTime localDateTime = (LocalDateTime) data;
        DateTimeFormatter dateTimeFormatter = genDateTimeFormatter(pattern);
        return localDateTime.format(dateTimeFormatter);
    }

    private DateTimeFormatter genDateTimeFormatter(String pattern) {
        if (StringUtils.isBlank(pattern)) {
            return DateTimeFormatter.ISO_LOCAL_DATE_TIME;
        }

        DateTimeFormatter formatter = formatterMap.get(pattern);
        if (formatter != null) {
            return formatter;
        }
        formatter = DateTimeFormatter.ofPattern(pattern);
        formatterMap.put(pattern, formatter);
        return formatter;
    }
}
```

新增的类型格式化类型需要添加到beetl配置中去，改进BeetlConf如下

```java
    ...
    public BeetlGroupUtilConfiguration getBeetlGroupUtilConfiguration() {
        ...
        beetlGroupUtilConfiguration.setTypeFormats(genTypeFormats());
        ...
    }
    private Map<Class<?>, Format> genTypeFormats() {
        Map<Class<?>, Format> typeFormats = new HashMap<>();
        typeFormats.put(LocalDateTime.class, new LocalDateTimeFormat());
        typeFormats.put(LocalDate.class, new LocalDateFormat());
        return typeFormats;
    }
    ...
```

配置完成后，可以在btl中以如下方式使用

```
${entity.datetime, 'yyyy-MM-dd HH:mm:ss'}
${entity.date, 'yyyy-MM-dd'}
```

## spring方法

sputil 提供了spring内置的一些工具方法，见官方文档`5.1.5. Spring 相关函数`，如下:

```java
// 测试source中是否包含了candidates的某个成员(相当于交集非空)
sputil.containsAny(Collection<?> source, Collection<?> candidates)
// 返回在source集合总第一个也属于candidates集的元素
sputil.findFirstMatch(Collection<?> source, Collection<?> candidates)
// 测试指定文本是否匹配指定的Ant表达式(\*表达式), 多个表达式只要一个匹配即可
sputil.antMatch(String input, String... patterns)
// 返回指定路径表示的文件的扩展名(不带点.)
sputil.fileExtension(String path)
// 忽略大小写的endsWith
sputil.endsWithIgnoreCase(String input, String suffix)
// 忽略大小写的startsWith
sputil.startsWithIgnoreCase(String input, String prefix)
// 测试输入值是否为空白, null视为空白, 无视字符串中的空白字符
sputil.isBlank(String input)
// 首字母大写转换
sputil.capitalize(String input)
// 首字母小写转换
sputil.uncapitalize(String input)
// 在集合或数组元素之间拼接指定分隔符返回字符串
// null表示空集, 其他类型表示单元素集合
sputil.join(Object collection, String delim)
// 同上, 只是会在最后结果前后加上前缀和后缀
// 注意这个函数名叫做joinEx
sputil.joinEx(Object collection, String delim, String prefix, String suffix)
// 对文本进行html转义
sputil.html(String input)
// 对文本进行javascript转义
sputil.javaScript(String input)
```

可以通过如下方式集成sputil

```java
    ...
    public BeetlGroupUtilConfiguration getBeetlGroupUtilConfiguration() {
        ...
        beetlGroupUtilConfiguration.setFunctionPackages(genFunctionPackages());
        ...
    }
    public Map<String, Object> genFunctionPackages() {
        Map<String, Object> functionPackages = new HashMap<>();
        functionPackages.put("sputil", new UtilsFunctionPackage());
        return functionPackages;
    }
    ...
```

## 自定义方法

举例，自定义一个方法，支持返回一个特定的时间戳，该方法可在btl中使用

```java
    ...
    public BeetlGroupUtilConfiguration getBeetlGroupUtilConfiguration() {
        ...
        beetlGroupUtilConfiguration.setFunctions(genFunctions());
        ...
    }

    public Map<String, Function> genFunctions() {
        Map<String, Function> functionMap = new HashMap<>();
        functionMap.put("getTimestamp", (Object[] paras, Context ctx) -> TimeUtils.currentTimestampInMillis());
        return functionMap;
    }
```

可直接在btl中使用：`${ getTimestamp() }`或者`<% var timestamp = getTimestamp() %>`