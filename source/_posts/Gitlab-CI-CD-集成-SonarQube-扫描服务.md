---
layout: next
title: Gitlab CI/CD 集成 SonarQube 扫描服务
date: 2020-09-15 16:26:48
tags:
---
# Gitlab CI/CD 集成 SonarQube 扫描服务

[TOC]

    环境：Docker
    
    gitlab
    gitlab-runner
    sonarqube
    postgresql
    

## 使用示例
  
```yml
# .gitlab-ci.yml

stages:
  - sonar-scanner
sonar:
  stage: sonar-scanner
  script:
    - sonar-scanner 
      -Dsonar.projectKey=cd_demo 
      -Dsonar.sources=. 
      -Dsonar.host.url=http://10.18.27.80:9823 
      -Dsonar.login=a138bc0d36c7130bb30aebbaffbc44148b6ab8e4
  tags:
    - sonar
  when: always
```
    
    

## Sonar 服务
- [SonarQube 代码检查服务部署](http://note.youdao.com/noteshare?id=61644a0153db0c34075be66430fbe3c4)


### Sonar 集成 GitLab
- admin 安装 git 插件 `Administration`->`marketpalce`-> `search git`->`restart server`
- `ALM Intergrations`
![sonar-gitlab](https://note.youdao.com/yws/api/personal/file/E6645A1C4C6E47B78402FED13B5F369A?method=download&shareKey=814d1fda2ad408c56016122cef5c90fa)
- `Gitlab Application` `Token-Secret`
![Gitlab Application T-S](https://note.youdao.com/yws/api/personal/file/4DE448654E9F41D7A1CC7AE6A8E375CF?method=download&shareKey=90a053673cc128ed3be7b9cf566b5bc6)

## Gitlab-runner 服务
- 构建 `gitlab-runner` 镜像，集成 `node` ,`sonar-scanner`
- 注册 `Runner` ：
    - `gitlab-runner` `register`
    - 输入 `gitlab-host`
    - 输入 `runner-token`
    - 输入 `tag`
    - 选择执行方式 `shell`
- 开启 `Runner` 不匹配 `tag` 执行,在 `CI/CD` -> `runner` 的设置里

![注册](https://note.youdao.com/yws/api/personal/file/B4C37DC2CA36420180D3CC8C00A1DC4D?method=download&shareKey=d17249b1446369fff0a6912c6f824eb7)


--------
### Gitlab-runner 容器编排

    docker-compose.yml

```yml
runner:
  container_name: 'gitlab-runner'
  build: ../../server/gitlab-runner/
  restart: always
  ports:
    - '8093:8093'
  # volumes:
  #   - '$GITLAB_HOME/gitlab-runner/config:/etc/gitlab-runner'
  #   - '/var/run/docker.sock:/var/run/docker.sock'
web:
  image: 'gitlab/gitlab-ee:latest'
  restart: always
  hostname: 'gitlab.example.com'
  environment:
    GITLAB_OMNIBUS_CONFIG: |
      external_url 'http://gitlab.example.com:8929'
      gitlab_rails['gitlab_shell_ssh_port'] = 2224
  ports:
    - '8929:8929'
    - '2224:22'
  volumes:
    - '$GITLAB_HOME/config:/etc/gitlab'
    - '$GITLAB_HOME/logs:/var/log/gitlab'
    - '$GITLAB_HOME/data:/var/opt/gitlab'

```
    
    gitlab-runner/Dockerfile

```
FROM gitlab/gitlab-runner:latest

LABEL MAINTAINER=baqianxin@360.cn

RUN export LANG=en_US.UTF-8  && export LANGUAGE=en_US

RUN apt-get update &&  apt-get  install -y  nodejs vim unzip 
RUN cd /opt && \ 
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.0.0.1744-linux.zip && \
unzip sonar-scanner-cli-4.0.0.1744-linux.zip && \ 
mv sonar-scanner-4.0.0.1744-linux sonar-scanner

RUN  ln -s /opt/sonar-scanner/bin/sonar-scanner /usr/bin/sonar-scanner && sonar-scanner -v


```

### 注意问题
- 系统语言需设置 LCALL LANGUAGE LANG=en_US.UTF-8
    
    
    
    
    
    
    
    
    
    
    
    
    
    [submodule "golang/example"]
    	active = true
    	url = git@github.com:baqianxin/examples.git
    [submodule "spider/chineseocr_lite"]
    	url = git@github.com:baqianxin/chineseocr_lite.git
    	active = true
