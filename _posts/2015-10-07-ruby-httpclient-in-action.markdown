---
layout: post
title:  "ruby实战-发送http请求"
date:   2015-10-07 01:00:00
categories: ruby
---

## Ruby类库Net::HTTP简述

An HTTP client API for Ruby.

使用Net::HTTP类库提供的丰富API，可以发送任意http请求

	require 'net/http'

官方案例

- GET

		Net::HTTP.get('example.com', '/index.html') # => String

- GET by URL

		uri = URI('http://example.com/index.html?count=10')
		Net::HTTP.get(uri) # => String

- POST

		uri = URI('http://www.example.com/search.cgi')
		res = Net::HTTP.post_form(uri, 'q' => 'ruby', 'max' => '50')
		puts res.body

详细内容请参见 [官方文档](http://ruby-doc.org/stdlib-2.0.0/libdoc/net/http/rdoc/Net/HTTP.html){:target="_blank"}

## Ruby实战-封装易于使用的HttpClient

实际项目中，客户端有多处逻辑需要执行http请求，传输json格式数据

对Net::HTTP进行适度的封装可以有效简化程序


###使用封装后的HttpClient

执行get请求，传递url参数userId

	  # 查询账号资金
	  def get_amount(user_id)
	    http_client = HttpClientFactory.create
	    http_client.path 'account/getAmount'
	    http_client.params(userId: user_id)
	    res = http_client.get

	    if res.code == '200'
	      return res.body.to_i
	    else
	      return 0
	    end
	  end

执行Post请求，传递url参数，解析返回的json数据

	  # 购买商品
	  def buy(user_id, key)
	    http_client = HttpClientFactory.create
	    http_client.path 'shopping/buy'
	    http_client.params userId: user_id, key: key
	    res = http_client.post
	    check_res res
	  end

HttpClientFactory.create代码如下，创建HttpClient实例，参数为后台服务地址

	class HttpClientFactory
	  def self.create
	    AntHttp::HttpClient.new(NetworkConfig::WEB_SERVICE_ENDPOINT)
	  end
	end

###封装HttpClent

1. 构造器接受服务地址及content_type参数(默认为json格式)
2. path方法接受uri地址拼接
3. params方法接受API请求参数
4. 支持get/post/put/delete方法，对应不同httpmethod请求，若有需要，可自行扩展
5. post等请求支持body参数，作为body中的数据传输

实现源码如下

	module AntHttp
	  require 'net/http'

	  class HttpClient
	    def initialize(uri, content_type='application/json')
	      @uri = uri.clone
	      @content_type = content_type
	      @params = nil
	    end

	    def path(path)
	      return if path.nil? || path == ''
	      @uri.chomp!('/')
	      @uri << '/' unless path.start_with? '/'
	      @uri << path
	      self
	    end

	    def params(params)
	      @params = params
	      self
	    end

	    def get
	      uri = get_uri
	      req = Net::HTTP::Get.new(uri)
	      execute(uri, req, nil)
	    end

	    def put(body=nil)
	      uri = get_uri
	      req = Net::HTTP::Put.new(uri)
	      execute(uri, req, body)
	    end

	    def post(body=nil)
	      uri = get_uri
	      req = Net::HTTP::Post.new(uri)
	      execute(uri, req, body)
	    end

	    def delete
	      uri = get_uri
	      req = Net::HTTP::Delete.new(uri)
	      execute(uri, req, body)
	    end

	    private

	    def get_uri
	      uri = URI(@uri)
	      uri.query = URI.encode_www_form(@params) unless @params.nil?
	      uri
	    end

	    def execute(uri, req, body)
	      req.content_type = @content_type
	      req.body=body unless body.nil?
	      res = Net::HTTP.start(uri.host, uri.port) do |http|
	        http.request(req)
	      end
	      res
	    end
	  end
	end