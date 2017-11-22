---
layout: post
title: golang negroni-gzip 源码分析
comments: true
description: 对negroni 中间件　negroni-gzip的个人分析
tags:
    - golang
    - negroni
    - negroni-gzip
---
negroni-gzip是基于negroni开发的对请求内容进行gzip压缩以加快加载速度的中间件。
用法比较简单。只要把它放在其它需要回应请求的中间件前面就好了。比如

```
package main

import (
    "fmt"
    "net/http"

    "github.com/urfave/negroni"
    "github.com/phyber/negroni-gzip/gzip"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
    	  fmt.Fprintf(w, "Welcome to the home page!")
    })

    n := negroni.Classic()
    n.Use(gzip.Gzip(gzip.DefaultCompression))　//使用默认的压缩级别
    n.UseHandler(mux)
    n.Run(":3000")
}
```
首先从`Gzip`这个函数开始分析
```
func Gzip(level int) *handler {
	h := &handler{}
	h.pool.New = func() interface{} {
		gz, err := gzip.NewWriterLevel(ioutil.Discard, level)
		if err != nil {
			panic(err)
		}
		return gz
	}
	return h
}
```
第一行定义了一个handler结构体的指针
并设置成员变量`pool`.

`handler`结构如下
```
type handler struct {
	pool sync.Pool
}
```


`sync.Pool`的结构
```
type Pool struct {

        // New optionally specifies a function to generate
        // a value when Get would otherwise return nil.
        // It may not be changed concurrently with calls to Get.
        New func() interface{}
        // contains filtered or unexported fields
}
```
我摘取了官方文档对`sync.Pool`的部分说明
>A Pool is a set of temporary objects that may be individually saved and retrieved.
>Any item stored in the Pool may be removed automatically at any time without notification. If the Pool holds the only reference when this happens, the item might be deallocated.
>A Pool is safe for use by multiple goroutines simultaneously.
>Pool's purpose is to cache allocated but unused items for later reuse, relieving pressure on the garbage collector. 
{: .blockquote }
`Pool`会维持一个共享的对象池，，`Pool.Get()`会从中任意移除一个实体并返回，如果对象池空了，在`pool.New != nil`的情况下，会调用`New()`返回一个新的对象，否则返回`nil`
利用`pool`减少内存分配，可以减轻GC的压力，并且是goroutine安全的。
所以这段代码目的是为handler维持一个共享的`pool`,获得并发性能上的提升
对象池里面是一个`*gzip.Writer`对象

`Gzip`返回一个`handler`指针,也就是一个negroni-gzip中间件
`handler`实现了`ServeHTTP(w http.ResponseWriter, r *http.Request, next http.HandlerFunc){...}`方法

`ServeHttp`的处理流程
```
1. 处理客户端不能使用gzip编码的情况
2. pool.Get()返回一个共享的*gzip.Writer对象gz
3. gz.Reset(w)将输出定向到http.ResponseWriter
4. 将http.ResponseWriter转换成negroni.NewResponseWriter(w)
5. gzipResponseWriter将gzip.Writer和negroni.ResponseWriter封装在一起
6. 将配置好的gzipResponseWriter交给下一个中间件
7. 处理完要将gz放回pool
```
可以看出，每次请求都会从`pool`中获取`*gzip.Writer`,用完后放回pool,通过Pool的方式实现了多个goroutine之间安全共享同一个`*gzip.Writer`变量
`gzipResponseWriter`实现了`http.ResponseWriter`接口

`writeHeader(code int)`设置响应头的编码方式和状态码
```
// Check whether underlying response is already pre-encoded and disable
// gzipWriter before the body gets written, otherwise encoding headers
func (grw *gzipResponseWriter) WriteHeader(code int) {
	headers := grw.ResponseWriter.Header()
	if headers.Get(headerContentEncoding) == "" {
		headers.Set(headerContentEncoding, encodingGzip)
		headers.Add(headerVary, headerAcceptEncoding)
	} else {
        //如果响应已经编码了，就将grw.w重定向到null,将grw.w设为nil
		grw.w.Reset(ioutil.Discard) 
		grw.w = nil
	}
	grw.ResponseWriter.WriteHeader(code)
	grw.wroteHeader = true
}


```
```
// Write writes bytes to the gzip.Writer. It will also set the Content-Type
// header using the net/http library content type detection if the Content-Type
// header was not set yet.
func (grw *gzipResponseWriter) Write(b []byte) (int, error) {
	if !grw.wroteHeader {
		grw.WriteHeader(http.StatusOK)
	}
    // 如果已经编码了，grw.w会为nil，则直接调用ResponseWriter.Write并返回
	if grw.w == nil {
		return grw.ResponseWriter.Write(b)
	}
	if len(grw.Header().Get(headerContentType)) == 0 {
		grw.Header().Set(headerContentType, http.DetectContentType(b))
	}
	return grw.w.Write(b)
}
```


这个中间件把`compress/gzip`封装为一个可以对http响应文件编码的gzipResponseWriter，通过临时对象池共享`*gzip.Writer`，提高并发性能。是一个高效的中间件。