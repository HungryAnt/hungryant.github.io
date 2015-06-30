---
layout: post
title:  "Spring JdbcTemplate 使用总结与经验分享"
date:   2015-01-15 22:54:00
categories: java
---

## 引言

近期开发的几个项目，均是基于Spring boot框架的web后端项目，使用JdbcTemplate执行数据库操作，实际开发过程中，掌握了一些有效的开发经验，踩过一些坑，在此做个记录及总结，与各位读者分享。
欢迎留言与我交流。
正确使用JdbcTemplate执行数据库操作

## Bean声明

新增类型DatabaseConfiguration，添加注解@Configuration
该类型用于DataSource及JdbcTempate Bean的声明
 
基础代码如下

	@Configuration
	class DatabaseConfiguration {
		@Bean
		public DataSource dataSource() {
			DataSource dataSource;
			...
			return dataSource;
		}
	 
		@Bean
		public JdbcTemplate jdbcTemplate() {
			return new JdbcTemplate(dataSource());
		}
	}

注意这里将DataSource定义为Bean，Spring boot默认创建的TransactionManager对象依赖DataSource，若未将DataSource声明为Bean，则无法使用数据库事务
 
## 封装Dao类型

对于每一个数据库表，构建独立的Dao类型，提供供业务层调用的接口，注入JdbcTemplate对象，以实际操作db
可以定义基类如下

	/**
	 * Created by Ant on 2015/1/1.
	 */
	public abstract class AntSoftDaoBase {
		@Resource(name = "jdbcTemplate")
		private JdbcTemplate jdbcTemplate;
	 
		private String tableName;
	 
		protected AntSoftDaoBase(String tableName) {
			this.tableName = tableName;
		}
	 
		protected JdbcTemplate getJdbcTemplate() {
			return jdbcTemplate;
		}
	 
		public void clearAll() {
			getJdbcTemplate().update("DELETE FROM " + tableName);
		}
	 
		public int count() {
			return getJdbcTemplate().queryForObject( "SELECT count(*) FROM " + tableName, Integer.class);
		}
	}
通过@Resource注入jdbcTemplate对象，由于我仅定义了一个类型为jdbcTemplate的bean，可以这里可以省略掉name参数，及@Resource即可，或者使用@Autowired

如对于数据库中的table app

![table app](/image/posts/2015-02-12-java-spring-jdbctemplate-01.jpg)

建立对应的Dao派生类

	/**
	 * Created by Ant on 2015/1/1.
	 */
	@Repository
	public class AppDao extends AntSoftDaoBase{
		private Logger logger = LoggerFactory.getLogger(getClass());
	
		private static final String TABLE_NAME = "app";
	
		private static final String COLUMN_NAMES = "name, user_id, title, description, ctime, status";
	
		public AppDao() {
			super(TABLE_NAME);
		}
	
		public int create(final AppInfo appInfo) {
		   ...
		}
	
		public List<AppInfo> list(int pageNo, int pageSize) {
			...
		}
	
		public AppInfo get(int appId) {
		   ...
		}
	
		public void update(AppInfo appInfo) {
			...
		}
	}

该Dao类型提供了对AppInfo数据的增删查改接口，对这些接口的具体实现，后面再进行详细介绍

## 使用Tomcat-jdbc数据库连接池
	
引入数据库连接池，将大幅度提升数据库操作性能
本例描述Tomcat-jdbc数据库连接池使用方式
Pom文件中引入tomcat-jdbc依赖项

	<dependency>
		<groupId>org.apache.tomcat</groupId>
		<artifactId>tomcat-jdbc</artifactId>
		<version>7.0.42</version>
	</dependency>

创建连接池DataSource的逻辑封装在如下方法中，DatabaseConfiguration.dataSource方法内部可以直接调用此方法获取具备连接池功能的DataSource

	private DataSource getTomcatPoolingDataSource(String databaseUrl, String userName, String password) {
		org.apache.tomcat.jdbc.pool.DataSource dataSource = new org.apache.tomcat.jdbc.pool.DataSource();
		dataSource.setDriverClassName("com.mysql.jdbc.Driver");
		dataSource.setUrl(databaseUrl);
		dataSource.setUsername(userName);
		dataSource.setPassword(password);

		dataSource.setInitialSize(5); // 连接池启动时创建的初始化连接数量（默认值为0）
		dataSource.setMaxActive(20); // 连接池中可同时连接的最大的连接数
		dataSource.setMaxIdle(12); // 连接池中最大的空闲的连接数，超过的空闲连接将被释放，如果设置为负数表示不限
		dataSource.setMinIdle(0); // 连接池中最小的空闲的连接数，低于这个数量会被创建新的连接
		dataSource.setMaxWait(60000); // 最大等待时间，当没有可用连接时，连接池等待连接释放的最大时间，超过该时间限制会抛出异常，如果设置-1表示无限等待
		dataSource.setRemoveAbandonedTimeout(180); // 超过时间限制，回收没有用(废弃)的连接
		dataSource.setRemoveAbandoned(true); // 超过removeAbandonedTimeout时间后，是否进 行没用连接（废弃）的回收
		dataSource.setTestOnBorrow(true);
		dataSource.setTestOnReturn(true);
		dataSource.setTestWhileIdle(true);
		dataSource.setValidationQuery("SELECT 1");
		dataSource.setTimeBetweenEvictionRunsMillis(1000 * 60 * 30); // 检查无效连接的时间间隔 设为30分钟
		return dataSource;
	}

关于各数值的配置请根据实际情况调整

配置重连逻辑，以在连接失效是进行自动重连。默认情况下mysql数据库将关闭掉超过8小时的连接，开发的第一个java后端项目，加入数据库连接池后的几天早晨，web平台前几次数据库操作总是失败，配置重连逻辑即可解决

 

## 使用HSQL进行数据库操作单元测试

**忠告：数据库操作需要有单元测试覆盖**

本人给出如下理由：

1. 对于使用JdbcTemplate，需要直接在代码中键入sql语句，如今编辑器似乎还做不到对于java代码中嵌入的sql语句做拼写提示，经验老道的高手，拼错sql也不罕见

2. 更新db表结构后，希望快速知道哪些代码需要更改，跑一便单测比人肉搜索来的要快。重构速度*10

3. 有单测保证后，几乎可以认为Dao层完全可靠。程序出错，仅需在业务层排查原因。Debug速度*10

4. 没有单测，则需要在集成测试时构建更多更全面的测试数据，实际向mysql中插入数据。数据构建及维护麻烦、测试周期长

### 内嵌数据库HSQLDB

HSQLDB是一个开放源代码的JAVA数据库，其具有标准的SQL语法和JAVA接口

1. 配置HSQL DataSource

	引入HSQLDB依赖项
	
		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<version>2.3.0</version>
		</dependency>
	
	生成DataSource的方法可以如下方式实现
	
		@Bean
		public DataSource antsoftDataSource() {
			DataSource dataSource;
			if (antsoftDatabaseIsEmbedded) {
				dataSource = getEmbeddedHsqlDataSource();
			} else {
				dataSource =
						getTomcatPoolingDataSource(antsoftDatabaseUrl, antsoftDatabaseUsername, antsoftDatabasePassword);
			}
			return dataSource;
		}
	
	其中antsoftDatabaseIsEmbedded等对象字段值的定义如下
	
			@Value("${antsoft.database.isEmbedded}")
			private boolean antsoftDatabaseIsEmbedded;
		
			@Value("${antsoft.database.url}")
			private String antsoftDatabaseUrl;
		
			@Value("${antsoft.database.username}")
			private String antsoftDatabaseUsername;
		
			@Value("${antsoft.database.password}")
			private String antsoftDatabasePassword;
	
	通过@Value指定配置项key名称，运行时通过key查找配置值替换相应字段
	
	配置文件为`resources/application.properties`
	
		antsoft.database.isEmbedded=false
		antsoft.database.url=jdbc:mysql://127.0.0.1:3306/antsoft_app
		antsoft.database.username=root
		antsoft.database.password=ant
	
	单元测试配置文件为`resources/application-test.properties`
	
		antsoft.database.isEmbedded=true
	
	表示单测使用内嵌数据库

2. HSQL数据库初始化脚本

	创建Hsql DataSource时，同时执行数据库初始化操作，构建所需的表结构，插入初始数据
	
	getEmbeddedHsqlDataSource方法实现如下
	
		private DataSource getEmbeddedHsqlDataSource() {
			log.debug("create embeddedDatabase HSQL");
			return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.HSQL).addScript("classpath:db/hsql_init.sql").build();
		}
	
	通过addScript指定初始化数据库SQL脚本resources/db/hsql_init.sql，内容如下
	
		SET DATABASE SQL SYNTAX MYS TRUE;
		
		CREATE TABLE app (
		  id int GENERATED BY DEFAULT AS IDENTITY (START WITH 1, INCREMENT BY 1) NOT NULL,
		  name varchar(64) NOT NULL,
		  user_id varchar(64) NOT NULL,
		  title varchar(64) NOT NULL,
		  description varchar(1024) NOT NULL,
		  ctime datetime NOT NULL,
		  status int NOT NULL,
		  PRIMARY KEY (id),
		  UNIQUE (name)
		);
		
		CREATE TABLE app_unique_name (
		  id int GENERATED BY DEFAULT AS IDENTITY (START WITH 1, INCREMENT BY 1) NOT NULL,
		  unique_name varchar(64) NOT NULL UNIQUE,
		  PRIMARY KEY (id)
		);
		
		...

	HSQL语法与MySql语法存在差异，使用是需注意，我在开发过程中注意到的不同点列举如下
	
	- 不支持tinyint等数据类型，int后不允许附带表示数据长度的括号，如不支持int(11)
	- 不支持index索引，但支持unique index
	- 不支持AUTO_INCREMENT语法
　

3. 验证你的HSQL脚本

	可采用如下方式验证hsql语句正确性
	
	在本地maven仓库中找到hsqldb（正确引入过hsqldb），博主本机目录 C:\Users\ant\.m2\repository\org\hsqldb\hsqldb\2.3.2
	
	执行hsqldb-2.3.2.jar  (java -jar hsqldb-2.3.2.jar)
	
	默认窗体一个提示框，点击ok。在右侧输入SQL语句，执行工具栏中中Execuete SQL
	
	如下截图，显示SQL执行成功

	![HSQL Database Manager截图](/image/posts/2015-02-12-java-spring-jdbctemplate-02.jpg)

	上图SQL语句如下

		CREATE TABLE app_message (
		  id bigint GENERATED BY DEFAULT AS IDENTITY (START WITH 1, INCREMENT BY 1) NOT NULL,
		  app_id int NOT NULL,
		  message varchar(1024) NOT NULL,
		  ctime datetime NOT NULL,
		  status int NOT NULL,
		  PRIMARY KEY (id)
		);

	该SQL语句中使用`GENERATED BY DEFAULT AS IDENTITY (START WITH 1, INCREMENT BY 1)`  替代 `AUTO_INCREMENT`，似乎是HSQL不支持该语法，读者亲自尝试一下

	> AUTO_INCREMENT替代方案来源:
	[create-table-syntax-not-working-in-hsql](http://stackoverflow.com/questions/13206473/create-table-syntax-not-working-in-hsql)
 
### 编写单元测试覆盖Dao数据库操作

使用JUnit及Spring-test。单测可以直接注入所需的Bean
统一定义单元测试注解

	@Retention(RetentionPolicy.RUNTIME)
	@Target(ElementType.TYPE)
	@SpringApplicationConfiguration(classes = Application.class)
	@WebAppConfiguration
	@IntegrationTest("server.port=0")
	@ActiveProfiles("test")
	public @interface AntSoftIntegrationTest {
	}

定义测试类型，添加如下注解

	@AntSoftIntegrationTest
	@RunWith(SpringJUnit4ClassRunner.class)

我对自己代码的期望是，尽可能100%的Dao方法都被单元测试覆盖。

以下代码演示对AppService（其接口实现转发调用AppDao相应接口）进行的基本单元测试，其中测试了create、update及get三种操作

	@AntSoftIntegrationTest
	@RunWith(SpringJUnit4ClassRunner.class)
	public class AppServiceTests {
		@Autowired
		private AppService appService;
	
		@Autowired
		private TestService testService;
	
		@Before
		public void clearApp() {
			testService.clearApp();
		}
	
		@Test
		public void testApp() {
			final String name = "xxx";
			final String userId = "Ant";
			final String title = "Hello World";
			final String description = "Description for Hello World";
	
			final String updatedName = "xxx";
			final String updatedUserId = "Ant";
			final String updatedTitle = "Hello World";
			final String updatedDescription = "Description for Hello World";
	
			int appId;
			{
				// 创建应用
				AppInfo appInfo = new AppInfo();
				appInfo.setName(name);
				appInfo.setUserId(userId);
				appInfo.setTitle(title);
				appInfo.setDescription(description);
				appId = appService.createApp(appInfo);
			}
	
			CheckAppInfo(appId, name, userId, title, description, AppStatus.NORMAL);
	
			{
				// 更新应用
				AppInfo appInfo = new AppInfo();
				appInfo.setId(appId);
				appInfo.setName(updatedName);
				appInfo.setUserId(updatedUserId);
				appInfo.setTitle(updatedTitle);
				appInfo.setDescription(updatedDescription);
				appService.updateApp(appInfo);
			}
	
			CheckAppInfo(appId, updatedName, updatedUserId, updatedTitle, updatedDescription, AppStatus.NORMAL);
		}
	
		// 获取应用，并验证数据
		private void CheckAppInfo(int appId, String name, String userId, String title, String description,
								  AppStatus appStatus) {
			AppInfo appInfo = appService.getApp(appId);
			assertEquals(appId, appInfo.getId());
			assertEquals(name, appInfo.getName());
			assertEquals(userId, appInfo.getUserId());
			assertEquals(title, appInfo.getTitle());
			assertEquals(description, appInfo.getDescription());
			assertEquals(appStatus, appInfo.getStatus());
		}
	}
  

## 开发经验分享

本节记录笔者在实际项目开发过程中遇到的问题及解决方法、以及一些良好的开发实践

### 失效的事务

在使用Spring提供的事务处理机制时，事务的start与commit rollback操作由TransactionManager对象维护，开发中，我们只需在需要进行事务处理的方法上添加@Transactional注解，即可轻松开启事务
 
见Spring boot源码

	spring-boot/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/jdbc/DataSourceTransactionManagerAutoConfiguration.java
	/**
	 * {@link EnableAutoConfiguration Auto-configuration} for
	 * {@link DataSourceTransactionManager}.
	 *
	 * @author Dave Syer
	 */
	@Configuration
	@ConditionalOnClass({ JdbcTemplate.class, PlatformTransactionManager.class })
	public class DataSourceTransactionManagerAutoConfiguration implements Ordered {
	
		@Override
		public int getOrder() {
			return Integer.MAX_VALUE;
		}
	
		@Autowired(required = false)
		private DataSource dataSource;
	
		@Bean
		@ConditionalOnMissingBean(name = "transactionManager")
		@ConditionalOnBean(DataSource.class)
		public PlatformTransactionManager transactionManager() {
			return new DataSourceTransactionManager(this.dataSource);
		}
	
		@ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)
		@Configuration
		@EnableTransactionManagement
		protected static class TransactionManagementConfiguration {
	
		}
	
	}

由此可见，若未将DataSource声明为Bean，将不会创建transactionManager，@Transactional注解将毫无作用

 
当然，也有另一种方法，及不使用默认的transactionManager，而是自行定义，如下，在DatabaseConfiguration类中增加如下方法

	@Bean
	public PlatformTransactionManager transactionManager() {
		return new DataSourceTransactionManager(myDataSource());
	}

默认会使用方法名作为bean的命名，因此此处覆盖了默认的transactionManager Bean对象

 
如何多数据源启用事务（此处描述的非分布式事务）?

如果项目中涉及操作多个数据库，则存在多个数据源DataSource。解决方案同上例，即自行声明transactionManager Bean，与每个DataSource一一对应。需要注意的是，在使用@Transactional注解是，需要添加transactionManager Bean的名称，如@Transactional("myTransactionManager")
 
### 获取新增数据的自增id

如下Dao类型，方法create演示了如何创建一条MessageInfo记录，同时，获取该新增数据的主键，即自增id
 
	@Repository
	public class MessageDao extends AntSoftDaoBase {
		private static final String TABLE_NAME = "app_message";
	
		private static final String COLUMN_NAMES = "app_id, message, ctime, status";
	
		protected MessageDao() {
			super(TABLE_NAME);
		}
	
		private static final String SQL_INSERT_DATA =
				"INSERT INTO " + TABLE_NAME + " (" + COLUMN_NAMES + ") "
						+ "VALUES (?, ?, ?, ?)";
		
		public int create(final MessageInfo messageInfo) {
			KeyHolder keyHolder = new GeneratedKeyHolder();
			getJdbcTemplate().update(new PreparedStatementCreator() {
										 public PreparedStatement createPreparedStatement(Connection connection) throws
												 SQLException {
											 PreparedStatement ps =
													 connection.prepareStatement(SQL_INSERT_DATA, Statement.RETURN_GENERATED_KEYS);
											 int i = 0;
											 ps.setInt(++i, messageInfo.getAppId());
											 ps.setString(++i, messageInfo.getMessage());
											 ps.setTimestamp(++i, new Timestamp(new Date().getTime()));
											 ps.setInt(++i, 0); // 状态默认为0
											 return ps;
										 }
									 }, keyHolder
			);
			return keyHolder.getKey().intValue();
		}
		...
	}

### SQL IN 语句

IN语句中的数据项由逗号分隔，数量不固定，"?"仅支持单参数的替换，因此无法使用。此时只能拼接SQL字符串，如更新一批数据的status值，简单有效的实现方式如下

	private static final String SQL_UPDATE_STATUS =
			"UPDATE " + TABLE_NAME + " SET "
					+ "status = ? "
					+ "WHERE id IN (%s)";
	
	public void updateStatus(List<Integer> ids, Status status) {
		if (ids == null || ids.size() == 0) {
			throw new IllegalArgumentException("ids is empty");
		}
		String idsText = StringUtils.join(ids, ", ");
		String sql = String.format(SQL_UPDATE_STATUS , idsText);
		getJdbcTemplate().update(sql, status.toValue());
	} 
 

### 查询数据一般方法，及注意事项

AppDao类型中提供get方法，以根据一个appId获取该APP数据，代码如下

	private static final String SQL_SELECT_DATA =
			"SELECT id, " + COLUMN_NAMES + " FROM " + TABLE_NAME + " WHERE id = ?";
	
	public AppInfo get(int appId) {
		List<AppInfo> appInfoList = getJdbcTemplate().query(SQL_SELECT_DATA, new Object[] {appId}, new AppRowMapper());
		return appInfoList.size() > 0 ? appInfoList.get(0) : null;
	}
 

注意点：由于主键id会唯一标识一个数据项，有些人会使用queryForObject获取数据项，若未找到目标数据时，该方法并非返回null，而是抛异常EmptyResultDataAccessException。应使用query方法，并检测返回值数据量

AppRowMapper用于解析每行数据并转成Model类型，其代码如下

	private static class AppRowMapper implements RowMapper<AppInfo> {
		@Override
		public AppInfo mapRow(ResultSet rs, int i) throws SQLException {
			AppInfo appInfo = new AppInfo();
			appInfo.setId(rs.getInt("id"));
			appInfo.setName(rs.getString("name"));
			appInfo.setUserId(rs.getString("user_id"));
			appInfo.setTitle(rs.getString("title"));
			appInfo.setDescription(rs.getString("description"));
			appInfo.setCtime(rs.getTimestamp("ctime"));
			appInfo.setStatus(AppStatus.fromValue(rs.getInt("status")));
			return appInfo;
		}
	}
