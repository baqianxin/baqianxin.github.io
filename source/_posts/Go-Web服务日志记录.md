---
layout: next
title: 'Go:Web服务日志记录'
date: 2020-09-15 16:42:50
tags:
---
# GoWeb 服务日志管理

## 前情
最新使用 Go 语言开发了一个滑动验证码 SDK 的服务端，提供的接口都具备基础功能了，但是业务逻辑日志以及请求日志没有一个好的记录方式，而且使用的容器服务，每次发布都会刷新容器，文件类的日志都会丢失。因此查了一下常见的Go  Log 处理方式。发现 logrus  包使用的人不少，感觉也很好使用毕竟带 hook （带钩子的都是工具人...）

## 目的
- 记录所有请求日志
- 记录业务逻辑关键信息
- 方便查询检索分析

## 实现细节

1. 自定义日志服务中间件，
2. 使用 ES 存储日志数据
3. 过滤接口检查的调用日志
4. ES使用时间索引
5. handler路由使用中间件拦截记录请求日志Mux.Use(m-)
4. 逻辑日志纪录使用全局Loges 对象


### Go项目-使用ES

### 自定义日志收集中间件
>  logrus	"github.com/sirupsen/logrus"

```go
package middleware

...

var Loges = logrus.New()

func init() {
	InitElasticForLog()
}

....


// ES日志中间件  用于收集请求日志
func AccessLogging(f http.Handler) http.Handler {

	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		_ = r.ParseForm()
		buf := new(bytes.Buffer)
		_, _ = buf.ReadFrom(r.Body)
		logEntry := Loges.WithFields(logrus.Fields{
			"host":         r.Host,
			"ip":           r.RemoteAddr,
			"method":       r.Method,
			"path":         r.RequestURI,
			"query":        r.URL.RawQuery,
			"request": buf.String(),
		})
		wc := &ResponseWithRecorder{
			ResponseWriter: w,
			statusCode:     http.StatusOK,
			body:           bytes.Buffer{},
		}
		f.ServeHTTP(wc, r)
		if !IsUnlogHost(r.Host, r.RequestURI) {
			defer logEntry.WithFields(logrus.Fields{
				"status":        wc.statusCode,
				"respone": wc.body.String(),
			}).Info()
		}

	})
}

```


> 第三方包 elastic	"github.com/olivere/elastic"



```go
client, err := elastic.NewClient(

	elastic.SetURL("http://xx.xx.xx.xx:9029"),
	
	elastic.SetBasicAuth("writerUser", "u_pwd"),
	// 其他配置项
	//
	//
	//
)
if err != nil {
	Loges.Panic(err)
}

```


### logrus 使用ES-hook

```go
// 使用时间索引
//"2006-01-02 15:04:05"
t := time.Now()
date := t.Format("200601")
hook, err := elogrus.NewElasticHook(client, GetHost(), logrus.DebugLevel, "YOUR_SERVER_NAME_LOG_"+date)
if err != nil {
	Loges.Panic(err)
}
Loges.AddHook(hook)

```

### 自定义响应数据格式
```
type ResponseWithRecorder struct {
	http.ResponseWriter
	statusCode int
	body       bytes.Buffer
}

func (rec *ResponseWithRecorder) WriteHeader(statusCode int) {
	rec.ResponseWriter.WriteHeader(statusCode)
	rec.statusCode = statusCode
}

func (rec *ResponseWithRecorder) Write(d []byte) (n int, err error) {
	n, err = rec.ResponseWriter.Write(d)
	if err != nil {
		return
	}
	rec.body.Write(d)

	return
}

```


### 拦截服务检查的日志记录
```
if !IsUnlogHost(r.Host, r.RequestURI) {
	defer logEntry.WithFields(logrus.Fields{
		"status":        wc.statusCode,
		"respone": wc.body.String(),
	}).Info()
}


func IsUnlogHost(h string, url string) bool {
	if strings.Contains(h, `lvscheck.xitong.xxx`) && url == "/status" {
		return true
	} else {
		return false
	}
}
```

### 业务逻辑使用
```go
//

```

### 效果
![Kibana](http:///asdasdasda.com)




## 注意
- 在日志中间件获取请求Body时 可能导致接口服务无法正常接受提交参数的问题:可以在获取请求参数时预先使用 ParseForm() 
```
// middleware

_ = r.ParseForm()
buf := new(bytes.Buffer)
_, _ = buf.ReadFrom(r.Body)
...

// API
req.ParseForm()
var paramsList = []string{"appId", "token".....}
e, reqData := GetParams(req.Form, paramsList, paramsList)

```
