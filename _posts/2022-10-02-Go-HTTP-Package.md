---
layout: post
title: "Go HTTP Server 学习笔记"
date: 2022-10-02
categories:
---

### 概述

在 Go 中启动一个最简单的 HTTP Server 的代码如下[示例地址](https://golang.google.cn/pkg/net/http/#example_ListenAndServe):

```
package main

import (
	"io"
	"log"
	"net/http"
)

func main() {
	// Hello world, the web server

	helloHandler := func(w http.ResponseWriter, req *http.Request) {
		io.WriteString(w, "Hello, world!\n")
	}

	http.HandleFunc("/hello", helloHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 启动流程分析

- 第一层，标准库创建 HTTP 服务是通过创建一个 Server(实质是调用了 server.ListenAndServe())。
```
func (srv *Server) ListenAndServe() error {
	if srv.shuttingDown() {
		return ErrServerClosed
	}
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(ln)
}
```
- 第二层，Server 先定义了 net.Listen() 然后 调用了 Serve() 方法。
```
func (srv *Server) Serve(l net.Listener) error {
	
    // 略
	for {
		rw, err := l.Accept()
		if err != nil {
			select {
			case <-srv.getDoneChan():
				return ErrServerClosed
			default:
			}
			if ne, ok := err.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return err
		}
		connCtx := ctx
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
		tempDelay = 0
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew, runHooks) // before Serve can return
		go c.serve(connCtx)
	}
}
```
- 第三层，Serve方法，通过 for 循环接受请求，每个连接默认开启一个 Goroutine 为其服务。
- 第四、五层，serverHandler 结构代表请求对应的处理逻辑，并且通过这个结构进行具体业务逻辑处理。
    代码太长了，只贴出最核心的部分`serverHandler{c.server}.ServeHTTP(w, w.req)`
- 第六层，Server 数据结构如果没有设置处理函数 Handler，默认使用 `DefaultServerMux` 处理请求。
```
// serverHandler delegates to either the server's Handler or
// DefaultServeMux and also handles "OPTIONS *" requests.
type serverHandler struct {
	srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
    // 略
	handler.ServeHTTP(rw, req)
}
```
- 第七层，`DefaultServerMux` 是使用 map 结构来存储和查找路由规则。


```
// DefaultServeMux is the default ServeMux used by Serve.

var DefaultServeMux = &defaultServeMux
var defaultServeMux ServeMux

// ServeMux is an HTTP request multiplexer.
// It matches the URL of each incoming request against a list of registered
// patterns and calls the handler for the pattern that
// most closely matches the URL.

type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}
```