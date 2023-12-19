---
title: 我的个人计划表
date: 2022-11-07 09:47:08
categories:
   - 无聊的开发
tags: 
   - 持续部署
   - 个人网页
TOC: true
---

## 起因
想把自己的工作计划在网页里展示出来，方便监督自己。

## 想法
plantuml、mermaid、excalidraw都是不错的工具。

plantuml和mermaid最主要的是展示甘特图，最终选择mermaid，原因是界面稍微漂亮一些，而plantuml渲染需要外部服务器。

excalidraw是个非常自由的白板，本打算使用多人协作方式集成它，这样就能多端实时协作，但因为跨域的安全性问题无法集成到网页中。

有了想法就开工。

## 集成mermaid
mermaid提供js脚本，可以直接集成到网页中。

``` html
<div id="mermaid-gantt" class="mermaid">
gantt
    dateFormat  YYYY-MM-DD
    axisFormat  %y/%d/%m
    title       Schedule
    excludes    weekdays
    
    section 个人安排网页<br>开发
    集成甘特图                      : done, 2022-11-02,3h
</div>
<script src="js/mermaid.min.js"></script>
<script>
    window.onload = function () {
        mermaid.initialize({
            theme: 'forest',
            // themeCSS: '.node rect { fill: red; }',
            logLevel: 3,
            securityLevel: 'loose',
            flowchart: {curve: 'basis'},
            gantt: {axisFormat: '%m/%d/%Y'},
            sequence: {actorMargin: 50},
            // sequenceDiagram: { actorMargin: 300 } // deprecated
        });
    }
</script>
```

## 集成excalidraw
### 方案1--分享链接集成
这个方案很简单，效果也很好，将excalidraw的分享链接塞进iframe里。
每回修改白板后就把新的链接更新到代码中即可。

最后并没有采用这个方案，原因在于数据不在自己手上。

**这个方案并没有真正测试过，可能也会出现方案2的问题。**

### 方案2--实时写作集成
集成失败，由于部署环境没有部署https，所以静态页的安全上下文（[secure context](https://w3c.github.io/webappsec-secure-contexts/)）为不安全。
因此，嵌在静态页中的iframe的安全上下文也为不安全。

这个安全上下文不安全导致excalidraw的实时写作无法使用加密信道。

### 方案3--使用静态数据由kroki渲染
[kroki](https://kroki.io/)提供一堆各种图的在线渲染，将数据给他，他给你一个svg，就是这么简单，但是数据存在泄露的风险。

**有严重缺陷，kroki仅支持4096字节长度的数据渲染，一旦图大了也就无法渲染。**

``` javascript
function textEncode(str) {
    if (window.TextEncoder) {
        return new TextEncoder('utf-8').encode(str);
    }
    var utf8 = unescape(encodeURIComponent(str));
    var result = new Uint8Array(utf8.length);
    for (var i = 0; i < utf8.length; i++) {
        result[i] = utf8.charCodeAt(i);
    }
    return result;
}

// 数据编码
var data = textEncode("your data here");
// 数据压缩
var compressed = pako.deflate(data, {level: 9, to: 'string'})
var result = btoa(compressed)
    .replace(/\+/g, '-').replace(/\//g, '_');
var kroki_url = 'https://kroki.io/excalidraw/svg/' + result;

// 请求svg
request.open('GET', kroki_url, false);
request.send(null);
if (request.status === 200) {
	// 向div中插入svg
	document.getElementById("excalidraw").innerHTML = request.responseText;
}
```

### 方案4
将excalidraw转为SVG。

使用前端库[excalidraw/utils](https://www.npmjs.com/package/@excalidraw/utils)将excalidraw转换为svg，插入到网页中。

官方的示例代码有点问题，正确的如下。需要注意的地方是，不能直接传入string作为参数，需要将string转换成结构体传入（excalidraw数据本身就是json格式）。
```javascript
function loadExcalidraw() {
    var request = new XMLHttpRequest();
    request.open('GET', '/data/schedule.excalidraw', false);
    request.send(null);
    if (request.status === 200) {
        var data = eval("(" + request.responseText + ")");
        var svg = exportToSvg(data);
        svg.then((value) => {
            console.log(value);
            document.getElementById("excalidraw").innerHTML = value.outerHTML;
        })
    }
}
```

## 在线编辑
在github上打开项目，将com改为dev或者按一下“.”就可以进入GitHub.dev在线编辑环境，在其中安装excalidraw和mardown preview插件（网页端渲染mermaid还暂时不支持），就可以在线编辑excalidraw，还有预览mermaid图了。

## 持续部署
因为是静态网页，所谓的部署也就是把最新的网页拉取回来。
没有将静态页打包成docker镜像，若采用docker打包，则需要拉取 镜像--停止旧容器--运行新容器--删除旧镜像 这一系列操作；远不如本地部署nginx，更新对应的web资源目录来的方便。

触发使用github webhook触发，这里没有选择写个简单web服务通过内网穿透暴露到互联网上，主要担心内网穿透服务不稳定。

[ntfy](https://ntfy.sh/)是一个消息发布、订阅的工具，原本我是想用telegram来打通树莓派和github的，但是telegram不支持两个bot间通信。
在ntfy官方服务器中发布消息，你得先有一个主题，这个主题任何人都可以订阅、发布（没错，官方服务器就是这么设定的，想使用权限，你需要自己部署），所以起一个又长又难猜的主题吧。消息发布、和接受文档里都写得非常详细，不再赘述。

我这里使用python去轮询ntfy接口收取消息，并加上了访问频率限制，未对消息内容做检查，因为用不上，反正做的操作也是从github拉取项目。

注：github webhook支持内容验证，ntfy不能与之对接，得自己写web服务对接，所以这一点来说，ntfy有很大缺点。

``` python
import json
import os
import time

import requests as requests


class RateLimiter:
    def __init__(self):
        self.capacity = 4.0
        self.rate = 0.25
        self.tokens = 0.0
        self.timeStamp = time.time()

    def control(self):
        now = time.time()
        self.tokens = min(self.capacity, self.tokens + int(now - self.timeStamp) * self.rate)
        self.timeStamp = now
        if self.tokens < 1:
            return False
        else:
            self.tokens -= 1
            return True


rateLimiter = RateLimiter()
last_time = time.time()
while True:
    try:
        tmp_time = time.time()
        resp = requests.get(
            "https://ntfy.sh/yourtopic/json?poll=1&since="
            + str(int(last_time)), stream=False)
        last_time = tmp_time
        for line in resp.iter_lines():
            if line:
                print(line)
                data = json.loads(line)
                if data['event'] == 'message':
                    if data['message'] == 'Update finish':
                        continue
                    if rateLimiter.control():
                        print("get message")
                        os.system("git pull")
                        requests.post(
                            "https://ntfy.sh/yourtopic",
                            data="Update finish".encode(encoding='utf-8'))
        time.sleep(10)
    except Exception as e:
        print(e)
```

## 实现效果
![实现效果](https://raw.githubusercontent.com/gomi1992/blog_images/main/img/202211081623858.png)
