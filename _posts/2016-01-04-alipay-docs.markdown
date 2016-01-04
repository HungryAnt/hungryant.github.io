---
layout: post
title:  "支付宝资料整理"
date:   2016-01-04 11:00:00
categories: java pay alipay
---

## 资料下载

[alipay资料整合.rar](http://pan.baidu.com/s/1nu3ggzR)

## 接口说明

#### 即时到账

- 功能说明

    支付宝网页即时到账功能，可让用户在线向开发者的支付宝账号支付资金，交易资金即时到账，帮助开发者快速回笼资金。
    
    [文档说明](http://doc.open.alipay.com/doc2/detail?treeId=62&articleId=103566&docType=1)

- 接口说明
    
    - [请求参数说明](http://doc.open.alipay.com/doc2/detail?spm=0.0.0.0.zZ73CH&treeId=62&articleId=103740&docType=1)

        请求参数是商户在与支付宝进行数据交互时，提供给支付宝的请求数据，以便支付宝根据这些数据进一步处理。

    - [页面跳转同步参数说明](http://doc.open.alipay.com/doc2/detail?spm=0.0.0.0.sYLvnn&treeId=62&articleId=103742&docType=1)

        支付宝对商户的请求数据处理完成后，会将处理的结果数据通过系统程序控制客户端页面自动跳转的方式通知给商户网站。这些处理结果数据就是页面跳转同步通知参数。

    - [服务器异步通知参数说明](http://doc.open.alipay.com/doc2/detail?spm=0.0.0.0.X1KA9v&treeId=62&articleId=103743&docType=1)
    
        支付宝对商户的请求数据处理完成后，会将处理的结果数据通过服务器主动通知的方式通知给商户网站。这些处理结果数据就是服务器异步通知参数。

    - [验证是否是支付宝发来的通知](http://doc.open.alipay.com/doc2/detail?treeId=58&articleId=103597&docType=1)

        该处的验证是指，对支付宝通知回来的参数notify_id合法性验证。

    - [业务出错时通知参数说明](http://doc.open.alipay.com/doc2/detail?spm=0.0.0.0.WhkABM&treeId=62&articleId=103744&docType=1)

        当商户提交请求给支付宝，支付宝在处理过程中发生业务异常时，支付宝会通过服务器主动通知的方式把出错的信息通知给商户网站。

- 目录`alipaydirect` （包含详细接口说明及Demo）


#### 即时到账有密退款

- 功能说明

    对通过即时到账等接口付款完成的交易进行部分或全部的退还。商户需输入支付密码。

    [文档说明](http://doc.open.alipay.com/doc2/detail?treeId=66&articleId=103571&docType=1)

- 退款流程

    退款流程1

    1. 获取要退款的 支付宝交易号
    2. 按要求 拼接成 退款参数
    3. 请求退款接口
    4. 跳转到我的收银台，输入 支付密码
    5. 确认 退款
    6. 支付宝返回 退款 操作结果 给 notify回调地址
    7. 在回调中处理 退款 结果。
    
    退款流程2

    1.登录收款支付宝账号
    2.找到该笔交易
    3.在交易详情中操作退款

    注意：退款接口是pc端接口使用的，目前没有手机端可操作退款接口，另外，退款接口目前需要输入支付密码

- 接口说明

    - [发起退款参数说明](http://doc.open.alipay.com/doc2/detail?spm=0.0.0.0.7HHXEE&treeId=66&articleId=103600&docType=1)
    
        请求参数是商户在与支付宝进行数据交互时，提供给支付宝的请求数据，以便支付宝根据这些数据进一步处理。

    - [服务器异步通知参数说明](http://doc.open.alipay.com/doc2/detail?spm=0.0.0.0.mu3kw1&treeId=66&articleId=103601&docType=1)

        支付宝对商户的请求数据处理完成后，会将处理的结果数据通过服务器主动通知的方式通知给商户网站。这些处理结果数据就是服务器异步通知参数。

    - 验证是否是支付宝发来的通知（同上）
    

- 目录`alipayrefund`（包含Demo）


#### 单笔交易查询

- 功能说明

    通过单笔交易查询接口，商户可以根据支付宝交易订单号或商户网站唯一订单号查询交易详细信息。

    详细说明见目录中的PDF文档

- 接口说明

    - 查询交易 `service=single_trade_query`

        trade_no参数(支付宝交易id)或者out_trade_no(商户维护的交易id)二选一
    
        GET请求，返回数据为XML格式，需解析且完成签名验证

- 目录`alipaysinglequery`（包含文档及Demo）


#### 财务明细查询

- 功能说明

    账务明细查询接口的功能为：通过HTTP协议使用该接口，可以查询到某个支付宝账号的账务明细。

    详细说明见PDF文档

- 接口说明

    - 查询财务明细 `service=export_trade_account_report`

        主要筛选条件：开始时间、结束时间 

        单条财务明细数据包含（元素csv_data）包含如下数据，以“,”分隔

            外部订单号
            账户余额（元）
            时间
            流水号
            支付宝交易号
            交易对方Email
            交易对方
            用户编号
            收入（元）
            支出（元）
            交易场所
            商品名称
            类型
            说明

- 目录`alipay_account`（包含文档及Demo）

#### 账务明细分页查询

- 功能说明
    
    查询某段时间指定的支付宝账号的账务明细。该接口能够根据页号来查询某一页的账务明细。

- 接口说明

    - 查询财务明细分页请求 `service=account.page.query`

        单条财务明细数据包含如下字段

            balance 余额
            income 收入金额
            outcome 支出金额
            trans_date交易付款时间
            sub_trans_code_msg 子业务类型
            trans_code_msg 业务类型
            merchant_out_order_no 商户订单号
            trans_out_order_no 订单号？
            bank_name 银行名称
            bank_account_no 银行账号
            bank_account_name 银行账户名字
            memo 备注
            buyer_account 买家支付宝人民币资金账号
            seller_account 卖家支付宝人民币资金账号
            seller_fullname 卖家姓名
            currency 货币代码
            deposit_bank_no 充值网银流水号
            goods_title 商品名称
            iw_account_log_id 账务序列号
            trans_account 账务本方支付宝人民币资金账号
            other_account_email 账务对方邮箱
            other_account_fullname 账务对方全称
            other_user_id 账务对方支付宝用户号
            partner_id 合作者身份ID
            service_fee 交易服务费
            service_fee_ratio 交易服务费率
            total_fee 交易总金额
            trade_no 支付宝交易号
            trade_refund_amount 累积退款金额
            sign_product_name 签约产品
            rate 费率

- 目录`alipay_account_page`（包含文档及Demo）