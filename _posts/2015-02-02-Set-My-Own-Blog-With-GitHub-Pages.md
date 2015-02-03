---
category: posts
date: 2015-02-02 17:15:16 UTC
title: 使用Github Pages + Jekyll搭建个人Blog 
tags: [Github Pages, Jekyll, RedCarpet]
---

  正式投身于Scala的开发已有1年由余，期间一直在不断学习新的知识Lift、Git、Mongo、Maven, Lucene......但是一直没有机会将自己所学所想写下来，遗憾没能抓住写博客总结这个机会来加深对于各个知识点的理解。终于在2015年开始行动起来, 开始了这个博客系统的搭建。

# 选择
#### CSDN  
  之前在CSDN上面写过一篇博客但是感觉对于代码的输出等做的不是太好，这样写的时候就会不太方便。

#### [作业部落](https://www.zybuluo.com)
  Cmd Markdown是一款基于Markdown的编辑器，对于熟悉Markdown语法的人来说是个福音，而且界面很简洁，功能也清晰。但是我在写这篇博客的时候，好像还没有提供很便捷的"分享"方式，**也就是没有一个前台能够很好的展现所有的分享的笔记**(只能通过链接分享的形式)。

#### [GitHub Pages](https://pages.github.com/) + [Jekyll](http://jekyllrb.com/)
  这个是基于Github Pages的博客系统，官方推荐使用Jekyll，它相当于是一个静态文本的转换器（遵循特定的格式，支持Markdown语法）。
  最后考虑到作业部落有一个<span class="highlight">导出为Markdown格式文件</span>的功能再加上它的易用性, 这样最终的组合为
      
  最后的方案, Cmd Markdown(博客内容编辑，导出) --》Jekyll的内容渲染 --》 Github Pages发布页面， 至于图片的存储选择了又拍云。 
    
# 搭建
  整个项目的搭建主要是集中在Jekyll系统的搭建，至于在Github上新建这个博客项目按照此页面所说的即可([GitHub Pages](https://pages.github.com/))。
  Jekyll官网对于整个配置感觉讲的不是很清晰，下面简单回顾一下，我自己搭建过程中总结的知识点。
  
  * 项目结构  

    在Github上创建了博客项目之后，最初就只有一个index.html。

    ![项目的最初结构](http://jacoffee-blog.b0.upaiyun.com/2015-02-03/Blog%20Directroy%20Structure.png)
    
    其它几个文件夹(<b style="color:red">都需要你自己手动创建的</b>)
    _layouts是放你的网站的基本结构页面, 比如说header.html, footer.html, default.html等
    _posts是放你的博客的，支持.md 以及.textile
    _site是Jekyll将你的博客以及layouts中的html文件渲染成的站点文件
    static是放静态文件的地方，比如说css，image什么的

  * 语法高亮  

    Jekyll默认采用的 Liquid templating language来处理模板，比说高亮代码语法

    ![高亮Ruby](http://jacoffee-blog.b0.upaiyun.com/2015-02-03/Ruby%20Highlight.png)

    但是这种写法比较繁琐，这时我们需要引入[redcarpet](https://rubygems.org/gems/redcarpet), 它支持下面的写法

    ![Redcarpet高亮Ruby](http://jacoffee-blog.b0.upaiyun.com/2015-02-03/Ruby%20Highlight%20Redcarpet.png)

    虽然Jekyll最新版(2.0)已经支持redcarpet2了, 但是我在搭建的过程中还是需要在_config.yml中配置如下的参数，否则```式高亮无法生效
    <div class="highlight">
       markdown: redcarpet
    </div>
    不过默认的高亮的颜色不是太好看, 并且没有行号，所以参考[这篇博客](http://blog.leonardfactory.com/2013/05/05/code-fenced-blocks-pygments-and-line-numbers-with-jekyll/)解决了这个问题。
    这样一来，基本博客的书写就没有什么问题了。

    ```scala
      def printlnyou = {
        println("xxxxx")
      }
    ```

# 计划
  * 完善网站的基本结构,包括网站域名的优化，顶部导航，底部的网站声明以及侧边栏的一些功能
  * 添加基本元素的样式,包括"p"元素的缩进，一些特殊字符的高亮
  * 增加博客文章评论框
  * 逐步最好网站的SEO
  