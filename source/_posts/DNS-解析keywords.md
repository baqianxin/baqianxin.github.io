---
layout: next
title: DNS 解析keywords
date: 2020-09-15 16:30:45
tags:
---

# DNS 解析keywords
```
graph LR
A(Client)-->B(Local DNS)
```
```
Client --> hosts文件 
--> DNS Service Local Cache --> DNS Server (recursion递归查看本地配置的解析文件) 
--> Server Cache 
--> iteration(迭代) --> 根 --> 顶级域名DNS --> 二级域名DNS…
最终本地dns查看结果后返回给客户端
```
> * DNS 解析，若缓存命中就会直接返回缓存结果，不再进行递归DNS查询

**缓存**
- DNS解析缓存数据，在递归获取真实 访问IP之后更新


**递归**
- 从缓存数据中无法获取目的地IP，则开始递归查询各级域名服务器，获取最终IP
    - 根域名服务器
    - 顶级域名服务器
    - **权威** 域名服务器

**权威**
- 域名对应的真实IP存储节点，由权威服务器选择返回域名对应的真实且最合适的IP


**Q.域名劫持**
- 并对终端用户的 Local DNS 进行篡改，指向伪造的 Local DNS ，返回错误的 IP 信息
- 监听用户的域名解析请求，伪造的 DNS 解析响应传递给用户
- **DNS缓存污染** ，Local DNS缓存数据被修改了

**Q.精准调度**
- 区域
- 运营商

> 除了解析转发对调度精准性带来的影响外，Local DNS的布署情况同样影响着域名智能解析的精准性。

**Q.DNS解析服务耗时**
- DNS服务部署不合理导致的解析域名耗时

**Q.权威服务数据更新滞后**
- 域名IP变更，主要发生在权威服务上，同步到全网DNS缓存延迟严重