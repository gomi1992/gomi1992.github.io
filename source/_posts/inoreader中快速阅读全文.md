---
title: inoreader中快速阅读全文
comments: true
copyright: true
date: 2022-07-24 08:19:14
categories:
tags:
---

# 过时的RSS
现在用RSS追资讯的人可能不多了。
RSS没有相似推荐，没有社交，仅仅只有非常单纯的信息流。
随着google reader的关闭，我现在用的最多的阅读器就是[inoreader](https://www.inoreader.com/)了。

# 过时的订阅源
现在新兴的媒体基本都不提供RSS订阅方式，毕竟用RSS没法看到他们的广告，也没法去产生各种联动，提高他们的效益。
于是诞生了[RSSHub](https://github.com/DIYgod/RSSHub)这样的项目，让我们能够用RSS跟上时代。

# inoreader的痛
inoreader免费版对于我已经够用了，但是仍存在一点点小问题。
比如我没办法在inoreader中阅读cnbeta的全文，必须得离开rss阅读器。
于是我开始寻求解决方案，比如寻找其他的阅读器作为替代。
但成效甚微，不是不支持阅读区内全文阅读，就是限制太多（需要付费）。

# 柳暗花明
放弃寻找阅读器替换这条路后，我试着寻找油猴脚本来辅助，也确实找到了一个比较适合我的脚本[InoReader Full Feed](https://greasyfork.org/zh-CN/scripts/897-inoreader-full-feed)。
为什么说比较适合呢？因为我的需求很简单，只需要能够以iframe的方式将目标网页嵌入阅读器就可以，而这款脚本的功能超乎了我的想象。
于是我也毫不留情，开始简单粗暴的改造。

在其代码1.06版本上，第1716行(initFullFeed函数中)后插入如下代码。
``` javascript
    for(var i=0,j=c.articleContainer.children.length;i<j;i++){
        c.articleContainer.removeChild(c.articleContainer.children[0]);
    }
    var p = document.createElement("iframe");
    p.setAttribute("src",c.itemURL);
    p.style.width="100%";
    p.style.height="1000px";
    c.articleContainer.appendChild(p);
    return;
````
插入后如下图

![插入后效果](https://raw.githubusercontent.com/gomi1992/blog_images/main/img/202207240844928.png)

这段代码先删除文章内的已有元素，然后插入一个iframe元素到文章界面中，并频闭掉剩下的代码。

# 不完整的美
出于安全性考虑，浏览器禁止向iframe中传入cookie，这就导致一些小功能不好实现。