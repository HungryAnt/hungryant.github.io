---
layout: post
title:  "C#与Java服务通过Http协议通讯实例"
date:   2015-11-10 00:00:00
categories: csharp
---

## HttpClient类型 Nuget
todo

## 客户端HttpClient封装
todo

## 基本MD5签名安全验证

签名认证 todo

### MD5签名工具类c#/java实现
- c#实现

		public static class MD5SignatureUtil
		{
		    public static String GetSignAsHex(String rowText)
		    {
		        MD5 md5 = new MD5CryptoServiceProvider();
		        byte[] md5Bytes = md5.ComputeHash(Encoding.UTF8.GetBytes(rowText));
		        StringBuilder hexStringBuilder = new StringBuilder();
		        foreach (var md5Byte in md5Bytes)
		        {
		            hexStringBuilder.Append(md5Byte.ToString("x2"));
		        }
		        return hexStringBuilder.ToString();
		    }
		}

	对应单元测试

		[TestClass()]
		public class MD5SignatureUtilTest
		{
		    [TestMethod()]
		    public void GetSignAsHexTest()
		    {
		        string rowText = "ant111AntLogin";
		        string expected = "71736cfd60d4523017cc23ae12231c13";
		        string actual;
		        actual = MD5SignatureUtil.GetSignAsHex(rowText);
		        Assert.AreEqual(expected, actual);
		    }
		}

- java实现

		public class MD5SignatureUtil {
		    public static String getSignAsHex(String rowText) {
		        return DigestUtils.md5DigestAsHex(getBytes(rowText));
		    }

		    private static byte[] getBytes(String text) {
		        try {
		            return text.getBytes("utf-8");
		        } catch (UnsupportedEncodingException e) {
		            throw new RuntimeException(e);
		        }
		    }
		}

	对应单元测试

		public class MD5SignatureUtilTest {
		    @Test
		    public void testGetSignAsHex() {
		        String expectedSign = "71736cfd60d4523017cc23ae12231c13";
		        String actual = MD5SignatureUtil.getSignAsHex("ant111AntLogin");
		        assertEquals(expectedSign, actual);
		    }
		}