---
layout: post
title: "用Octopress在GitHub上搭建博客"
date: 2013-09-01 13:23
comments: true
categories: [octopress]
keywords: octopress, setup, 配置, 博客
description: octopress配置
#published: false
---

备料
===============================================================================

注册 [GitHub](http://github.com) 账户并创建一个空仓库
-------------------------------------------------------------------------------
* (**假定注册名为 yourname, 注册邮箱 yourname@gmail.com**, 下同)
* 创建空仓库 yourname.github.io

下载并配置 [Git](http://git-scm.com/)
-------------------------------------------------------------------------------

### 下载
* Linux
  * 使用系统包管理安装git

* Windows
  * [MsysGit](http://code.google.com/p/msysgit/downloads/list)
  * vim着色：MsysGit上的vim着色文件不全, 可从完整vim73的syntax目录拷过来, 如 `/usr/share/vim/vim73/syntax`

### 配置
设置 LANG 环境变量(可以不设LC_ALL; 可以不是zh_CN, 后缀是UTF-8即可;)

```bash
# locale和中文显示
touch ~/.bashrc
echo 'export LANG="en_US.UTF-8"' >> ~/.bashrc
touch ~/.inputrc
echo 'set meta-flag on' >> ~/.inputrc
echo 'set convert-meta off' >> ~/.inputrc
echo 'set input-meta on' >> ~/.inputrc
echo 'set output-meta on' >> ~/.inputrc
touch ~/.vimrc
echo 'set fileencodings=utf-8' >> ~/.vimrc
# git options
git config --global user.name "yourname"
git config --global user.email "yourname@gmail.com"
git config core.autocrlf false
git config credential.helper 'cache --timeout=3600' # Keep your password cached in memory
git config github.user "yourname"
# ssh key
ssh-keygen -t rsa -C "yourname@gmail.com"
cat ~/.ssh/id_rsa.pub # 将内容复制到 https://github.com/settings/ssh 上
```

<!--more-->

下载并配置 Ruby(1.9.3)
-------------------------------------------------------------------------------

### 下载
* [Windows](http://rubyinstaller.org/downloads)
  * 7zip包解压(假设到 D:\ruby1.9.3), 添加到系统PATH
  * 下载并配置 DevKit([wiki](http://github.com/oneclick/rubyinstaller/wiki/Development-Kit))
    * 1.9.3 配对 [DevKit-tdm](http://github.com/downloads/oneclick/rubyinstaller/DevKit-tdm-32-4.5.2-20111229-1559-sfx.exe), 解压(假设到 D:\ruby1.9.3-DevKit), 添加到系统PATH

```bash
cd /D/ruby1.9.3-DevKit
ruby dk.rb init
ruby dk.rb review # 确认ruby位置正确
ruby dk.rb install
# 验证DevKit
gem install json --platform=ruby # 能看到 build native
ruby -rubygems -e "require 'json'; puts JSON.load('[42]').inspect" # 确认json gem安装成功
```

* Linux
  * 使用 [RVM(Ruby版本管理)](http://rvm.io/rvm/install)

```bash
# install RVM stable with ruby in user's $HOME
\curl -L http://get.rvm.io | bash -s stable --ruby # 反斜杠是防止使用到 ~/.curlrc 定义的 alias
# rvm安装完毕
# rvm list known
rvm install 1.9.3
rvm use 1.9.3 --default
# ruby -v
```

### 配置

```bash
# gem更新源
gem sources --remove http://rubygems.org/ # 要包含最后的斜杠
gem sources -a http://ruby.taobao.org/
gem sources -l # 验证源只有 ruby.taobao.org
```

安装Octopress
===============================================================================

```bash
git clone git://github.com/imathis/octopress.git mygithubio
cd mygithubio
gem install bundler # 不是 bundle
bundle install # 下载安装依赖项目(bundle是ruby的依赖管理工具)
rake install # 编译octopress项目(Rake to Ruby == Make to C)
```

```bash
rake setup_github_pages
```
* `hellip; 不是内部命令`错误 [^1]
  * `Rakefile`文件 `My Octopress Page is coming soon &hellip;` 在 `&hellip;` 前加 `^` (Windows cmd转义)

* setup_github_pages目标主要做了2件事:
  * 将原来git upstream的 origin 改到 octopress
  * 将你在 GitHub 上的博客地址(如 yourname.github.io) 作为 origin. 验证: `git remote -v`

```bash
# rake new_post['hello octopress'] # 创建新markdown博文
rake generate # 确保 `.gitignore` 包含忽略 _deploy 目录
# rake preview # 可通过本机4000端口预览
# rake deploy # push 到 GitHub 博客项目的 master 分支
```

基本使用和配置
===============================================================================

使用
-------------------------------------------------------------------------------
* `rake new_post['new-post-today']` 生成新博文
* `rake new_page['new-page-in-here']` 生成新页面(不属于博文系列)

配置[^2]
-------------------------------------------------------------------------------

### _config.yml
```yaml
# 博客链接格式
permalink: /blog/:year/:month-:day-:title.html
# 使用kramdown
markdown: kramdown
# SEO
description: yourname的技术博客
```

### 侧栏
* about me
  * `_config.yml` 的 `default_asides` 里添加 `custom/asides/about.html`
  * `touch source/_includes/custom/asides/about.html`, 添加内容
{% raw %}
```html
<section>
	<h1>About Me</h1>
	<p>一句话介绍</p>
	<p>微博: <a href="http://weibo.com/yourname">@yourweiboname</a><br/>
	<p>豆瓣: <a href="http://douban.com/yourname">@yourdoubanname</a><br/>
	</p>
</section>
```
{% endraw %}

### Header
* about页面

```bash
rake new_page['about'] # 生成 source/about/index.markdown
```

头部导航菜单 `/source/_includes/custom/navigation.html` 加入 about页面 链接.

### 字体
* `source/_includes/custom/head.html` 全部注释掉, 不装载 Google Webfonts(此字体没有包含中文, 粗体中文显示不出)
* `sass/custom/_fonts.scss` 添加([最佳 Web 中文默认字体](http://lifesinger.wordpress.com/2011/04/06/best-web-default-fonts/))

```css
$heading-font-family: arial, sans-serif;
$header-title-font-family: arial, sans-serif;
$header-subtitle-font-family: arial, sans-serif;
```

社交功能
===============================================================================

分享功能
-------------------------------------------------------------------------------

### [JiaThis](http://www.jiathis.com)[^3]
`_config.yml` 加入变量

```yaml
# JiaThis
jiathis: true
```

`source/_includes/post/sharing.html` 尾部 `</div>` 之前添加

```yaml
{% if site.jiathis %}
  {% include post/jiathis.html %}
{% endif %}
```

`touch source/_includes/post/jiathis.html`, 将从 JiaThis 获得的代码放入其中

评论功能
-------------------------------------------------------------------------------

### [多说](http://duoshuo.com)[^4]
`_config.yml` 加入变量

```yaml
# DuoShuo comments
duoshuo_comments: true
duoshuo_short_name: yourname
```

`source/_layouts/post.html` 中 disqus 代码下添加 (单独页面也加评论的话 `source/_layouts/page.html` 中也放相同代码)

{% raw %}
```html
{% if site.duoshuo_short_name and site.duoshuo_comments == true and page.comments == true %}
  <section>
    <h1>Comments</h1>
    <div id="Comments" aria-live="polite">{% include post/duoshuo-thread.html %}</div>
  </section>
{% endif %}
```
{% endraw %}

创建 `source/_includes/post/duoshuo-thread.html`, 将从多说获得的代码放入其中

换主题
===============================================================================
[bootstrap-theme](http://github.com/bkutil/bootstrap-theme) , 或者[其他的](http://github.com/imathis/octopress/wiki/3rd-Party-Octopress-Themes)

```bash
git clone http://github.com/bkutil/bootstrap-theme.git .themes/bootstrap-theme
rake install['bootstrap-theme'] # 注意: 换主题后所有非custom目录下的内容都会被覆盖掉！！
rake generate
```

i18n
===============================================================================
I forked from [hendricius/jekyll-i18n](https://github.com/hendricius/jekyll-i18n.git) and adapted to octopress(in branch [octopress-i18n](http://github.com/wangmuy/octopress-i18n.git)).

However, AFAIK there's no i18n capable themes right now. You have to create your own theme branch and adapt to i18n.

_______________________________________________________________________________
[^1]: [Octopress 笔记](http://netwjx.github.io/blog/2012/03/18/octopress-note/)
[^2]: [我的Octopress配置](http://www.yanjiuyanjiu.com/blog/20130402/)
[^3]: [在 Windows7 下从头开始安装部署 Octopress](http://sinosmond.github.io/blog/2012/03/12/install-and-deploy-octopress-to-github-on-windows7-from-scratch/)
[^4]: [为 Octopress 添加多说评论系统](http://ihavanna.org/internet/2013-02/add-duoshuo-commemt-system-into-octopress.html)