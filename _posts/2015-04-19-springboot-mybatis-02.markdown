---
layout: post
title:  "Spring Boot + Mybatis 简易使用指南（二）多参数方法支持 与 Joda DateTime类型支持"
date:   2015-04-19 23:51:00
categories: java
---

### 前言 ###
今天在开发练习项目时遇到两个mybatis使用问题

第一个问题是mapper方法参数问题，在参数大于一个时，mybatis不会自动识别参数命名

第二个问题是Pojo中使用Joda DateTime类型的字段，mybatis并不天然支持DateTime，这个问题想必有众多开发者都遇到过

两个问题均得到有效解决，在此进行总结记录


### Mapper中针对多参数的方法正确声明方式 ###
	
**UserMapper** interface中包含如下方法定义，用于删除用户权限

	// 错误定义，此为演示
	void deleteAuthority(String username, Authority authority);

对应**UserMapper.xml**中内容如下

	<delete id="deleteAuthority">
		DELETE FROM authorities WHERE username = #{username} AND authority = #{authority};
	</delete>

执行时出错，提示无法找到username与authority参数

修复方式，在方法声明中，参数增加**@Param**注解，如下

	void deleteAuthority(@Param("username") String username, @Param("authority") Authority authority);


### 支持Joda DateTime ###
例如 Pojo/Model 类型定义如，ctime表示用户创建时间，使用joda DateTime类型

		public class User {
				private String username;
				private String password;
				private DateTime ctime;
				private List<UserAuthority> authorities;
		
				... 各种 get set 方法
		}

mapper中addUser方法定义如下，对于ctime字段，Mybatis默认不支持DateTime类型，且没有提供对应的TypeHander，因此需自行实现，如下的实现必然报错

	错误案例
	<insert id="addUser" parameterType="user">
		INSERT INTO users(username, password, ctime) VALUES(#{username}, #{password}, #{ctime})
	</insert>


**自定义实现DateTimeTypeHandler**

参考此处外国友人的讨论 [Mybatis Joda Time Support](http://mybatis-user.963551.n3.nabble.com/Mybatis-Joda-Time-Support-td3629891.html)

参考其源码 [LukeL99/joda-time-mybatis的实现](https://github.com/LukeL99/joda-time-mybatis/blob/master/src/main/java/org/joda/time/mybatis/handlers/DateTimeTypeHandler.java#L1)

以下是是本人对**DateTimeTypeHandler**的实现，前人基础上稍作重构

	public class DateTimeTypeHandler implements TypeHandler<DateTime> {
		@Override
		public void setParameter(PreparedStatement preparedStatement, int i, DateTime dateTime, JdbcType jdbcType)
		throws SQLException {
			if (dateTime != null) {
				preparedStatement.setTimestamp(i, new Timestamp(dateTime.getMillis()));
			} else {
				preparedStatement.setTimestamp(i, null);
			}
		}

		@Override
		public DateTime getResult(ResultSet resultSet, String s) throws SQLException {
			return toDateTime(resultSet.getTimestamp(s));
		}

		@Override
		public DateTime getResult(ResultSet resultSet, int i) throws SQLException {
			return toDateTime(resultSet.getTimestamp(i));
		}

		@Override
		public DateTime getResult(CallableStatement callableStatement, int i) throws SQLException {
			return toDateTime(callableStatement.getTimestamp(i));
		}

		private static DateTime toDateTime(Timestamp timestamp) {
			if (timestamp != null) {
				return new DateTime(timestamp.getTime(), DateTimeZone.UTC);
			} else {
				return null;
			}
		}
	}

正确使用方式，在mapper xml中需要指定DateTime类型参数对应的 typeHandler

	<insert id="addUser" parameterType="user">
			INSERT INTO users(username, password, ctime) VALUES(#{username}, #{password},
			#{ctime, typeHandler=DateTimeTypeHandler})
	</insert>

这里的DateTimeTypeHandler为使用完全限定名（无命名空间），原因是我已经在配置中加好了alias，方法请参考本人上一篇博客，即在构建**sqlSessionFactoryBean**时，通过setTypeAliases方法指定使用的类型