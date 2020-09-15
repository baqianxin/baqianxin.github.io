---
layout: next
title: 'ELK:Nginx日志记录'
date: 2020-09-15 16:47:54
tags:
---
# 目的

- 目的：需要统计推广带来的转化效果
- 思路：通过统计 Nginx 日志，来分析不同渠道带来的访问量的变化。（可能再通过新增用户比去计算下转化率，不确定是不是需要这个值）

# 技术方案

- 修改 Nginx 日志格式为JSON ，通过 Qbus 收集 Nginx 的访问日志到消息队列。再使用 Logstash 同步到 ES 中；接着创建 Kibana 索引查询模板，筛选展示数据
    - Qbus
    - Elasticsearch
    - Kibana
    - Logstash
    - Nginx 日志

## 修改 Nginx 日志格式

> 日志格式修改 是为了 Logstash 同步方便，可以自动创建索引 mapping 。 在Hulk 平台修改配置。

```conf
log_format logstash '{"@timestamp":"$time_iso8601",'
               '"@version":"1",'
               '"request_id":"$request_id",'
               '"client":"$remote_addr",'
               '"url":"$uri",'
               '"status":"$status",'
               '"domain":"$host",'
               '"host":"$server_addr",'
               '"server_name":"$server_name",'
               '"request":"$request",'
               '"request_length":"$request_length",'
               '"http_x_forwarded_for":"$http_x_forwarded_for",'
               '"http_x_real_ip":"$http_x_real_ip",'
               '"size":$body_bytes_sent,'
               '"rsp_time":$request_time,'
               '"referer": "$http_referer",'
               '"ua": "$http_user_agent"'
'}';

server {
    ......
}

# 日志内容

  $request_length       请求长度（包括请求行，标题和请求正文）
  $request_method       请求的动作（get或者post）
  $request_time         请求时间(以毫秒为单位的请求处理时间（1.3.9,1.2.6）; 从客户端读取第一个字节后经过的时间)
  $request_url          完整的原始请求URL（带参数）  
  $scheme               返回用的协议，是http还是https
  $remote_addr          客户端的地址
  $remote_port          client port
  $remote_user          基本认证的身份
  $server_addr          服务端的地址
  $server_port          server port
  $server_protocol      使用的http的版本“HTTP/1.0”, “HTTP/1.1”, or “HTTP/2.0”
  $status               回应状态
  $body_bytes_sent      给你主体发送的字节
  $http_refrere         请求的上个页面来至于哪里
  $http_x_forwarded_for 代理服务器的IP地址
  $http_user_agent      浏览器的型号
  $uri                  除去域名和协议的URL

  ================upstream 模块所支持的变量==============
  $upstream_addr            处理请求的上游服务器的地址
  $upstream_cache_status    表示是否命中缓存
  $upstream_status          上游服务器的响应状态码
  $upstream_response_time   上游服务器的响应时间，精度到毫秒
  $upstream_http_$HEADER    HTTP的头部，如upstream_http_host

```



### 日志收集

> Qbus 日志收集服务：本质上还是使用 Logstash 收集 Nginx 日志文件，导入到 kafka

- 指定收集路径，日志量，保留期限
- 指定生产机
- 指定消费机

### 日志同步到ES

> 使用 Logstash 同步；kafka 做 Input , ES 做 Output;

- logstash 2.4 -> logstash 5.0 + ; 配置项有修改：[文档](https://www.elastic.co/guide/en/logstash/5.2/plugins-inputs-kafka.html)

```conf
[root@p34108v baqianxin]# cat /home/logstash/jiagu_nginx_log.conf 
input {
    kafka {
        bootstrap_servers => "10.209.xxx.xxx:xxxxx"
        group_id => "jiagu_web_log_es"
        topics => ["jiagu-web-nginx"]
        codec => json {  charset => "GB2312"}
        consumer_threads => 6  # number (optional)， default: 1
        decorate_events => false # boolean (optional)， default: false
    }
}


filter {

    mutate {
      convert => [ "status","integer" ]
      convert => [ "size","integer" ]
      convert => [ "rsp_time","float"]
      convert => [ "upstreatime","float" ]
    }

    if [client] != "-" {
        geoip {
            source => "client"
            target => "geoip"
            add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
            add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
        }
        mutate {
            convert => [ "[geoip][coordinates]", "float"]
        }
    }
}

output{
    elasticsearch {
        hosts => ["10.216.xxx.xxx:xxxxx"]
        index => "jiagu_nginx_log_%{+YYYY.MM}"
        document_type => "nginx_log"
        user => "pxxxmgdxxcxxxxx_w"
        password => "xxxxxxxxxxxx"
    }
}

```

### Kibana 筛选展示

- 按照来源IP 转换为坐标信息统计区域热力图
- 按照访问地址 统计访问量最大的接口或页面
- 按照每小时日志量来判断高峰期出现情况
- .......

> 创建查询索引；具体使用查阅 (kibana 官方文档)[https://www.elastic.co/guide/en/kibana/index.html]







## 问题记录

问题1：input-kafka 配置项错误
```
[baqianxin@p34108v ~]$ /usr/share/logstash/bin/logstash -f ~/jiagu_nginx_log.conf 
WARNING: Could not find logstash.yml which is typically located in $LS_HOME/config or /etc/logstash. You can specify the path using --path.settings. Continuing using the defaults
Could not find log4j2 configuration at path /usr/share/logstash/config/log4j2.properties. Using default config which logs to console
14:46:04.961 [main] FATAL logstash.runner - An unexpected error occurred! {:error=>#<ArgumentError:
 Path "/usr/share/logstash/data" must be a writable directory. 
 It is not writable.>,
 ....

[logstash@p34108v baqianxin]$ /usr/share/logstash/bin/logstash -f  /home/baqianxin/jiagu_nginx_log.conf 
WARNING: Could not find logstash.yml which is typically located in $LS_HOME/config or /etc/logstash. You can specify the path using --path.settings. Continuing using the defaults
Could not find log4j2 configuration at path /usr/share/logstash/config/log4j2.properties. Using default config which logs to console
14:53:39.151 [LogStash::Runner] INFO  logstash.agent - No persistent UUID file found. Generating new UUID {:uuid=>"72c79b50-3410-4474-bd70-a4a5d003d95e", :path=>"/usr/share/logstash/data/uuid"}
14:53:39.264 [LogStash::Runner] INFO  logstash.agent - No config files found in path {:path=>"/home/baqianxin/jiagu_nginx_log.conf"}
14:53:39.269 [LogStash::Runner] ERROR logstash.agent - failed to fetch pipeline configuration {:message=>"No config files found: /home/baqianxin/jiagu_nginx_log.conf. Can you make sure this path is a logstash config file?"}

解决：logstash5.0升级之后配置项修改了

```

问题2：logstash 常见自定义索引模板，需要根据版本修改模板内容

```yaml
{
  "template" : "jiagu_nginx_log-*",
  "version" : 50001,
  "settings" : {
    "index.refresh_interval" : "5s"
  },
  "mappings" : {
    "_default_" : {
      /* ***_all 再6.0被废弃*** */
      "_all" : {"enabled" : true, "norms" : false},
      "dynamic_templates" : [ {
        "message_field" : {
          "path_match" : "message",
          "match_mapping_type" : "string",
          "mapping" : {
            "type" : "text",
            "norms" : false
          }
        }
      }, {
        "string_fields" : {
          "match" : "*",
          "match_mapping_type" : "string",
          "mapping" : {
            "type" : "text", "norms" : false,
            "fields" : {
              "keyword" : { "type": "keyword" }
            }
          }
        }
      } ],
      "properties" : {
         /* ***include_in_all 再6.0被废弃*** */
        "@timestamp": { "type": "date", "include_in_all": false },
        "@version": { "type": "keyword", "include_in_all": false },
        "geoip"  : {
          "dynamic": true,
          "properties" : {
            "ip": { "type": "ip" },
            "location" : { "type" : "geo_point" },
            "latitude" : { "type" : "half_float" },
            "longitude" : { "type" : "half_float" }
          }
        }
      }
    }
  }
}



```