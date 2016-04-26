---
layout: post
title:  "Id Generator实现方案"
date:   2016-4-26 17:40:55 +0800
categories: java id-generator mybatis
---

## Id生成器需求

- 整数id作为数据唯一标识
- 相同类型数据支持未来水平分表，id必须全局唯一
- 不同类型的数据采用独立的数据生成器

## 一种极为优秀的主键生成策略

参考

> flickr开发团队在2010年撰文介绍了flickr使用的一种主键生成测策略
>
> [一种极为优秀的主键生成策略](http://blog.csdn.net/bluishglc/article/details/7710738)

暂不考虑参考文献中的多点部署，仅考虑单库

get到几点重要信息：

1. 为生成全局唯一id，创建新的Sequence表

    若需要多个全局id，则创建多个Sequeance表

2. 表中仅包含`stub`列，`char(1) NOT NULL`，表仅有一条记录

3. 使用REPLACE INFO修改`stub`数据，通过MYSQL的LAST_INSERT_ID()函数获取id

## 实现

假设我们需要存储Account、AccountSet两种数据

这两种数据都支持存放于不同的表中，需要为这两种数据分别维护全局id

### 建表sql

    CREATE TABLE `user_id_sequence` (
        `id` bigint(20) NOT NULL AUTO_INCREMENT,
        `stub` char(1) NOT NULL,
        PRIMARY KEY  (`id`),
        UNIQUE KEY `idx_stub` (`stub`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

    CREATE TABLE `code_id_sequence` (
        `id` bigint(20) NOT NULL AUTO_INCREMENT,
        `stub` char(1) NOT NULL,
        PRIMARY KEY  (`id`),
        UNIQUE KEY `idx_stub` (`stub`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

### IdGeneratorMapper接口

    public interface IdGeneratorMapper {
        int count(@Param("tableName") String tableName);
        void clear(@Param("tableName") String tableName);
        void generate(IdHolder idHolder);
    }

### IdGeneratorMapper.xml

    <?xml version="1.0" encoding="UTF-8" ?>
    <!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
    <mapper namespace="com.baidu.bce.finance.fundpool.biz.mapper.IdGeneratorMapper">
        <delete id="clear">
            DELETE FROM ${tableName}
        </delete>

        <select id="count" resultType="int">
            SELECT COUNT(*) FROM ${tableName}
        </select>

        <insert id="generate" useGeneratedKeys="true" keyProperty="id">
            REPLACE INTO ${tableName} (stub) VALUES ('a')
        </insert>
    </mapper>

不同数据的id升级，需要使用到不同的表，表结构相同，因此将表明参数化

### Id Generator实现类

提供两种数据id生成方法

    @Service
    public class IdGeneratorServiceImpl implements IdGeneratorService {

        @Autowired
        private IdGeneratorMapper idGeneratorMapper;

        private static final String TABLE_USER_ID_SEQUENCE = "user_id_sequence";
        private static final String TABLE_CODE_ID_SEQUENCE = "code_id_sequence";

        @Transactional
        @Override
        public long genUserId() {
            return genId(TABLE_USER_ID_SEQUENCE);
        }

        @Transactional
        @Override
        public long genCodeId() {
            return genId(TABLE_CODE_ID_SEQUENCE);
        }

        private long genId(final String tableName) {
            IdHolder idHolder = new IdHolder() {
                {
                    setTableName(tableName);
                }
            };
            idGeneratorMapper.generate(idHolder);
            return idHolder.getId();
        }
    }

`IdHolder`即用作表名称参数的传递，也用作获取id值

    @Data
    public class IdHolder {
        private long id;
        private String tableName;
    }

关于`@Data`注解，目的是自动生成`get`/`set`，请查阅 Lombok相关资料

### 关于单元测试

一贯使用内嵌数据库H2以进行完整单元测试覆盖

而H2不支持Mysql的REPLACE 语法

因此编写mock类

    @Service
    @Primary
    public class IdGeneratorServiceMock implements IdGeneratorService {
        private Map<String, Long> idMap = new HashMap<>();

        @Override
        public long genUserId() {
            return genId("UserId");
        }

        @Override
        public long genCodeId() {
            return genId("CodeId");
        }

        private long genId(final String name) {
            Long id = idMap.get(name);
            if (id == null) {
                id = 0L;
            }
            ++id;
            idMap.put(name, id);
            return id;
        }
    }