---
layout: post
title:  "java开发技术分享 第二期"
date:   2015-01-01 00:00:00
categories: java
---

## java技术分享 第二期

### 目的

- 一线coder，分享一些比较好的编码实践经验
- 促进交流，互相学习，避免重复调研、重复采坑。让新同学技术上得到成长


### 热身

[Hello World](http://stackoverflow.com/questions/15182496/why-does-this-code-using-random-strings-print-hello-world)


### 实战

#### 使用@Primary注解，指定注入优先级


#### Mybatis sql generator
    
    <select id="getSumCouponConsumption" resultType="bigdecimal">
        SELECT IFNULL(sum(amount), 0) as sum_consume
        FROM frozen_amount
        <where>
            <if test="startTime != null and endTime != null">
                mtime BETWEEN '${startTime}' AND '${endTime}'
            </if>
            AND status = 2
        </where>
    </select>

#### 使用H2数据库（替代hsql）

代码见 DatabaseUtility

    public static DataSource getEmbededH2DataSource(String ... scripts) {
        EmbeddedDatabaseBuilder builder = new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2)
				.addScript("classpath:db/h2_init.sql");
        for (String script : scripts) {
            builder.addScript(script);
        }
        return builder.build();
    }

#### BigDecimal类型使用

相比与double类型不会有精度丢失，无数据范围限制

使用示例


	@Test
    public void testBigDecimal() {
        BigDecimal a = new BigDecimal("99.99");
        BigDecimal b = new BigDecimal("0.01");
        BigDecimal c = new BigDecimal(100);

        assertTrue(a.compareTo(b) > 0);
        assertTrue(c.compareTo(a.add(b)) == 0);
    }

基本用法

	// 对于整数
	BigDecimal amount = new BigDecimal(100);

	// 对于小数
	BigDecimal amount = new BigDecimal("99.9");
	
	// 判断两个值是否相等
	a.compareTo(b) == 0;
    
	// 基本运算
	c = a.add(b);
	c = a.subtract(b);
	c = a.multiply(b);
	c = a.divide(b);

对应数据库操作

Mysql数据库支持类型decimal，对于数据库中需要存储小数的场合，应当都使用decimal



#### Guava几个实用的集合类型

- Multiset

    List<String> 包含若干单词，需统计不同单词出现次数
    
    原始方法
    
        Map<String, Integer> counts = new HashMap<String, Integer>();
        for (String word : words) {
            Integer count = counts.get(word);
            if (count == null) {
                counts.put(word, 1);
            } else {
                counts.put(word, count + 1);
            }
        }
    
    使用Multiset<E>
    
        Multiset<String> multiset = HashMultiset.create();
    
        for (String word : words) {
            multiset.add(word);
        }
        
        // 展示各单词出现次数
        for (Multiset.Entry<String> entry : multiset.entrySet()) {
            System.out.println(String.format("%s\t%d", entry.getElement(), entry.getCount()));
        }

- Multimap

- BiMap

- Table

- ClassToInstanceMap

- RangeSet

- RangeMap


---
待定

初识java nio

goole guava 集合类型

分页方法

