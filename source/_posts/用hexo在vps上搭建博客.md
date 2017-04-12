---
title: 用hexo在vps上搭建博客
date: 2016-07-01 21:27:28
tags: 那些年的坑
categories: [乱七八糟]
---

在搬瓦工买了个vps用来搭梯子，但是似乎仅仅弄个梯子有点浪费资源了，于是就打算搭个博客玩玩，记录下学习新的啥的。网上这方面的教程贼多，看了一下各家方案，决定采用hexo进行博客的搭建，然后利用
git hooks自动部署到vps上。网上教程千千万，结果是一个比一个比，看得我是头皮发麻，感觉全是从哪里
copy过来的，具体能不能他们就不管了，就是那么任性。反正大体上的思路还是知道的：*先用hexo生成静态网页，
然后通过hexo的deploy部署到远程仓库，然后设置git hooks将博客自动部署到服务器上*，出了问题就找
谷歌爸爸咯0.0 于是抱着这个想法，没两小时博客就搭好跑起来了，我还能说啥，只能记录一波这个坑
<!--more-->

#### 本地安装hexo
推荐去官网下载最新本的Node.js source code进行编译安装（需要安装cmake和g++）
- 下载nodejs包：     `wget  https://nodejs.org/dist/v6.10.1/node-v6.10.1.tar.gz`
- 解压： `tar zxvf node-v6.10.1.tar.gz`
- 构建, 部署： 先configure `cd node-v6.10.1.tar.gz && sudo ./configure`然后再install
   `sudo make install`
- 安装好了nodejs默认会自带npm，不过有可能不是最新版的(`npm -v`)，可以更新到最新版`npm install npm -g`
- 安装hexo： `npm install hexo-cli -g`
- 初始化hexo: `mkdir hexo && cd hexo && hexo init`这样便完成了初始化工作
- 测试： `hexo new post "hello-world"`然后`hexo g && hexo s`,运行完命令之后便可以在本地4000端口访问刚刚的hello world页面了
- 测试没问题则对_config.yml进行配置，增加或者修改deploy条目为
```
deploy:
  type: git
  repo: git@remote_host:/home/hexo.git
  branch: master
```

#### vps添加git用户
- `user add git`
- `sudo vim /etc/sudoers`并且在root的配置处增加一行`git   ALL=(ALL：ALL)     ALL`（为git用户增加sudo权限）

#### 生成公钥并上传到服务器
- `ssh-keygen -t rsa -C 4096`
-  `ssh-copy-id -i ~/.ssh/id_rsa.pub -p port git@remote_host` port是你ssh访问服务器的端口号，一般来说都不是22所以得指明一下

#### 建立远端仓库并且配置git hooks
- `mkdir hexo && cd hexo`
- `git init --bare`
- `cd hooks && vim post-receive`写入内容如下
```
#！ /bin/bash
git --work-tree=/var/www/hexo --git-dir=/home/git/hexo.git checkout -f
```
- `chmod +x post-receive`
- `cd ~ && chown git:git hexo.git`

#### vps安装配置运行博客的服务器(我用的是nginx)
- `sudo apt-get install nginx`
- `cp /etc/nginx/sites-available/default /etc/nginx/sites-available/hexo`复制一份默认的配置文件，然后进行修改。主要是两个条目一个是root表示你博客部署所在地址，一个是server_name域名配置，也可以不加这个。以我的为例，内容如下：
```
server {
       	listen 80 default_server;
       	listen [::]:80 default_server;
       	root /var/www/hexo;
       	# Add index.php to the list if you are using PHP
       	index index.html index.htm index.nginx-debian.html;
       	server_name jianghuiqiang.com;
       	location / {
       		# First attempt to serve request as file, then
       		# as directory, then fall back to displaying a 404.
       		try_files $uri $uri/ =404;
       	}
}
```

#### 测试
以上步骤完成之后，基本上就完工了，在本地hexo项目运行`hexo g && hexo d`若提示访问端口22被远端服务器拒绝访问，则可以在~/.ssh文件夹下面增加一个config文件，以我的为例，内容如下：
```
Host remote_host（这里是vps ip）
User git
PreferredAuthentications publickey
IdentityFile  ~/.ssh/id_rsa
Port  port（ssh访问vps的端口）
```
配置好了之后基本上就没啥问题了，直接在浏览器输入你的vps ip或者是域名就可以看到你的hello world测试页面了
