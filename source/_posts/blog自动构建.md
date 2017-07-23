title: BLOG部署
date:
tags: 
mp3: 
cover: http://photo.weibo.com/2008451101/wbphotos/large/mid/4033554566500514/pid/77b6881djw1f91kgdgi52j20u00u0tc8
---


### 实现目标
任何终端一次构建,之后只需要git push即可自动更新博客,后续只更新博文的话git环境即可


### 基础环境

- 安装[node.js](https://nodejs.org/en/)
- 安装[git](https://git-scm.com/)
- 安装[hexo](https://hexo.io/)

> 我用的系统是WIN10和DEBIAN 8.6,下面的例子是基于WIN10的    
> 需要注意的就是命令行用的是git bash,安装node.js和git的时候记得把环境变量勾上


### 部署过程
```
$ hexo init blog #初始化博客

创建github pages，参考https://pages.github.com/

commit一个index.html到master来生成主分支

hexo生成部分push到blog分支

在https://github.com/settings/tokens生成一个TOKEN记录备用,
权限只需要Full control of private repositories


```



### Travis-ci部分


- 用GitHub登录[Travis-ci](https://travis-ci.org)自动关联，选择同步github pages库

- 在more options里把Build branch updates开关打开

- Environment Variables里创建KEY为GH_TOKEN,VALUE为TOKEN的环境变量

- 在本地仓库blog分支创建 **.travis.yml**文件,内容如下
```
language: node_js
node_js: stable

# Travis-CI Caching
cache:
  directories:
    - node_modules


# S: Build Lifecycle
install:
  - npm install

before_script:
 # - npm install -g gulp


script:
  - hexo g


after_script:
  - cd ./public
  - git init
  - git config user.name "yourusername"
  - git config user.email "yourusername@xx.com"
  - git add .
  - git commit -m "Update docs"
  - git push --force --quiet "https://${GH_TOKEN}@${GH_REF}" master:master
# E: Build LifeCycle

branches:
  only:
    - blog
env:
 global:
   - GH_REF: github.com/yourusername/yourgithubpage.github.io.git
```
> 最后只要git push到blog分支就可以等Travis-ci自动构建静态页面部署到master,github pages上也就有了