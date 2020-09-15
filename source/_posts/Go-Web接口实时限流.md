---
layout: next
title: 'Go:Web接口实时限流'
date: 2020-09-15 16:58:03
tags: Golang API LimitRate
---
# Golang 接口实时限流功能

## 实现需求
 对于同一IP，同一APP包名的请求，每小时只允许100次
 
## 方案逻辑
 1.自定义限流中间件，对需要限流的接口做监听 
 
 2.使用指定参数生成唯一Key,记录请求频率
 
 3.若请求时间超过key的有效时间则重新计数（！是否触发惩罚操作，更新有效时间为2小时）
 
 4.复杂逻辑-结合业务数据限流-指定设备ID|指定...
### 代码
```golang

// 使用
mux.Use(middleware.LimitRate)


// 限流中间件
package middleware

import (
	"crypto/md5"
	"encoding/hex"
	"github.com/garyburd/redigo/redis"
	"jiagu-user-service/internal/captcha/common"
	"net/http"
	"sync"
)

type RequestLimitService struct {
	Interval int
	MaxCount int
}

// 设置过期时间 设置频率数值
func NewRequestLimitService(interval int, maxCnt int) *RequestLimitService {
	if common.IsDev() {
		interval = interval / 10
		maxCnt = maxCnt / 10
	}
	reqLimit := &RequestLimitService{
		Interval: interval,
		MaxCount: maxCnt,
	}
	return reqLimit
}

func (reqLimit *RequestLimitService) IsAvailable(r *http.Request) bool {
	// 获取必要的验证参数// IP  // pn  // path /api/v1/auth
	_ = r.ParseForm()
	ip := r.RemoteAddr
	pn := r.Form.Get("pn")
	path := r.RequestURI
	if path == common.API_PATH_V1_AUTH {
		limitRateKey := Md5V(ip + pn)
		c := reqLimit.GetLimitCountCache(common.REDIS_LIMIT_RATE_PREFIX + limitRateKey)
		return c < reqLimit.MaxCount
	}
	return true

}

func (reqLimit *RequestLimitService) GetLimitCountCache(limitRateKeyCount string) int {
	redisCw := RedisW.Get()
	redisCr := RedisW.Get()
	defer redisCr.Close()
	defer redisCw.Close()

	var c = 1
	//检查Key 存在与否
	isKeyEx, err := redis.Bool(redisCr.Do("EXISTS", limitRateKeyCount))

	CheckErr(err, false)

	if isKeyEx {
		_, _ = redisCw.Do("INCR", limitRateKeyCount)
		c, _ = redis.Int(redisCr.Do("GET", limitRateKeyCount))
	} else {
		_, err = redisCw.Do("SET", limitRateKeyCount, 1, "EX", reqLimit.Interval)
		CheckErr(err, false)
	}
	return c
}

var RequestLimit = NewRequestLimitService(common.REDIS_LIMIT_RATE_EXPIRE, common.API_LIMIT_RATE_NUM)

func Md5V(str string) string {
	h := md5.New()
	h.Write([]byte(str))
	return hex.EncodeToString(h.Sum(nil))
}
func LimitRate(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {

		if RequestLimit.IsAvailable(r) {
			next.ServeHTTP(w, r)
		} else {
			http.Error(w, http.StatusText(http.StatusTooManyRequests), http.StatusTooManyRequests)
			return
		}
	})
}



```


