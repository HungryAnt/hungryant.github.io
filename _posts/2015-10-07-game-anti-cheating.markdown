---
layout: post
title:  "游戏开发-防加速器作弊"
date:   2015-10-07 02:00:00
categories: game
---

##加速器作弊

游戏客户端程序中通畅会涉及游戏人物的行走、攻击、使用持续性道具等，拿人物行走来说，通畅游戏人物会有个固定的行动速度，根据时间差来求得人物的移动位置，这种情况自然会使用到用户计算机本地时间

一些加速器软件利用了这点，通过一些技术手段篡改时间值，游戏获取的时间被加速，玩家可以飞快的在游戏地图中移动，时间可以被加速数倍或者更高，严重影响游戏平衡性

鉴于此种作弊手段，博主提供一种有效的检测手段

> 一些独立网游，发布初期版本时，就有玩家使用加速器，这种简单的作弊手段，必须要防范

##一种检测加速器作弊的可行手段

加速器检测，首先想到的就是时间差异检测，时间差异放在客户端做还是服务端做，怎样进行通讯

博主提供一个可行方案

客户端/服务端交互

1. 初始调用防作弊接口

	客户端初始时发送一次初始化请求给服务端，携带客户端时间戳

	服务端记录客户端时间戳与服务端时间戳，已客户唯一标识存入hash表

2. 定期调用防作弊接口

	客户端间隔一段时间（10s）发送一次反作弊验证请求给服务端，携带客户端时间戳

	服务端进行防作弊检测

	检测不通过，返回错误

	检测通过，记录客户端与服务端时间戳，更新hash表数据

3. 防作弊检测

	计算两次时间戳间隔

	验证客户端时间戳间隔超出服务端时间戳间隔高于某阈值，则视为加速作弊，返回检测不通过

## Java服务端实现

AntiCheatingController 内定义如下API接口

	// 初始调用防作弊接口
	@RequestMapping(value = "/initClientTimestamp", method = RequestMethod.PUT)
	public void initClientTimestamp(@RequestParam(value = "userId") String userId,
	                                @RequestParam(value = "timestamp") long clientTimestampInS) {
	    antiCheatingService.initClientTimestamp(userId, clientTimestampInS);
	}

	// 定期调用防作弊接口 执行防作弊验证
	@RequestMapping(value = "/checkCheating", method = RequestMethod.PUT)
	public boolean checkIsCheating(@RequestParam(value = "userId") String userId,
	                               @RequestParam(value = "timestamp") long clientTimestampInS) {
	    return antiCheatingService.checkCheating(userId, clientTimestampInS);
	}


AntiCheatingService 封装核心逻辑，实现如下

	@Service
	public class AntiCheatingService {
	    private Map<String, TimestampPair> userClientTimestampMap = new HashMap<>();

	    public void initClientTimestamp(String userId, long clientTimestampInS) {
	        updateTimestamp(userId, clientTimestampInS, getServerTimestampInS());
	    }

	    public boolean checkCheating(String userId, long clientTimestampInS) {
	        TimestampPair timestampPair = userClientTimestampMap.get(userId);
	        if (timestampPair == null) {
	            initClientTimestamp(userId, clientTimestampInS);
	            return false;
	        }

	        long serverTimestampInS = getServerTimestampInS();

	        long clientDiff = clientTimestampInS - timestampPair.getClientTimestampInS();
	        if (clientDiff == 0) {
	            return false;
	        }

	        if (clientDiff < 0) {
	            throw new RuntimeException("wrong clientDiff, current: " + clientTimestampInS
	                    + "last: " + timestampPair.getClientTimestampInS());
	        }

	        long serverDiff = getServerTimestampInS() - timestampPair.getServerTimestampIns();
	        if (serverDiff < 10) { // 小于10秒，检测意义不大，忽略
	            return false;
	        }

	        if (clientDiff <= serverDiff) {
	            return false;
	        }

	        BigDecimal speedUpRate = DecimalUtility.toDecimal(
	                (clientDiff - serverDiff) * 1.0 / serverDiff);

	        if (speedUpRate.compareTo(new BigDecimal("0.2")) >= 0) {
	            ... //  检测到作弊，加速超过20%，此时可以记录作弊用户信息
	            return true;
	        }

	        updateTimestamp(userId, clientTimestampInS, serverTimestampInS);
	        return false;
	    }

	    private void updateTimestamp(String userId, long clientTimestampInS, long serverTimestampInS) {
	        TimestampPair timestampPair = new TimestampPair();
	        timestampPair.setClientTimestampInS(clientTimestampInS);
	        timestampPair.setServerTimestampIns(serverTimestampInS);
	        userClientTimestampMap.put(userId, timestampPair);
	    }

	    private long getServerTimestampInS() {
	        return DateTime.now().getMillis() / 1000;
	    }
	}

## Ruby游戏客户端实现

程序在启动时，创建一个子线程执行反作弊验证

子线程执行逻辑如下

初始执行init_client_timestamp，对应调用`initClientTimestamp`API接口

每15s执行check_cheating，对应调用`check_cheating`API接口

	  def init_check_cheating_thread
	    Thread.new {
	      begin
	        sleep(1)
	        init_client_timestamp
	        loop {
	          begin
	            check_cheating
	          rescue Exception => e
	            puts "check_cheating raise exception:#{e.message}"
	            puts e.backtrace.inspect
	          end
	          sleep(15)
	        }
	      rescue Exception => e
	        puts "init_check_cheating_thread raise exception:#{e.message}"
	        puts e.backtrace.inspect
	      end
	    }
	  end

执行HTTP请求的方式参考另一篇博文[ruby实战-发送http请求](/ruby/2015/10/07/ruby-httpclient-in-action.html){:target="_blank"}

check_cheating方法中在获知用户作弊时，则出发相关回调，上层逻辑给予违规用户相关警告

在本人的游戏中，给出的警告信息如下图

![作弊警告截图](http://7xk402.com1.z0.glb.clouddn.com/yecai_cheating.png)

##补充

部分违规用户可能会采用截获http请求伪造返回报文以达到禁用此防御措施的目的，游戏开发者需对此方案进行升级，在此不再赘述