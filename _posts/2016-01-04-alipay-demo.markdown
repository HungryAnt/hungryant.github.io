---
layout: post
title:  "支付宝对接流程及Demo"
date:   2016-01-04 10:00:00
categories: java pay alipay
---

## 支付宝对接交互逻辑

支付宝所有请求均使用api`https://mapi.alipay.com/gateway.do?`，根据`service`参数路由到不同的处理逻辑

### 准备工作

注册支付宝企业账号

取的如下身份标识信息

- partner (合作者身份id)
- sellerEmail (商户email)
- security_key (商户安全码)

### 签名

所有请求均需要进行签名，签名方式通过`sign_type`参数指定，目前支持 DSA、RSA、MD5 三种签名方式，`sign`参数存储签名字符串

所有支付宝回调请求，也都会进行签名

使用MD5方式签名说明

对所有请求params参数排序拼接字符串，加上`security_key`，取MD5值赋值到`sign`参数

### 通用参数说明

- `_input_charset`

    所有接口调用，使用utf-8编码，将所有接口请求参数`_input_charset`设置为`utf-8`

- `partner` 固定值

- `seller_email` 固定值

### 下单

下单并非有财务后台服务直接发起，而是财务后台服务生成一个支付宝收银台页面url，由用户浏览器跳转至该url，完成下单

请求支付宝收银台页面后，支付宝会产生一笔即时到账交易，等待用户支付

[支付宝即时到账文档说明](http://doc.open.alipay.com/doc2/detail?treeId=62&articleId=103566&docType=1)

[请求参数说明](http://doc.open.alipay.com/doc2/detail?spm=0.0.0.0.zZ73CH&treeId=62&articleId=103740&docType=1)

GET请求

重点参数说明

    service: 固定为 create_direct_pay_by_user
    payment_type： 固定为 1, 表示 商品购买
    return_url: 客户端支付完成跳转url
    notify_url: 支付宝异步通知url
    out_trade_no： 重要！商户订单id
    subject： 订单名称
    total_fee： 订单总额
    qr_pay_mode： 支付模式，可选择设置为前置二维码支付，影响到支付界面展示
    

**如何跳转到支付宝支付页面**

1. 服务器生成支付宝支付url，设置各类订单参数、身份参数、签名参数，具体参数请参考demo

2. 服务器返回重定向命令到支付宝支付url

3. 客户浏览器跳转到支付宝支付页面


**内嵌前置扫码页面**

- 前置扫码页面 与 支付宝支付页面url 基本完全一致，仅通过`qr_pay_mode`参数区分

- 支付完成浏览器跳转页面url稍有不同，由于前置扫码页面通过iframe嵌入在总览支付页中，跳转后仍旧是在iframe页面中，通过js控制对外部页面url执行跳转（后续前端可执行AJax请求可对此进行优化）

### 支付宝支付完成通知

通知分两种方式，说明如下

**浏览器跳转通知**
    
用户在支付宝支付页面完成支付后，跳转到支付成功页面，随后浏览器跳转至商户支付完成页面，携带交易id、交易状态等信息

[页面跳转同步参数说明](http://doc.open.alipay.com/doc2/detail?spm=0.0.0.0.sYLvnn&treeId=62&articleId=103742&docType=1)

GET请求

重点参数说明

    out_trade_no： 商户订单id
    trade_no： 支付宝交易id
    trade_status： 支付宝交易状态，可取值 TRADE_SUCCESS/TRADE_FINISHED
    notify_id： 通知id

返回页面信息已告知用户支付已经完成


**异步回调通知**

交易状态变更后，支付宝异步通知商户，若失败，则在24小时内重试8次

[服务器异步通知参数说明](http://doc.open.alipay.com/doc2/detail?spm=0.0.0.0.X1KA9v&treeId=62&articleId=103743&docType=1)

POST请求，数据格式 form-encoded

重点参数说明
    
    out_trade_no： 商户订单id
    trade_no： 支付宝交易id
    trade_status： 支付宝交易状态，可取值 TRADE_SUCCESS/TRADE_FINISHED
    notify_id： 通知id

返回值须设置为`success`表示成功

**通知总结**

不保证两类通知到达顺序
 
安全验证

- 签名验证
- 消息验证，以参数`notifyId`，调用支付宝接口验证此通知正确性

**交易状态流转**

`trade_status`状态，仅如下两个状态，支付宝会回调通知商户

- TRADE_SUCCESS 交易成功
- TRADE_FINISHED 交易完成

用户在支付宝完成支付，交易状态变更为TRADE_SUCCESS

交易成功后，可对该笔交易执行退款操作，当交易完成后，则不允许执行退款

交易超时时间，默认为15天，交易成功后，且超过超时时间，则交易状态变为完成

### 退款

发起退款，调用支付宝有密退款接口（批量接口）

有密意味着财务人员操作退款，需要输入支付宝支付密码

**发起退款**

GET请求，浏览器跳转，财务后端完成url拼接

[发起退款参数说明](http://doc.open.alipay.com/doc2/detail?spm=0.0.0.0.7HHXEE&treeId=66&articleId=103600&docType=1)

重点参数说明

    service: refund_fastpay_by_platform_pwd
    notify_url: 退款通知url
    refund_date: 退款北京时间
    batch_no: 退款批次号,保证唯一性,格式遵循 201511020000001
    batch_num: 退款记录集数量
    detail_data: 退款记录集

**退款通知**

POST请求，数据格式 form-encoded

[服务器异步通知参数说明](http://doc.open.alipay.com/doc2/detail?spm=0.0.0.0.mu3kw1&treeId=66&articleId=103601&docType=1)

重点参数说明

    batch_no: 退款批次号
    success_num: 退款记录集数量
    result_details: 退款结果集明细

返回值须设置为`success`表示成功

### 订单查询（该API接口权限需单独申请）

避免极端情况未能成功收到支付宝的状态通知，应对尚未过期且未完成支付的订单向支付宝查询交易状态，完成数据同步

可依据支付宝交易id或者商户订单id查询订单交易数据

GET请求 财务后台服务发起请求

重点参数说明

    service: single_trade_query
    trade_no: 支付宝交易id
    out_trade_no： 商户订单id

返回数据 xml格式，详情见pdf文档

xml解析需注意对待，返回结果数据层次较深，应仅对结果数据绑定实体类

### 订单明细查询

GET请求

重点参数说明

    service: export_trade_account_report
    gmt_create_start: 账务明细的开始时间 格式为yyyy-MM-dd HH:mm:ss
    gmt_create_end: 账务明细的结束时间
    user_id: 用户的支付宝账号对应的支付宝唯一用户号 可空

开始时间与结束时间的间隔不能大于一天

### 订单分页明细查询

GET请求

    service: account.page.query

与订单明细查询类似，返回数据字段全，具体参数及返回值见pdf文档
