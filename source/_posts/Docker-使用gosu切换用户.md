---
layout: next
title: Docker 使用gosu切换用户
date: 2020-09-15 16:40:59
tags:
---
# 容器启动时切换用户gosu执行脚本

## 问题
想在小组内部推一下代码检查工具Sonar，申请了容器空间用于部署。在自己本地编译镜像之后推送到公司镜像源之后，发现拉取后启动容器失败了：检查输出日志
```bash
java.lang.RuntimeException: don't run elasticsearch as root.
        at org.elasticsearch.bootstrap.Bootstrap.initializeNatives(Bootstrap.java:94)
        at org.elasticsearch.bootstrap.Bootstrap.setup(Bootstrap.java:160)
        at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:286)
        at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:35)
```
这就奇怪了，本地构建的镜像，运行都是正常的啊；难道公司容器服务启动的时候会强制以root账户运行？

## 排查
> 本地构建镜像的时候修改 ENDPOINT 脚本 `run.sh` ；输出当前执行用户 whoami ;确实是  `root`

### 尝试1
```Dockerfile
  # 前置切换用户
  USER sonarqube
  ENDPOINT ["run.sh"]
  # 推送，部署还是报错 ES 无法使用 root 账户启动
```

### 尝试2

```bash
  # 修改ENDPOINT
  
  ENDPOINT su - sonarqube -s "./run.sh"
  # 推送 部署还是错误： 环境变量，账户目录/home/sonarqube(本来就没创建账户目录)都找不到了
   
```

### 尝试3【√】
```bash
    # 修改镜像 添加 gosu / su-exec (sonarqube 官方脚本有这个)
    RUN apt update && apt install gosu -y
    
    # 同时修改 run.sh 脚本
    exec gosu sonarqube "$cmd"
    
    # 推送&启动，OK
    
```
