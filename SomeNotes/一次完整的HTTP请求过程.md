### HTTP 通信过程
HTTP 通信机制是在一次完整的HTTP通信过程，web浏览器与服务器将完成以下7个过程

1. 域名解析
2. 发起TCP的3次握手
3. 建立起TCP连接发起http请求
4. 服务器响应http请求，浏览器得到HTML代码
5. 浏览器解析HTML代码，并请求HTML代码中的资源
6. 浏览器对页面进行渲染呈现给用户

#### 域名解析
将域名转为IP地址 浏览器缓存DSN  运营商DSN HOST文件

#### 发起TCP的3次握手

#### 发起HTTP请求
请求行   
HTTP头：HTTP头在HTTP请求可以是3种HTTP头：1.请求头（request header）2.普通头（genneral header）3.实体头（entity header）   

内容：只在post请求中， 因为GET请求中不包含任何实体

#### 服务器端HTTP响应请求
当Web服务器收到HTTP请求后，会根据请求的信息做某些处理，相应的返回一个HTTP响应。HTTP响应在结构上很类似于HTTP请求，由三部分组成：
1. 状态行 HTTP/1.1 200 OK  第一部分是HTTP版本， 第二部分是响应状态码，  第三部分是状态码的描述  
信息类（100 - 199）  
响应成功（200 - 299）  
重定向类（300 - 399）  
客户端错误（400 - 499）  
服务端错误（500 - 599）  

2. HTTP头：响应头中包含头包括 响应头 普通头 实体头 
3. 返回内容 HTTP响应内容就是HTTP请求所请求的信息。这个信息可以是一个HTML，也可以是一个图片。响应的数据格式是通过content-Type字段来获得。   
    text/plain  
    text/html   
    text/css  
    image/jpeg  
    image/png  
    image/svg+xml  
    audio/mp4  
    video/mp4  
    application/javascript  
    application/pdf  
    application/zip  
    application/atom+xml  
4. 浏览器解析HTML代码，并请求HTML代码中的资源   
有时候我们获取一个HTML页面，在对浏览器对HTML解析的过程中，如果发现额外的URL需要获取的内容，会再次发起HTTP请求去服务器获取，比如样式文件，图片。许多个HTTP请求，只依靠一个TCP连接就够了，这就是所谓的持久连接。也是所谓的一次HTTP请求完成。

### HTTP和HTTPS的基本概念
HTTPS： 以安全为目标的HTTP通道，简单讲是HTTP的安全版，即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就是需要SSL。   
HTTPS的协议主要作用分为两种：一是建立一个信息安全通道，来保证数据传输的安全，另外一种是确认网站的真实性。  

### HTTP与HTTPS区别
1. HTTPS是需要CA申请证书。
2. HTTP是超文本传输协议，信息是明文传输， HTTPS是具有安全性的SSL加密传输协议  
3. HTTP与HTTPS是完全不同的连接方式， 用的端口不一样，前者是80，后者是443.

### HTTPS的工作原理
![++ 工作原理++](http://www.mahaixiang.cn/uploads/allimg/1507/1-150H120343I41.jpg)

客户端在使用HTTPS方式与Web服务器通信时有以下几个步骤：   
1. 客户端使用HTTPS的URL访问Web服务器，要求与Web服务器建立SSL连接。
2. Web服务器收到客户端请求后， 会将网站的证书信息（证书中包含公钥）传送一份给客户端。  
3. 客户端的浏览器与Web服务器开始协商SSL连接的安全等级，信息加密的等级。 
4. 客户端的浏览器根据双方同意的安全等级，建立会话密钥，然后利用网站的公钥将会话密钥加密，并传送给网站  
5. Web服务器利用自己的私钥解密出会话密钥  
6. Web服务器利用会话密钥加密与客户端的通信

![++加密过程++](https://pic002.cnblogs.com/images/2012/339704/2012071410212142.gif)
