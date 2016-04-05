---
layout: post
comments: true
title: "Github博客搭建"
description: "Github博客搭建"
category: jekyll
tags: [Jekyll]
---


运行环境：Windows 8	

准备工作：  
   1.搭建git 环境  
   2.安装RubyInstaller  
   3.DevKit  
   4.安装jekyll  
   5.安装python  
   6.安装python setuptools，配置Scripts环境变量  
   7.安装pygments  
   8.本地调试  
   9.本地博客与github同步

详细步骤：  
1.搭建git环境  
git下载地址：`http://msysgit.github.io/`  
运行可执行程序Git-1.9.4-preview20140929.exe，安装完成后click git bash即可使用。

2.安装RubyInstaller  
下载地址：`http://rubyinstaller.org/`  
根据运行环境选择合适的版本下载，我下载的是rubyinstaller-2.1.3-x64.exe。运行安装，安装过程中勾选添加环境变量	

3.安装DevKit  
下载地址：`http://rubyinstaller.org/downloads/`  
选择与  RubyInstaller版本和操作系统版本 相匹配的版本号下载，我下载的是DevKit-mingw64-64-4.7.2-20130224-1432-sfx.exe  
解压已下载的文件。  
notice：RubyInstaller和DevKit版本一定不要弄错，不然后面jekyll没办法安装  

4.安装jekyll  
启动cmd命令行  
首先，执行如下指令，完成ruby的安装：  

	cd C:\DevKit  
	ruby dk.rb init  
	ruby dk.rb install

然后，通过rubygem安装jekyll：  
由于国内网络原因，默认源很慢，往往安装不成功  

	gem sources --remove https://rubygems.org/	删除默认源
	gem sources -a http://ruby.taobao.org/		添加新源
	gem sources list

执行后，将看到输出为`http://ruby.taobao.org/`  

安装jekyll:

	gem install jekyll  安装jekyll
	jekyll -v	查看是否安装成功

安装rdiscount:

	gem install rdiscount  用于解析Markdown标记的解析包

之所以安装python 因为通过jekyll创建的项目可能会出现Pygments高亮等一系列效果，不安装python将不能正确部署网站  

5.安装python  
Python下载地址：`https://www.python.org/download/	`  
我安装的是python-2.7.8.amd64.exe，双击进行安装，安装完成后将Python安装路径加到环境变量Path  
通过`python -v`指令查看安装版本号，确认安装成功			

6.安装python setuptools  
打开网址 `https://pypi.python.org/pypi/setuptools`，根据Installation Instructions 先下载ez_setup.py，然后cmd命令行或双击执行该脚本	`ez_setup.py`完成setuptools安装，安装完成后python路径下Scripts中将含有easy_install，然后将在python路径下生成的Scripts添加到Path环境变量。		

7.安装pygments  
在命令行中执行`easy_install pygments` 完成pygments的安装。  
然后在jekyll项目配置文件_config.yml中设置`pygments:true`		

8.本地调试  
执行`jekyll new blog` 生成一个默认jekyll项目  
执行`jekyll build --trace`将所有文章根据模板进行编译，结果在_sites文件夹下  
执行指令`jekyll serve`将本地开启一个监听端口4000的服务器，浏览器中查看localhost:4000就能看到博文了。			

9.本地博客与github同步
在github上新建名为`xxx.github.io`的repository，此处是master分支  
打开Git Bash命令行，`cd github`进入本地check路径下，执行指令

	git clone https://github.com/xxx/xxx.github.io.git		
将远程master分支clone到本地。  
然后将jekyll项目所有文件复制到本地的Git项目	`*/github/xxx.github.io`  
`cd xxx.github.io`切到Git项目			
执行如下指令，完成本地修改push到github上：

	github add .		
	git commit -m "the first commit"	
	git push -u origin master		
 
等大概10分钟左右，再访问`https://xxx.github.io/`将看到博客内容		

参考文献  
http://jekyllcn.com/ jekyll学习网站  
https://help.github.com/categories/github-pages-basics/  
http://segmentfault.com/blog/skyinlayer/1190000000406011  
http://www.pchou.info/web-build/2014/07/04/build-github-blog-page-08.html		

TODO：  
jekyll语法，样式研究。  
域名绑定研究  

