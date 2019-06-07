---
layout: post
title: JS学习笔记——跨域
description: 跨域
date: 2017-04-17
---

现在的web应用越来越丰富，一个web上的内容往往会抓取其他web上的数据。在默认情况下，web上的交互需要遵循同源策略（Same-origin policy），即同协议、同域名、同端口。在URL`http://store.company.com/dir2/other.html`中，协议是Http、域名是store.company.com，端口是80。当不符合同源策略时，这时的通信就可以叫做跨域通信。跨域通信有许多奇奇怪怪的方法可以做到，这里就简单介绍几种。

### document.domain + iframe
`http://www.a.com`和`http://script.a.com`的交互是跨域交互，因为两者域名不同，虽然主域名都为`a.com`。
对于主域相同而子域不同的例子，可以通过设置document.domain的办法来解决。具体的做法是可以在http://www.a.com/a.html和http://script.a.com/b.html两个文件中分别加上`document.domain = 'a.com'`；然后通过在a.html文件中创建一个`<iframe>`，去控制iframe的contentDocument，这样两个js文件就可以跨域通信了。当然这种办法只能解决主域相同而二级域名不同的情况。
```javascript
//www.a.com/a.html
document.domain = 'a.com';
var ifr = document.createElement('iframe');
ifr.src = 'http://script.a.com/b.html';
ifr.style.display = 'none';
document.body.appendChild(ifr);
ifr.onload = function(){
    var doc = ifr.contentDocument || ifr.contentWindow.document;
    // 在这里操纵script.a.com/b.html，可以对doc进行各种DOM操作
    alert(doc.getElementsByTagName("h1")[0].childNodes[0].nodeValue);
};
```

```javascript
//script.a.com/b.html
document.domain = 'a.com';
```


### JSONP
JSONP(JSON with padding)利用`<script>`跨域加载脚本的原生能力来做到跨域通信。

我们知道，如果要引用JQuery库，除了下载到本地引入外，还可以这么引入：`<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.2.1/jquery.min.js"></script>`，这个就是JSONP的基本原理。JSONP是一种非正式的传输协议，可以说是劳动人民的智慧。在JSONP的跨域通信中，客户端和服务器都需要遵循一定的规则。

`<script>`的src属性应该填入所需服务的URL，URL中有查询字符串`code=CA998&jsoncallback=callbackFunction`，`code=CA998`是传送给服务器的数据，`jsoncallback=callbackFunction`是传送给服务器的回调函数。前者jsoncallback由服务器定义，可能是callback、callbackFunc等等，双方约定好即可。后者callbackFunction由客户端定义，这是一个回调函数。将这样一个查询字符串发送给服务器之后，服务器会做两件事：1.根据查询字符串中的数据，准备好客户端需要的结果数据；2.将数据传入回调函数，把执行函数的字符串返回给客户端。客户端加载入这样一个js文件后，会执行该js文件中的执行语句。

服务器返回了带有执行语句的js，在客户端这边，需要定义或者声明这个回调函数`callbackFunction()`。
```javascript
//回调函数
function callbackFunction(data)
{
    console.log(data);
}

var url = "http://www.runoob.com/try/ajax/jsonp.php?jsoncallback=callbackFunction";
// 创建script标签，设置其属性
var script = document.createElement('script');
script.setAttribute('src', url);
// 把script标签加入head，此时调用开始
document.getElementsByTagName('head')[0].appendChild(script); 
```

```javascript
//服务器根据查询字符串的请求，用服务器端的语言，生成并返回js文件，该js文件大概内容如下
callbackFunction(["customername1", "customername2"]);
```

### HTML5 postMessage
HTML5为window对象新增了一个postMessage方法，该方法用于解决跨域通信的问题。
在`http://test.com`下，向`http://lslib.com`发送信息的步骤：
1.在发送方建立`<iframe>`，`src`属性填写接收方的url
2.调用frame的postMessage()方法，该方法接收两个参数：
message：只支持字符串信息，若要发送对象，可用JSON.stringify转换成字符串；
targetOrigin：目标域
3.在接收端为window对象绑定message事件，MessageEvent对象有三个重要的属性：
data：获取字符串
source：发送消息的窗口对象
origin：发送消息的源
```html
http://test.com/index.html

<div>
    <iframe src="http://lsLib.com/lsLib.html"></iframe>
</div>

<script type="text/javascript">
    window.onload=function(){
        window.frames[0].postMessage('getcolor','http://lslib.com');
    }
</script>
```

```html
http://lslib.com/lslib.html

<script type="text/javascript">
    window.addEventListener('message',function(e){
        //e.data = "getcolor"
    },false);
</script>
```

### 图像ping
一个网页可以从其他任何网页中加载图，那么图像ping主要通过`<img>`标签的src属性来进行跨域。客户端的请求通过查询查询字符串发送给服务器。这种方式主要有两点不足：1.只能用Get请求；2.只能进行客户端至服务器的单向通信（通过查询字符串），无法访问服务器的响应文本。
```
var img = new Image();
img.onload = function() {
    alert("onload!");
}
img.onerror = function() {
    alert("onerror!");
}
//img.src="http://xxx.jpg";
img.src="http://www.example.com/test?name=pxz";//name=pxz是就是客户端发送给服务器的请求
```

【Reference】
1. <a href="https://earthsplitter.github.io/2017/03/21/%E5%90%8C%E6%BA%90%E6%94%BF%E7%AD%96%E4%B8%8E%E8%B7%A8%E5%9F%9F%E8%AF%A6%E8%A7%A3/"> 同源政策与跨域详解 </a>
2. 说说JSON和JSONP，也许你会豁然开朗，含jQuery用例  http://www.cnblogs.com/dowinning/archive/2012/04/19/json-jsonp-jquery.html
3. http://www.cnblogs.com/rainman/archive/2011/02/20/1959325.html#m3
