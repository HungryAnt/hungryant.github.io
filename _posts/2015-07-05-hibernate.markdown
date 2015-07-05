---
layout: post
title:  "Hibernate实践"
date:   2015-07-05 22:35:47
categories: java
---

## 环境准备

开发环境: Intellij IDEA

本地mysql数据库

[Demo源码下载](https://github.com/HungryAnt/hibernate-demo){:target="_blank"}

## 创建Project

新增项目，命名为 hibernate-demo

创建目录结构 如下

	hibernate-demo
		/src
			/main
				/java
				/resources	
			/test
				/java
				/resources	
		pom.xml


## 依赖项配置

pom.xml中配置如下依赖项

    <dependencies>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.35</version>
        </dependency>

        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-core</artifactId>
            <version>5.0.0.CR1</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.8.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

这里使用当前最新版的 Hibernate类库 以及 MySql JDBC驱动，也添加junit用作单元测试

[Core Hibernate O/RM Functionality » 5.0.0.CR1](http://mvnrepository.com/artifact/org.hibernate/hibernate-core/5.0.0.CR1){:target="_blank"}

[MySQL Java Connector » 5.1.35](http://mvnrepository.com/artifact/mysql/mysql-connector-java/5.1.35){:target="_blank"}

## 创建hibernate配置

在`/src/main/resources`目录下添加`hibernate.cfg.xml`配置文件，内容如下

	<?xml version='1.0' encoding='utf-8'?>
	<!DOCTYPE hibernate-configuration PUBLIC
	        "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
	        "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
	
	<hibernate-configuration>
	    <session-factory>
	        <!-- Database connection settings -->
	        <property name="connection.driver_class">com.mysql.jdbc.Driver</property>
	        <property name="connection.url">jdbc:mysql://localhost/ant_test?useUnicode=true&amp;charseterEncoding=UTF-8</property>
	        <property name="connection.username">root</property>
	        <property name="connection.password">123</property>
	
	        <!-- JDBC connection pool (use the built-in) -->
	        <!--<property name="connection.pool_size">1</property>-->
	
	        <!-- SQL dialect -->
	        <property name="dialect">org.hibernate.dialect.MySQLDialect</property>
	
	        <!-- Enable Hibernate's automatic session context management -->
	        <property name="current_session_context_class">thread</property>
	
	        <!-- Echo all executed SQL to stdout -->
	        <property name="show_sql">true</property>
	        <property name="format_sql">true</property>
	
	        <!-- Drop and re-create the database schema on startup -->
	        <property name="hbm2ddl.auto">create</property>
	
	        <mapping resource="demo/Event.hbm.xml"/>
	    </session-factory>
	</hibernate-configuration>

`connection.driver_class`指定为 Mysql jdbc驱动

`connection.url`设置数据库访问地址，指定数据库名，这里是`ant_test`

`connection.username`与`connection.password`顾名思义

关于连接池，Hibernate comes with support for two third-party open source JDBC connection pools: [c3p0](https://sourceforge.net/projects/c3p0){:target="_blank"} and [proxool](http://proxool.sourceforge.net/){:target="_blank"}

`<property name="current_session_context_class">thread</property>`

注意这里将`current_session_context_class`设置为 thread，后文有描述



`<mapping resource="demo/Event.hbm.xml"/>`关联类型映射文件，见后文

## 定义实体类

同官方文档，实现定义一个Event类型，我这里新增`/src/main/java/demo`包，其中添加Event.java

	public class Event {
	    private Long id;
	
	    private String title;
	    private Date date;
	
	    public Event() {}
	
	    public Long getId() {
	        return id;
	    }
	
	    private void setId(Long id) {
	        this.id = id;
	    }
	
	    public Date getDate() {
	        return date;
	    }
	
	    public void setDate(Date date) {
	        this.date = date;
	    }
	
	    public String getTitle() {
	        return title;
	    }
	
	    public void setTitle(String title) {
	        this.title = title;
	    }
	}

可以注意到，这里setId方法访问权限设置为private，这是hibernate官方推荐的做法，id一般由hibernate来维护。hibernate通过反射可以访问方法与字段，即使设置了private与protected访问权限


## 定义映射文件

在`/src/main/resources/demo`中新增`Event.hbm.xml`:

	<?xml version="1.0"?>
	<!DOCTYPE hibernate-mapping PUBLIC
	        "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
	        "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">
	
	<hibernate-mapping package="demo">
	    <class name="Event" table="events">
	        <id name="id" column="id">
	            <generator class="native"/>
	        </id>
	        <property name="date" type="timestamp" column="date"/>
	        <property name="title"/>
	    </class>
	</hibernate-mapping>

映射文件以类名称开头，以hbm.xml 结尾


## 创建工具类型HibernateUtility

官方文档中HibernateUtil创建SessionFactory的方式可能由于版本或者其他原因，在我的机器上测试无法正确加载hibernate配置数据，修正后，内容如下：

	/**
	 * Created by ant on 2015/7/5.
	 */
	public class HibernateUtility {
	    private static final SessionFactory sessionFactory = buildSessionFactory();
	
	    private static SessionFactory buildSessionFactory() {
	        try {
	            Configuration configuration = new Configuration().configure();
	            configuration.addClass(Event.class);
	            ServiceRegistry serviceRegistry = new StandardServiceRegistryBuilder()
	                    .applySettings(configuration.getProperties()).build();
	            return configuration.buildSessionFactory(serviceRegistry);
	        }
	        catch (Throwable ex) {
	            // Make sure you log the exception, as it might be swallowed
	            System.err.println("Initial SessionFactory creation failed." + ex);
	            throw new ExceptionInInitializerError(ex);
	        }
	    }
	
	    public static SessionFactory getSessionFactory() {
	        return sessionFactory;
	    }
	}

> The getCurrentSession() method always returns the "current" unit of work. Remember that we switched the configuration option for this mechanism to "thread" in our src/main/resources/hibernate.cfg.xml? Due to that setting, the context of a current unit of work is bound to the current Java thread that executes the application.
>
>
> **Important**
> 
> Hibernate offers three methods of current session tracking. The "thread" based method is not intended for production use; it is merely useful for prototyping and tutorials such as this one. Current session tracking is discussed in more detail later on.

## 下面开始使用hibernate

在`/src/main/java`目录中新增入口类型`Demo.java`，内容如下:

	/**
	 * Created by ant on 2015/7/5.
	 */
	public class Demo {
	    public static void main(String[] args) {
	        Session session = HibernateUtility.getSessionFactory().getCurrentSession();
	        session.beginTransaction();
	
	        Event event = new Event();
	        event.setTitle("hello hibernate");
	        event.setDate(new Date());
	        session.save(event);
	
	        session.getTransaction().commit();
	
	        System.out.println("done!");
	    }
	}

可以在集成开发环境中直接运行，将会自动在`ant_test`数据库中新增`events`表，同时将新插入一行数据

之所以可以自动构建表，是有与在`hibernate.cfg.xml`中做了如下设置

	<property name="hbm2ddl.auto">create</property>

可以将该值修改为`update`


可以通过命令执行程序

编译:　`mvn compile`

执行: `mvn exec:java -Dexec.mainClass="Demo"`

可以通过`-Dexec.args="arg"`补充参数


## 参考
[官方文档](http://docs.jboss.org/hibernate/orm/4.3/manual/en-US/html/ch01.html#tutorial-firstapp-setup){:target="_blank"}