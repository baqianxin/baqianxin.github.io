---
layout: next
title: Sonar代码检查服务部署
date: 2020-09-15 16:36:41
tags:
---


SonarQube代码检查服务部署
---

>前言： 小组内的代码检查服务
<br/> 部署： Docker

[TOC]

## 服务构成
- PostgreSQL
- SonarQube

## 镜像

### SonarQube
```Dockerfile
FROM sonarqube:8.2-community
COPY run.sh /opt/sonarqube/bin/
USER root
RUN apt-get update && apt install -y gosu

USER sonarqube
ENTRYPOINT ["/opt/sonarqube/bin/run.sh"]

        # run.sh
        .
        # map legacy env variables
        set_prop_from_env_var "sonar.jdbc.username" "${SONARQUBE_JDBC_USERNAME:-}"
        set_prop_from_env_var "sonar.jdbc.password" "${SONARQUBE_JDBC_PASSWORD:-}"
        set_prop_from_env_var "sonar.jdbc.url" "${SONARQUBE_JDBC_URL:-}"
        set_prop_from_env_var "sonar.web.javaAdditionalOpts" "${SONARQUBE_WEB_JVM_OPTS:-}"
        
        exec gosu sonarqube java -jar "lib/sonar-application-$SONAR_VERSION.jar" \
          -Dsonar.log.console=true \
          "${sq_opts[@]}" \
          "$@"
```
### PostgreSQL

```Dockerfile
FROM postgres:latest
COPY postgres.log /opt/
# POSTGRESQL使用最新镜像
```

### 推送到私有仓库，远端部署
``` bash
starkctl push -u xxx -p xxx r.addops.xxx.cn/namespace/imagename:tag
```

### 其他
```yml
# 本地服务Docker-compose.yml
version: '3'
services:
  mydb:
    image: postgres
    volumes:
      - ./data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: sonar
      POSTGRES_DB: sonar
      POSTGRES_PASSWORD: sonar
    ports:
      - "5433:5432"
  sonarqube:
    image: sonarqube
    environment:
      sonar.jdbc.username: sonar
      sonar.jdbc.password: sonar
      sonar.jdbc.url: jdbc:postgresql://mydb:5432/sonar
    ports:
      - "9823:9000"
    volumes:
      - .sonar:/home/sonarqube/.sonar
      - ./scaner:/opt/sonarqube/scanner
```

### 问题
> 启动容器的时候会启动内置的 ES 服务，这里要求不能使用 root 账户运行,需要用到 gosu 或 su-exec 容器内切换用户执行脚本


### 相关连接

\# https://docs.docker.com/develop/develop-images/dockerfile_best-practices <br/>
\# https://github.com/SonarSource/docker-sonarqube


