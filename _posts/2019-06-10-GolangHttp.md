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

import (...)

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
		//go func() {
		resp, err := client.Get("http://127.0.0.1:8080/hi")
		if err != nil {
			fmt.Println(err)
		} else {
			// 一定要加，因为readLoop等待上层调用函数读取完body才会把连接回收掉
			_, _ = ioutil.ReadAll(resp.Body)
			fmt.Println(resp.StatusCode)
		}
		time.Sleep(1 * time.Millisecond)
		//}()
	}
}

```
以上两个程序分别采用循环运行curl命令和在一个go程序中循环调用client测试server的接口。
在实验中，观测两个程序的运行结果，
![](/images/httpclient/实验1.1.jpg)
![](/images/httpclient/实验1.2.jpg)

可以看出，在执行同样一个任务时，在单一的go程序内对TCP连接是做了复用的。

在生产环境中，我们的需求是程序尽量少的建立新的连接，尽可能多的复用连接，这样可以以最少的socket资源完成大额并发请求，同时减少建立连接的时间开销。

### Golang实现TCP连接池的过程
整体流程：
![](/images/httpclient/整体流程.png)
TCP连接池：
![](/images/httpclient/连接池.png)
调用连接处理Http请求：
![](/images/httpclient/DoRequest.png)

理解完对TCP连接复用的整体实现后，我们可以得出比较重要的几点结论：

- 使用TCP长连接复用，必须开启KeepAlive，Http1.0默认不支持，Http1.1默认支持；
- 单个transport是并发安全的，可以同时被多个goroutine使用；
- 单个HttpClient包含了对TCP连接的复用，当连接不够用时，会自动dialTCP连接；
- Go维护了一个记录当前可复用TCP连接的栈式结构，并且是每个目标ip单独维护，用`MaxIdleConnsPerHost`表示
- 连接使用完后会放回连接池中，但是必须等待调用者读取完response body。
- 对于一个连接，有如下几种情况会导致关闭：
	- 关闭全部连接，一般是调用httpClient.Close，或者程序退出；
	- idleConn池满时放入；
	- idleLRU满，一般指全部的连接达到MaxIdleConn，删除最老的；
	- 请求取消；
	- readLoop Peek报错；
	- 在idleConn中的连接达到IdleConnTimeOut；
	- readLoop发生异常退出时；
	- writeLoop报错；
	- response超时。

### Transport参数优化
idle conn 在整个请求复用的过程中发挥如下作用，理论上来讲，如果请求的速率和请求结束放回的速率一致，那么idle池的大小设为1就可以，但是实际上两者的速率并不能始终保持一致，所以需要一个较大的缓冲区，存放idle连接。在golang的实现里，这个复用缓冲区是栈式存取的。

![](/images/httpclient/transport.png)

我在实际测试时，想实时监测idleConn池的大小，但是这个是http库的私有变量，无法直接采样，且http库不能修改代码，于是看了代码发现httptrace包有一个ClientTrace，可以在请求内部注册事件函数，用于捕获连接的某些关键信息，代码如下：

```golang
package main
import (...)
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
	//duration := 1000000 / *qps
	for {
		go func() {
			//start := time.Now()
			request, _ := http.NewRequest("GET", "http://127.0.0.1:8080/hi", nil)
			resp, err := client.Do(request.WithContext(httptrace.WithClientTrace(context.Background(), &httptrace.ClientTrace{
				PutIdleConn: func(err error) {
					if err != nil {
						fmt.Println(err)
					}
				},
				//四个要一起写
				DNSStart: func(info httptrace.DNSStartInfo) {
				},
				DNSDone: func(info httptrace.DNSDoneInfo) {
				},
				ConnectStart: func(network, addr string) {
				},
				ConnectDone: func(network, addr string, err error) {
					fmt.Println(network, addr, err)
				},
			})))
			if err != nil {
				fmt.Println(err)
			} else {
				_, _ = ioutil.ReadAll(resp.Body)
			}
		}()
		time.Sleep(1 * time.Millisecond)
	}
}
```
测试某一接口时，可以通过调整max参数观察不同参数下的表现。

测试MaxIdleConnsPerHost:

测试IdleConnTimeout:

测试100QPS:



### HTTP/2.0


