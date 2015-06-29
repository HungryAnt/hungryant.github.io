---
layout: post
title:  "Spring Boot + Mybatis 简易使用指南（一）基础环境搭建"
date:   2015-04-13 21:59:00
categories: java
---


#### 前言 ###

作者: **Ant**  QQ:**517377100**

相对于使用JdbcTemplate，Mybatis可自动建立pojo类型与数据库列的映射关系，数据库访问层的开发简单了许多

所有数据库访问操作，均封装在各个Mapper接口中，接口的实现即为数据库sql操作，sql可以注解的形式提供，也可以定义在xml文件中（复杂的sql操作优选xml）

引入Mybatis框架步骤简单，这里做一些整理

本人集成开发环境使用 Intellij

#### 添加Mybatis依赖项 ###
pom.xml增加如下依赖项

	<dependency>
		<groupId>org.mybatis</groupId>
		<artifactId>mybatis</artifactId>
		<version>3.2.8</version>
	</dependency>

	<dependency>
		<groupId>org.mybatis</groupId>
		<artifactId>mybatis-spring</artifactId>
		<version>1.2.2</version>
	</dependency>

#### Bean声明 ####
相关Bean的声明统一在DatabaseConfiguration类型中

其中包含了UserMapper、CommodityMapper、CommodityCategoryMapper三个Bean的声明


代码如下

	@Configuration
	public class DatabaseConfiguration {
		@Bean
		public DataSource dataSource() {
			...
		}

		@Bean
		public SqlSessionFactory sqlSessionFactory() throws Exception {
			SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
			sqlSessionFactoryBean.setDataSource(dataSource());
			sqlSessionFactoryBean.setTypeAliases(new Class[]{User.class, Commodity.class, CommodityCategory.class});
			sqlSessionFactoryBean.getObject().getConfiguration().setMapUnderscoreToCamelCase(true);
			return sqlSessionFactoryBean.getObject();
		}
		
		private <T> MapperFactoryBean getMapper(Class<T> mapperInterface) {
			MapperFactoryBean<T> mapperFactoryBean = new MapperFactoryBean<T>();
			try {
				mapperFactoryBean.setSqlSessionFactory(sqlSessionFactory());
				mapperFactoryBean.setMapperInterface(mapperInterface);
			} catch (Exception ex) {
				logger.error("error when create mapper: ", ex);
				throw new RuntimeException(ex);
			}
			return mapperFactoryBean;
		}
		
		@Bean
		public MapperFactoryBean userMapper() {
			return getMapper(UserMapper.class);
		}
		
		@Bean
		public MapperFactoryBean commodityCategoryMapper() {
			return getMapper(CommodityCategoryMapper.class);
		}
		
		@Bean
		public MapperFactoryBean commodityMapper() {
			return getMapper(CommodityMapper.class);
		}
	}

创建SqlSessionFactory的同时，将其配置项**MapUnderscoreToCamelCase**设置为true，如数据库列 user_name将自动映射到pojo中的userName属性

通过**setTypeAliases**，指定使用的Pojo类型，后续Mapper.xml中就不需要指定Pojo类型的完整限定名（即无需指定namespace）

#### Mapper interface ####

	package com.antsoft.docoding.mapper;
	import java.util.List;
	import com.antsoft.docoding.model.User;

	public interface UserMapper {
		List<User> getUsers();

		User getUser(long id);

		void addUser(User user);

		void clear();

		void updateUser(User user);
	}


#### Mapper xml ####
Mapper.xml所在目录需要与对应的Mapper接口位于统一个包中

上述UserMapper接口，对应的UserMapper.xml 位于目录resources/com/antsoft/docoding/mapper/中，内容如下

	<?xml version="1.0" encoding="UTF-8" ?>
	<!DOCTYPE mapper
			PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
			"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
	<mapper namespace="com.antsoft.docoding.mapper.UserMapper">
		<resultMap id="user" type="user">
			<result column="user_type" property="userType" typeHandler="org.apache.ibatis.type.EnumOrdinalTypeHandler"/>
		</resultMap>

		<delete id="clear">
			DELETE FROM dc_user
		</delete>

		<select id="getUsers" resultMap="user">
			SELECT id, user_name, user_type FROM dc_user
		</select>

		<select id="getUser"  resultMap="user">
			SELECT id, user_name, user_type FROM dc_user WHERE id = #{id}
		</select>

		<insert id="addUser" useGeneratedKeys="true" keyProperty="id" parameterType="user">
			INSERT INTO dc_user(user_name, user_type)
			VALUES(#{userName}, #{userType, typeHandler=org.apache.ibatis.type.EnumOrdinalTypeHandler})
		</insert>

		<update id="updateUser" parameterType="user">
			UPDATE dc_user SET user_name = #{userNmae},
			user_type = #{userType, typeHandler=org.apache.ibatis.type.EnumOrdinalTypeHandler}
			WHERE id = #{id}
		</update>
	
	</mapper>

关于**EnumOrdinalTypeHandler**，可以完成枚举值叙述与枚举类型的自动转换


#### 使用Mapper ####
使用Mapper的方式与以前使用Dao类型的方法完全一致，通过**@Autowired**实现依赖注入，如下：

	@Service
	public class UserService {
		@Autowired
		private UserMapper userMapper;
		
		public void clear() {
			userMapper.clear();
		}
		
		public List<User> getUsers() {
			return userMapper.getUsers();
		}
		
		public void addUser(User user) {
			userMapper.addUser(user);
		}
		
		public void updateUser(User user) {
			userMapper.updateUser(user);
		}
		
		public User getUser(long id) {
			return userMapper.getUser(id);
		}
	}

#### 更多Mabatis内容 请阅读官方文档 ####
[入门](http://mybatis.github.io/mybatis-3/zh/getting-started.html)

[动态SQL 简化工作](http://mybatis.github.io/mybatis-3/zh/dynamic-sql.html)