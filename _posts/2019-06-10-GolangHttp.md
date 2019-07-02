---
layout: post
title: Golang HTTP包详解
categories: Golang
description: Golang HTTP包详解
keywords: Golang,HTTP,TCP
---

## 写在前面
在push项目中，有不少需要调用第三方Http服务的场景。这类场景和使用浏览器访问普通网站稍有区别，因为浏览器访问网页时，不会有类似push的大量调用同一接口的情况，因此对于接口访问时的连接复用的考虑并不复杂。我在查看push的TCP连接情况时，发现线上机器有大约2000多的TIME_WAIT连接，基本都是访问外网的请求。在之前一次做国际化推送业务调用对方服务http接口时，更是出现两三万TIME_WAIT的情况。虽然TIME_WAIT本身占用资源不多，但是大量TIME_WAIT连接还是不便管理，且容易引发文件描述符不够的错误。



## HTTP Client
### Golang为我们做了什么？
在测试机上，我们部署了一个server服务监听了8080端口，并设置了50ms的响应时间。
我们先来对比下以下两个程序在网络表现上的差异：

```shell
#!/usr/bin/env bash

while :
do
     curl http://127.0.0.1:8080/hi
done
```

```golang
package main

import (
	"flag"
	"fmt"
	"io/ioutil"
	"net"
	"net/http"
	"time"
)

var max = flag.Int("max", 2, "")

func main() {
	flag.Parse()
	defaultTransport := &http.Transport{
		Proxy: http.ProxyFromEnvironment,
		DialContext: (&net.Dialer{
			Timeout:   5 * time.Second,
			KeepAlive: 10 * time.Second,
			DualStack: true,
		}).DialContext,
		MaxIdleConns:          100,
		IdleConnTimeout:       10 * time.Second,
		TLSHandshakeTimeout:   5 * time.Second,
		ExpectContinueTimeout: 1 * time.Second,
	}
	defaultTransport.MaxIdleConnsPerHost = *max
	client := &http.Client{
		Timeout:   10 * time.Second,
		Transport: defaultTransport,
	}
	for {
		resp, err := client.Get("http://127.0.0.1:8080/hi")
		if err != nil {
			fmt.Println(err)
		} else {
			// 一定要加，因为readLoop等待上层调用函数读取完body才会把连接回收掉
			_, _ = ioutil.ReadAll(resp.Body)
			fmt.Println(resp.StatusCode)
		}
		time.Sleep(1 * time.Millisecond)
	}
}

```
以上两个程序分别采用循环运行curl命令和在一个go程序中循环调用client测试server的接口。
在实验中，观测两个程序的运行结果，
![](/images/httpclient/实验1.1.jpg)
![](/images/httpclient/实验1.2.jpg)

可以看出，在执行同样一个任务时，在单一的go程序内对TCP连接是做了复用的。


### Golang实现TCP连接池的过程
整体流程：
![](/images/httpclient/整体流程.png)
TCP连接池：
![](/images/httpclient/连接池.png)
调用连接处理Http请求：
![](/images/httpclient/DoRequest.png)

### Transport参数优化

连接池复用：

> QPS < (idle conns size) * (1000(ms) / 请求的响应时间(ms))

在

















## HTTP/2.0 和 TLS


## 实战技巧



