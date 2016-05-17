---
layout: post
title:  "Lombok典型示例"
date:   2016-5-17 21:15:45 +0800
categories: java lombok
---

## 传统Builder模式代码示例

这是个采用Builder模式的简单类型

    public class ConsumptionResult {
        private BigDecimal cash;
        private BigDecimal coupon;
        private BigDecimal rebate;

        public ConsumptionResult() {
        }

        private ConsumptionResult(Builder builder) {
            this.cash = builder.cash;
            this.coupon = builder.coupon;
            this.rebate = builder.rebate;
        }

        public static Builder newBuilder() {
            return new Builder();
        }

        public BigDecimal getCash() {
            return cash;
        }

        public void setCash(BigDecimal cash) {
            this.cash = cash;
        }

        public BigDecimal getCoupon() {
            return coupon;
        }

        public void setCoupon(BigDecimal coupon) {
            this.coupon = coupon;
        }

        public BigDecimal getRebate() {
            return rebate;
        }

        public void setRebate(BigDecimal rebate) {
            this.rebate = rebate;
        }


        public static final class Builder {
            private BigDecimal cash;
            private BigDecimal coupon;
            private BigDecimal rebate;

            private Builder() {
            }

            public ConsumptionResult build() {
                return new ConsumptionResult(this);
            }

            public Builder cash(BigDecimal cash) {
                this.cash = cash;
                return this;
            }

            public Builder coupon(BigDecimal coupon) {
                this.coupon = coupon;
                return this;
            }

            public Builder rebate(BigDecimal rebate) {
                this.rebate = rebate;
                return this;
            }
        }
    }


使用方式：

    public ConsumptionResult getConsumption(@RequestParam("userId") String userId) {
        BigDecimal cash = ...
        BigDecimal coupon = ...
        BigDecimal rebate = ...
        return ConsumptionResult.newBuilder().cash(cash).coupon(coupon).rebate(rebate).build();
    }

可以看出，使用方便，但类型定义过度冗余

## 使用Lombok简化后的类型

    @Data
    @Builder
    public class ConsumptionResult {
        private BigDecimal cash;
        private BigDecimal coupon;
        private BigDecimal rebate;
    }

显而易见，非常优秀的工具！


## Maven依赖

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.16.6</version>
        <scope>provided</scope>
    </dependency>

## Lombok官网地址

[https://projectlombok.org/](https://projectlombok.org/){:target="_blank"}