---
title: gin框架入门
date: 2019-02-04
categories:
 - go
tags:
 - gin
 - go 
---

* [下载安装](#下载安装)
* [路由](#路由)
* [中间件](#中间件)
* []


## 1.下载安装

```
go get -u github.com/gin-gonic/gin
```


## 2.路由

```go
	r ：= gin.Default()
	r.GET("/",func (ctx *gin.Context) {

		})
	r.POST("/",func (ctx *gin.Context)) {

		})

	r.Run(":9999")	

```

## 3.中间件

```go
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        t := time.Now()

        // Set example variable
        c.Set("example", "12345")

        // before request

        c.Next()

        // after request
        latency := time.Since(t)
        log.Print(latency)

        // access the status we are sending
        status := c.Writer.Status()
        log.Println(status)
    }
}

func main() {
    r := gin.New()
    r.Use(Logger())

    r.GET("/test", func(c *gin.Context) {
        example := c.MustGet("example").(string)

        // it would print: "12345"
        log.Println(example)
    })

    // Listen and serve on 0.0.0.0:8080
    r.Run(":8080")
}

```