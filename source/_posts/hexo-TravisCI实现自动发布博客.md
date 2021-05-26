---
title: hexo+TravisCI实现自动发布博客
date: 2021-05-26 15:07:11
tags: hexo
categories:
---

以下步骤假设你已经掌握了hexo基本命令和原理，并且已经初始化好了hexo，也安装好了自定义主题。

##### Hexo原理

Hexo简单来说能够把md渲染成html文件，然后把渲染后的结果发布到github Page上，我们只要关注博客内容即可。几个关键命令：

- hexo init(初始化hexo)
- hexo generate(开始渲染)
- hexo deploy(部署到github)
- hexo server(本地部署)

##### Hexo+TravisCI原理

- github Page使用的仓库名为username.github.io
- hexo渲染后的结果必须放在**master**分支，里面是渲染后的html
- 新建一个**hexo-source**分支用来存放原始的文件，里面是渲染前的md文件
- 平时在hexo-source分支上编写md文档，push到hexo-source远程分支上会触发TravisCI进行构建，把构建后的结果发布到master分支，实现push即发布的效果。

##### 配置过程

###### 配置github

1. 在一个空白目录下执行 hexo init，初始化一个hexo目录

2. 在此目录下执行git init，将hexo目录加入git管理

3. 执行 git remote add origin https://github.com/username/username.github.io.git

4. 执行 git checkout -b hexo-source，新建一个hexo-source分支

5. 执行 git add .

6. 执行 git commit -m "init"

7. 执行 git push -u origin hexo-source

8. 此时你仓库的远程分支会多出一个hexo-source分支，内容大致如下

   <img src="https://i.loli.net/2021/05/26/fAh6SFjsEI3LX9U.png" alt="image-20210526152959057" style="zoom:50%;" />

###### 配置TravisCI

这部分配置网上很多，可以参考以下博客：

https://segmentfault.com/a/1190000021987832

https://juejin.cn/post/6844903517853843470

主要内容是配置github与TravisCI的token秘钥之类的，让TravisCI能够有权限访问我们的github仓库，不多赘述。主要是大家的.travis.yml各不相同，下面贴出我的配置：

```yaml
os: linux
language: node_js 
node_js:
  - 10  # 使用 nodejs LTS v10
branches:
  only:
    - hexo-source # 只监控 hexo-source 的 branch
cache:
  directories:
    - node_modules # 缓存 node_modules 加快构建速度
before_script: ## 根据你所用的主题和自定义的不同，这里会有所不同
  - npm install -g hexo-cli # 在 CI 环境内安装 Hexo
  - npm install # 在根目录安装站点需要的依赖
script: 
  - hexo clean
  - hexo generate # generate static files
deploy: # 根据个人情况，这里会有所不同
  provider: pages
  skip_cleanup: true # 构建完成后不清除
  token: $GH_TOKEN # 你刚刚设置的 token
  keep_history: true # 保存历史
  #fqdn: blog.ne0ng.page # 自定义域名，使用 username.github.io 可删除
  on:
    branch: hexo-source # hexo 源文件所在的 branch
  local_dir: public 
  target_branch: master # 存放生成站点文件的 branch，使用 username.github.io 必须是 master
```

注意.travis.yml配置是写在hexo-source分支的， 写完后git push到远程分支，正常的话就会触发TravisCI构建了。

Tips：

- git push触发TravisCI构建前，TravisCi会发送一封确认邮件到github绑定的邮箱，不在邮件点确认的话也是不会触发构建的，有时候会很久都接受不到邮件，貌似是TravisCI的消息队列的问题，只能慢慢等了。

<img src="https://i.loli.net/2021/05/26/JdDbcA76zhuSV2G.png" alt="image-20210526154123366" style="zoom:50%;" />

- 构建会有延迟，可以在TravisCI中看构建过程



