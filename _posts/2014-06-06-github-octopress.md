---
layout: post
title: "用Octopress在Github搭建个人博客"
date: 2014-06-06 20:38:02 +0800
comments: true
categories: blog
tags: [能工巧匠]
---

一、安装Octopress运行的必要软件

<!-- more -->

1.Octopress官网及软件下载：

* 官方首页：[http://octopress.org](http://octopress.org "http://octopress.org")

* 这里是RubyInstaller[下载地址](http://pan.baidu.com/s/1eQotA0a) （码：aurm）。

* 这里是DevKit[下载地址](http://pan.baidu.com/s/1hq1gIUS) （码：vkd5）。

2.安装RubyInstaller

3.安装DevKit

4.启动Ruby命令框，用CD的命令进入你存放DevKit的目录中，执行以下命令继续安装
	
`ruby dk.rb init`

`ruby dk.rb install`

二、在本地安装Octopress博客系统

1.要安装Octopress，就得先改变一个软件更新的源，因为默认的官方下载源已经被Q了。执行以下命令。

```bash
gem sources -a http://ruby.taobao.org/
gem sources -r http://rubygems.org/
gem sources –l
```

2.然后执行：vi Gemfile 编辑配置文件，你也可以直接使用文本编辑器打开Gemfile，将第一行的source改成国内淘宝的。

3.依次进入你存放博客的目录中，安装bundler。

`gem install bundler`

`bundle install`

4.再安装Octopress默认的主题。
		
`rake install`

5.最后是生成和预览博客。

`rake generate`
`rake preview`

6.用的浏览器打开：localhost:4000，就可以看到Octopress博客效果了。

三、更改电脑环境变量让Octopress支持中文

1.我们需要改变一下我们计算机的环境变量，计算机–属性–高级系统设置–环境变量。

2.新增 LANG 和 LC_ALL ，值都是 zh_CN.UTF-8。

四、提交Octopress博客到Github免费空间

1.先进入你的Github的本地项目中。

2.连接Github服务器，填写你的Responsibility Url。

3.然后再执行生成和提交命令。

{% highlight bash %}
rake setup_github_pages
rake generate
rake deploy
{% endhighlight %} 

4.完成后，打开你的Github的二级域名后就可以看到刚刚提交的Octopress博客了。

5.还可以直接使用Git来提交你的Octopress博客

6.执行Octopress生成后，博客所有文件都存在一个Public的文件夹。你只要将这个Public中的文件复制或者直接上传到你的Github的空间也能实现浏览的效果。

五、Octopress博客发布文章和新建页面

1.发布一个文章前，先生成一个MD的文件，执行。

`rake new_post["hello world"]` 它会在项目/source/_posts/中生成一个MD文件，类似2014-06-01-hello-world.markdown这样的。
   
2.如果想要新建一个页面，则可以执行。

`rake new_page["about"]` Octopress需要使用markdown语法，并不是常用的HtmL，学习markdown：http://wowubuntu.com/markdown/
	
3.文章编辑完成后，就是生成和发布了。

`rake generate`
`rake deploy`

4.本地预览可以用以下命令。

`rake preview`

5.退出预览是：Ctrl+C
