---
layout: post
title:  "Mybatis特殊值Enum类型转换器-ValuedEnumTypeHanlder"
date:   2015-01-01 00:00:00
categories: Mybatis
---

## 引言

**typeHandlers**

MyBatis 在预处理语句（PreparedStatement）中设置一个参数时，Java对象将转换成数据库需要的数据格式，通过`ps.setInt`、`ps.setString`、`ps.setTimeStamp`等方法完成

在从结果集（ResultSet）中取出一个值时，则需要将数据库中获取到的数据转换为Java对象，其中会使用到`rs.getInt`、`rs.getString`、`rs.getTimeStamp`等方法

这两类操作通过 **类型处理器** 完成

- 类型处理器均实现`TypeHandler<T>`接口，所有基本类型均有对应的类型处理器

	[阅读官方文档 typeHandlers 一节](http://mybatis.github.io/mybatis-3/zh/configuration.html){:target="_blank"}

- Enum对应有两个类型处理器，分别为`EnumTypeHandler`、`EnumOrdinalTypeHandler`
	
	若需要将Enum类型的字段映射为数据库中的字符串，则使用`EnumTypeHandler`。 

	> 默认情况下，MyBatis 会利用 EnumTypeHandler 来把 Enum 值转换成对应的名字。

	若需要将Enum字段映射为int数值，则使用`EnumOrdinalTypeHandler`


但是`EnumOrdinalTypeHandler`局限性非常明显，其映射的数据直接使用枚举值的`ordinal`数值，与枚举值定义顺序紧耦合，本文将解决此问题


## 一般合理实现方案（针对每一种特定值Enum定义专属TypeHandler）

假设存在一个产品类型枚举类型定义，其产品均对应有特定int值，实现如下

	public enum ProductType {
	    AAA(100),
	    BBB(200),
	    CCC(300);

	    private int value;

	    private ProductType(int value) {
	        this.value = value;
	    }

	    public static ProductType fromValue(int value) {
	        for (ProductType productType : ProductType.values()) {
	            if (productType.value == value) {
	                return productType;
	            }
	        }
	        throw new IllegalArgumentException("Cannot create evalue from value: " + value + "!");
	    }

	    public int toValue() {
	        return value;
	    }
	}

同时，需要定义对应的`ProductTypeHandler`

	public class ProductTypeHandler implements TypeHandler<ProductType> {
	    @Override
	    public void setParameter(PreparedStatement preparedStatement, int i, ProductType productType, JdbcType jdbcType)
	            throws SQLException {
	        preparedStatement.setInt(i, productType.toValue());
	    }

	    @Override
	    public ProductType getResult(ResultSet resultSet, String s) throws SQLException {
	        return ProductType.fromValue(resultSet.getInt(s));
	    }

	    @Override
	    public ProductType getResult(ResultSet resultSet, int i) throws SQLException {
	        return ProductType.fromValue(resultSet.getInt(i));
	    }

	    @Override
	    public ProductType getResult(CallableStatement callableStatement, int i) throws SQLException {
	        return ProductType.fromValue(callableStatement.getInt(i));
	    }
	}


## 最佳实践

同样以上方的`ProductType`举例，枚举类型的定义稍作修改

	public enum ProductType implements ValuedEnum {
	    AAA(100),
	    BBB(200),
	    CCC(300);

	    private int value;

	    private PayMethod(int value) {
	        this.value = value;
	    }

	    public int getValue() {
	        return value;
	    }
	}

可以看到，这里实现了接口`ValuedEnum`，同时去除了`fromValue`类方法，`ValuedEnum`接口内容如下，仅声明一个`getValue`方法

	public interface ValuedEnum {
	    int getValue();
	}

不再需要为`ProductType`专门定义类型转换器，使用通用转换器`ValuedEnumTypeHanlder`（使用方式同使用内置转换器`EnumOrdinalTypeHandler`），实现参考`EnumOrdinalTypeHandler`源码，实现如下

	/**
	 * Created by sunlin05 on 2015/7/6.
	 * @author sunlin
	 */
	public class ValuedEnumTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {
	    private Class<E> type;
	    private Map<Integer, E> map = new HashMap<>();

	    public ValuedEnumTypeHandler(Class<E> type) {
	        if (type == null) {
	            throw new IllegalArgumentException("Type argument cannot be null");
	        }
	        this.type = type;
	        E[] enums = type.getEnumConstants();
	        if (enums == null) {
	            throw new IllegalArgumentException(type.getSimpleName() + " does not represent an enum type.");
	        }

	        for (E e : enums) {
	            ValuedEnum valuedEnum = (ValuedEnum) e;
	            map.put(valuedEnum.getValue(), e);
	        }
	    }

	    @Override
	    public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
	        ValuedEnum valuedEnum = (ValuedEnum) parameter;
	        ps.setInt(i, valuedEnum.getValue());
	    }

	    @Override
	    public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
	        int i = rs.getInt(columnName);
	        if (rs.wasNull()) {
	            return null;
	        } else {
	            return getValuedEnum(i);
	        }
	    }

	    @Override
	    public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
	        int i = rs.getInt(columnIndex);
	        if (rs.wasNull()) {
	            return null;
	        } else {
	            return getValuedEnum(i);
	        }
	    }

	    @Override
	    public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
	        int i = cs.getInt(columnIndex);
	        if (cs.wasNull()) {
	            return null;
	        } else {
	            return getValuedEnum(i);
	        }
	    }

	    private E getValuedEnum(int value) {
	        try {
	            return map.get(value);
	        } catch (Exception ex) {
	            throw new IllegalArgumentException(
	                    "Cannot convert " + value + " to " + type.getSimpleName() + " by value.", ex);
	        }
	    }
	}

核心逻辑见构造器，加注释说明如下

	// 获取所有枚举值
	E[] enums = type.getEnumConstants();

    for (E e : enums) {
    	// 类型转换为ValuedEnum接口对象
        ValuedEnum valuedEnum = (ValuedEnum) e;

        // 通过getValue()方法获取枚举值对应的Value int值，通过map记录映射关系
        map.put(valuedEnum.getValue(), e);
    }


**声明**

此方案由博主在今天日常工作中创造，若有更优秀的实践，请指教