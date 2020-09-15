---
layout: next
title: 'Linux:SSH 隧道功能'
date: 2020-09-15 17:00:39
tags: Linux SSH 隧道
---

```
 # 检查代理服务器是否可以转发
 // sshd 服务配置
 cat /etc/ssh/sshd_config | grep AllowTcp

 // 系统转发规则
 cat /etc/sysctl.conf | grep ip_forward


 // 修改配置文件 修改
 sysctl -w net.ipv4.ip_forward=1
 
 // 检查运行时配置
 cat /proc/sys/net/ipv4/ip_forward
 
 // 退出 重新连接
 ssh -L 8080:127.0.0.1:8080 root@121.xxxx.xxx.xxx
 
 本地端口：目标机器：端口
```