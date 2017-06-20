---
layout: post
title:  "基于spring boot项目的多数据源配置与分布式事务处理总结"
date:   2015-11-26 08:36:20
categories: java
---

## 多数据源配置

项目存在10个数据源，如下

- core\_biz 业务逻辑 数据库
- core\_sys 系统设置 数据库
- fund\_pool 资金池 数据库 分用户拆分了8个库

针对这10个数据源，分别进行创建

首先为业务逻辑数据库创建数据源，定义为Java Bean

`@Configuration`及`@Bean`注解的使用不做赘述

```java
@Configuration
public class CoreBizDataSourceConfiguration {
    @Value("${core.biz.database.isEmbedded}")
    private Boolean databaseIsEmbedded;

    @Value("${core.biz.database.url}")
    private String databaseUrl;

    @Value("${core.biz.database.username}")
    private String databaseUsername;

    @Value("${core.biz.database.password}")
    private String databasePassword;

    @Bean(name = "coreBizDataSource")
    public DataSource coreBizDataSource() {
        if (databaseIsEmbedded) {
            return DataSourceUtil.getEmbeddedH2XADataSource(
                    "core_biz", "classpath:db/core_biz_h2_init.sql");
        } else {
            return DataSourceUtil.getAtomikosXADataSource("core_biz",
                    databaseUrl, databaseUsername, databasePassword);
        }
    }
}
```

细节说明

- 配置信息定义在`application.properties`中
- `databaseIsEmbedded`标识是否使用内置数据库
- `DataSourceUtil.getEmbeddedH2XADataSource` 获取支持分布式事务的内嵌数据源，参数`core_biz`表示数据库名
- `DataSourceUtil.getAtomikosXADataSource` 获取支持分布式事务的MySql数据源

具体细节见后续小节

core\_sys数据库对应数据源与core\_biz类似

资金池数据源定义如下，共8个数据源

方案一：（如果你需要对该8个数据源共享使用事务，则方案一行不通）
通过实现`AbstractRoutingDataSource`抽象类，实现DataSource路由，对外仅暴露一个DataSource Bean实例

```java
@Component
public class FundPoolRoutingDataSource extends AbstractRoutingDataSource {
    @Value("${fundPool.database.isEmbedded}")
    private Boolean databaseIsEmbedded;

    @Value("${fundPool.database.url.format}")
    private String databaseUrlFormat;

    @Value("${fundPool.database.username}")
    private String databaseUsername;

    @Value("${fundPool.database.password}")
    private String databasePassword;

    @PostConstruct
    private void init() {
        Map<Object, Object> dataSourceMap = new HashMap<>();
        for (int i = 0; i < 8; i++) {
            dataSourceMap.put(i, createFundPoolDataSource(i));
        }
        setTargetDataSources(dataSourceMap);
    }

    @Override
    protected Object determineCurrentLookupKey() {
        return FundPoolContextHolder.getPoolIndex();
    }

    private DataSource createFundPoolDataSource(int index) {
        if (databaseIsEmbedded) {
            return DataSourceUtil.getEmbeddedH2XADataSource(
                    String.format("FN_CHOGORI_POOL_00%d", index),
                    "classpath:db/fund_pool_h2_init.sql");
        } else {
            return DataSourceUtil.getAtomikosXADataSource(String.format("fund_pool_%d", index),
                    String.format(databaseUrlFormat, index), databaseUsername, databasePassword);
        }
    }
}
```

再次注意，由于将多个数据源抽象为一个数据源，则无法对内部多个数据源共享事务处理

细节说明

- `init`方法创建8个数据源，构建`dataSourceMap`，调用`setTargetDataSources()`
- `FundPoolContextHolder`中存储当前使用的到数据源索引号
- 重写`determineCurrentLookupKey()`返回当前数据源索引

`FundPoolContextHolder` 实现如下

```java
public class FundPoolContextHolder {
    private static final ThreadLocal<Integer> CONTEXT_HOLDER = new ThreadLocal<>();

    public static void setPoolIndex(Integer poolIndex) {
        Assert.isTrue(poolIndex >= 0 && poolIndex < 8);
        CONTEXT_HOLDER.set(poolIndex);
    }

    public static Integer getPoolIndex() {
        return CONTEXT_HOLDER.get();
    }

    public static void clearPoolIndex() {
        CONTEXT_HOLDER.remove();
    }
}
```

参考 [Dynamic DataSource Routing](http://spring.io/blog/2007/01/23/dynamic-datasource-routing)

方案二（推荐）：（项目使用MyBatis实现数据访问，配置如下）

```java
@Configuration
public class FundPoolMybatisConfiguration {
    private Logger logger = LoggerFactory.getLogger(getClass());

    @Autowired
    private FundPoolDataSourceGenerator fundPoolDataSourceGenerator;

    @Bean
    public FundPoolMapperContainer fundPoolMapperContainer() throws Exception {
        FundPoolMapperContainer container = new FundPoolMapperContainer();
        for (int i = 0; i < FundPoolDefinition.FUND_POOL_COUNT; i++) {
            DataSource dataSource = fundPoolDataSourceGenerator.createFundPoolDataSource(i);
            SqlSessionFactory sqlSessionFactory = createSqlSessionFactory(dataSource);
            MapperFactoryBean<FundPoolMapper> mapperFactoryBean = getMapper(FundPoolMapper.class, sqlSessionFactory);
            mapperFactoryBean.afterPropertiesSet();
            container.put(i, mapperFactoryBean.getObject());
        }
        return container;
    }

    private SqlSessionFactory createSqlSessionFactory(DataSource dataSource) throws Exception {
        return createSqlSessionFactory(dataSource, Arrays.<Class>asList(FundPool.class));
    }

    private SqlSessionFactory createSqlSessionFactory(DataSource dataSource, List<Class> types) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        List<Class> allTypes = new ArrayList<>();
        allTypes.addAll(Arrays.asList(
                DateTimeTypeHandler.class,
                EnumTypeHandler.class, EnumOrdinalTypeHandler.class));
        allTypes.addAll(types);
        sqlSessionFactoryBean.setTypeAliases(allTypes.toArray(new Class[allTypes.size()]));
        sqlSessionFactoryBean.getObject().getConfiguration().setMapUnderscoreToCamelCase(true);
        return sqlSessionFactoryBean.getObject();
    }

    private <T> MapperFactoryBean<T> getMapper(Class<T> mapperInterface, SqlSessionFactory sessionFactory) {
        MapperFactoryBean<T> mapperFactoryBean = new MapperFactoryBean<>();
        try {
            mapperFactoryBean.setSqlSessionFactory(sessionFactory);
            mapperFactoryBean.setMapperInterface(mapperInterface);
        } catch (Exception ex) {
            logger.error("error when create mapper: ", ex);
            throw new RuntimeException(ex);
        }
        return mapperFactoryBean;
    }
}
```

说明

- `FundPoolDataSourceGenerator`用于创建基本DataSource

```java
@Component
public class FundPoolDataSourceGenerator {
    @Value("${fundPool.database.isEmbedded}")
    private Boolean databaseIsEmbedded;

    @Value("${fundPool.database.url.format}")
    private String databaseUrlFormat;

    @Value("${fundPool.database.username}")
    private String databaseUsername;

    @Value("${fundPool.database.password}")
    private String databasePassword;

    public DataSource createFundPoolDataSource(int index) {
        if (databaseIsEmbedded) {
            return DataSourceUtil.getEmbeddedH2XADataSource(
                    String.format("FN_CHOGORI_POOL_00%d", index),
                    "classpath:db/fund_pool_h2_init.sql");
        } else {
            return DataSourceUtil.getAtomikosXADataSource(String.format("fund_pool_%d", index),
                    String.format(databaseUrlFormat, index), databaseUsername, databasePassword);
        }
    }
}
```

- **FundPoolMapperContainer**保管8个库的Mapper实例，其实现如下
    
    ```java
    public class FundPoolMapperContainer {
        private Map<Integer, FundPoolMapper> fundPoolMapperMap = new Hashtable<>();
    
        public void put(int i, FundPoolMapper fundPoolMapper) {
            fundPoolMapperMap.put(i, fundPoolMapper);
        }
    
        public FundPoolMapper get(int i) {
            return fundPoolMapperMap.get(i);
        }
    }
    ```

    将`FundPoolMapperContainer`定义为Bean后，以如下方式使用

    ```java
    @Service
    public class FundPoolService {
        @Autowired
        private FundPoolMapperContainer fundPoolMapperContainer;
    
        private FundPoolMapper getFundPoolMapper(int poolIndex) {
            return fundPoolMapperContainer.get(poolIndex);
        }
    
        public int count(int poolIndex) {
            return getFundPoolMapper(poolIndex).count();
        }
    
        public static int getPoolIndex(long uid) {
            return (int) (uid % 8);
        }
    
        public FundPool get(long uid) {
            return getFundPoolMapper(getPoolIndex(uid)).get(uid);
        }

        // ...
    }
    ```

## JTA分布式事务

### 原理及实践教程

[Configuring Spring and JTA without full Java EE](
http://spring.io/blog/2011/08/15/configuring-spring-and-jta-without-full-java-ee/)

[JTA 深度历险 - 原理与实现](http://www.ibm.com/developerworks/cn/java/j-lo-jta/index.html)

### 开源实现Atomikos

[Why Use Atomikos](http://www.atomikos.com/Documentation/WhyUseAtomikos)

[Installing TransactionsEssentials](http://www.atomikos.com/Main/InstallingTransactionsEssentials)

### 依赖库添加

项目需添加如下依赖

```xml
<dependency>
    <groupId>com.atomikos</groupId>
    <artifactId>transactions-jdbc</artifactId>
    <version>3.9.3</version>
</dependency>

<dependency>
    <groupId>javax.transaction</groupId>
    <artifactId>jta</artifactId>
    <version>1.1</version>
</dependency>
```

### 创建JTA数据源

“多数据源配置”一节中对于mysql数据源的创建，使用工具方法`DataSourceUtil.getAtomikosXADataSource`

其实现如下

```java
public static DataSource getAtomikosXADataSource(
        String uniqueResourceName, String databaseUrl, String userName, String password) {
    MysqlXADataSource mysqlXADataSource = new MysqlXADataSource();
    mysqlXADataSource.setUrl(databaseUrl);
    mysqlXADataSource.setUser(userName);
    mysqlXADataSource.setPassword(password);

    AtomikosDataSourceBean atomikosDataSource = new AtomikosDataSourceBean();
    atomikosDataSource.setUniqueResourceName(uniqueResourceName);
    atomikosDataSource.setXaDataSource(mysqlXADataSource);
    atomikosDataSource.setMinPoolSize(5);
    atomikosDataSource.setMaxPoolSize(20);
    atomikosDataSource.setTestQuery("SELECT 1");
    return atomikosDataSource;
}
```

说明：
- 使用`MysqlXADataSource`创建支持XA协议的数据源
- `AtomikosDataSourceBean`实现连接池
    - 如果之前使用`org.apache.tomcat.jdbc.pool.DataSource`作为连接池，必须改为直接使用`MySqlXADataSource`（使用tomcat.jdbc.pool.XADataSource
- 不同数据源，`uniqueResourceName`需保证唯一

### 基于spring boot项目的JTA配置

```java
@Configuration
public class JtaTransactionConfiguration {
    @Autowired
    private AtomikosJtaConfiguration jtaConfiguration;

    @Bean(name = "financeCore")
    public PlatformTransactionManager platformTransactionManager()  throws Throwable {
        return new JtaTransactionManager(jtaConfiguration.userTransaction(), jtaConfiguration.transactionManager());
    }
}
```

项目中存在多个`PlatformTransactionManager`Bean实例，因此命名上加以区分，这里指定为`financeCore`

```java
@Configuration
public class AtomikosJtaConfiguration {
    @Bean
    public UserTransaction userTransaction() throws Throwable {
        UserTransactionImp userTransactionImp = new UserTransactionImp();
        userTransactionImp.setTransactionTimeout(1000);
        return userTransactionImp;
    }

    @Bean(initMethod = "init", destroyMethod = "close")
    public TransactionManager transactionManager() throws Throwable {
        UserTransactionManager userTransactionManager = new UserTransactionManager();
        userTransactionManager.setForceShutdown(false);
        return userTransactionManager;
    }
}
```

### 使用JTA事务

使用spring内置的`@Transactional`标识事务处理范围

```java
@Service
public class FundPoolService {
    @Transactional(value = "financeCore")
    public void increase(long uid, BigDecimal amount) {
        // ...
    }
}
```

### 单测，确保分布式事务生效

确保分布式事务正确有效执行，对多数据源数据操作实施单测验证

默认Embedded数据库不支持分布式事务，扩展方式见下一小节，也可本地搭建mysql服务进行验证

**单测**

实现纯用于单测的类型`JtaDemoService`，配套`JtaDemoServiceTest`实现如下：

```java
public class JtaDemoServiceTest extends FinanceCoreTestBase {
    @Autowired
    private JtaDemoService jtaDemoService;

    @Test
    public void testRunAllCommit() {
        jtaDemoService.runAllCommit();
        jtaDemoService.validateAllCommit();
    }

    @Test
    public void testRunAllRollback() {
        try {
            jtaDemoService.runAllRollback();
        } catch (RuntimeException ignore) {
            // ignore
        }
        jtaDemoService.validateAllRollback();
    }
}
```

**验证commit逻辑**

```java
@Service
public class JtaDemoService {

    // ...

    @Transactional(value = "financeCore")
    public void runAllCommit() {
        createTransferRecord(); // 操作BIZ数据库

        // 操作各fundPool数据库
        for (int i = 0; i < FundPoolDefinition.FUND_POOL_COUNT; i++) {
            createFundPool(i);
        }

        createFundPoolChangeRecord(); // 操作SYS数据库
    }

    public void validateAllCommit() {
        assertEquals(1, transferService.count());
        assertEquals(8, fundPoolService.count());
        // 操作各fundPool数据库
        for (int i = 0; i < FundPoolDefinition.FUND_POOL_COUNT; i++) {
            assertEquals(1, fundPoolService.count(i));
        }
        assertEquals(1, fundPoolChangeRecordService.count());
    }

    // ...
}
```

**验证callback逻辑**

```java
@Service
public class JtaDemoService {

    // ...

    @Transactional(value = "financeCore")
    public void runAllRollback() {
        createTransferRecord();
        for (int i = 0; i < FundPoolDefinition.FUND_POOL_COUNT; i++) {
            createFundPool(i);
        }
        createFundPoolChangeRecord();
        throw new RuntimeException();
    }

    public void validateAllRollback() {
        assertEquals(0, transferService.count());
        assertEquals(0, fundPoolService.count());
        assertEquals(0, fundPoolChangeRecordService.count());
    }

    // ...
}
```

## Spring Embedded数据库分布式事务支持

创建内置H2 Database常规方式如下：

```java
public static DataSource getEmbeddedH2DataSource(String name, String... scripts) {
    EmbeddedDatabaseBuilder builder = new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2)
            .setName(name)
            .setScriptEncoding("utf8").addScript("classpath:db/h2_init.sql");
    for (String script : scripts) {
        builder.addScript(script);
    }
    return builder.build();
}
```

EmbeddedDatabaseBuilder默认创建的DataSource并未实现XADataSource接口，因此不支持分布式事务

阅读EmbeddedDatabaseBuilder及相关代码，得出类结构（类结构图省略），要做的就是替换掉`SimpleDriverDataSource`

这里替换成新的类型`H2DriverDataSourceFactory`，该类实现如下：

```java
private static class H2DriverDataSourceFactory implements DataSourceFactory {
    private final JdbcDataSource dataSource = new JdbcDataSource();

    @Override
    public ConnectionProperties getConnectionProperties() {
        return new ConnectionProperties() {
            @Override
            public void setDriverClass(Class<? extends Driver> driverClass) {
                // dataSource.setDriverClass(driverClass);
            }

            @Override
            public void setUrl(String url) {
                dataSource.setUrl(url);
            }

            @Override
            public void setUsername(String username) {
                dataSource.setUser(username);
            }

            @Override
            public void setPassword(String password) {
                dataSource.setPassword(password);
            }
        };
    }

    @Override
    public DataSource getDataSource() {
        return this.dataSource;
    }
}
```

创建Embedded H2 Database 数据源时，使用如下工具方法

```java
public static DataSource getEmbeddedH2XADataSource(String name, String... scripts) {
    H2DriverDataSourceFactory h2DriverDataSourceFactory = new H2DriverDataSourceFactory();

    EmbeddedDatabaseBuilder builder = new EmbeddedDatabaseBuilder();
    builder.setDataSourceFactory(h2DriverDataSourceFactory);
    builder.setType(EmbeddedDatabaseType.H2).setName(name)
            .setScriptEncoding("utf8").addScript("classpath:db/h2_init.sql");
    for (String script : scripts) {
        builder.addScript(script);
    }
    builder.build();

    AtomikosDataSourceBean atomikosDataSource = new AtomikosDataSourceBean();
    atomikosDataSource.setUniqueResourceName(name);
    atomikosDataSource.setXaDataSource((XADataSource) h2DriverDataSourceFactory.getDataSource());
    return atomikosDataSource;
}
```