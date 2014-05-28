---
layout: post
title: levelDB Status分析
categories:
- Technology
tags:
- levelDB
- status
- 返回状态
---
# 概括
返回状态被封装成Status类, 其中 用char* 来表示一个状态, char* = {length of message, status code, message};
用char*来表示message,应该是出于比string更省空间的考虑.
status code用enum来编码

<pre class="preetyPrint">
	enum Code {
    		kOk = 0,
    		kNotFound = 1,
   		kCorruption = 2,
    		kNotSupported = 3,
    		kInvalidArgument = 4,
    		kIOError = 5
  	};
</pre>

Status提供 产生状态的const函数, 和 判断何种状态的方法.

# source

/leveldb/status.h

/util/status.cc


<pre class="prettyPrint">

</pre>

# 总结
	