# Web 攻击
## 主动攻击
攻击者直接访问 Web 应用，传入攻击代码。目的是对服务器资源进行攻击。
+ SQL 注入攻击  
通过输入精心构造的数据，使得拼接 SQL 之后得到意想不到的效果

+ OS 命令注入攻击
通过输入精心构造的数据，使得服务器执行意想不到的脚本

## 被动攻击 & 跨域
### 跨域

浏览器的同源策略

最初的 “同源策略”，主要是限制 Cookie 的访问，A 网页设置的 Cookie，B 网页无法访问，除非 B 网页和 A 网页是“同源”的。所谓“同源”就是指"协议+域名+端口"三者相同，**即便两个不同的域名指向同一个ip地址，也非同源**。随着互联网的发展，"同源策略"越来越严格，不仅限于Cookie的读取。目前，如果非同源，共有三种行为受到限制。

+ Cookie、LocalStorage 和 IndexDB 无法读取。
+  DOM 无法获得。
+ 请求的响应被拦截

> 跨域这个情况只会出现在浏览器页面里，因为实际上是**浏览器由于安全原因限制了这些请求的访问**
>
> 比如 [http://a.com](http://link.zhihu.com/?target=http%3A//a.com) 里的代码要request [http://b.com](http://link.zhihu.com/?target=http%3A//b.com)，这时候 [http://b.com](http://link.zhihu.com/?target=http%3A//b.com) 的server不能简单地判断这个request是用户自己在browser地址栏里输入的，还是别的有恶意的网页利用用户在自己网页上点来点去就发 request 的。
>
> 所以browser在这种由[http://a.com](http://link.zhihu.com/?target=http%3A//a.com) request [http://b.com](http://link.zhihu.com/?target=http%3A//b.com)的情况，会强制增加CORS的各种header（所以不要用不安全的浏览器）。换句话说，这是client（广义的所有能发起http request的一方）的君子协定。如果用户自己下载不安全的浏览器，或者恶意app，这个是避免不了的。

虽然这些限制是必要的，但是有时很不方便，合理的用途也会受到影响，所以为了能够获取非“同源”的资源，就有了跨域资源共享。

> 当然了，有三个标签本身就是允许跨域加载资源的：
>
> ```html
> <img src=XXX>
> <link href=XXX>
> <script src=XXX>
> ```

#### 跨域资源共享 Cross Origin Resource Sharing

是一个W3C标准。**该方案需要服务器的支持。**

当一个资源(origin)通过脚本向另一个资源(host)发起请求，而被请求的资源(host)和请求源(origin)是不同的源时(协议、域名、端口不全部相同)，浏览器就会发起一个 **跨域 HTTP 请求** ，并且浏览器会自动将当前资源的域添加在请求头中一个叫 Origin 的 Header 中。

##### 简单请求

简单请求同时满足：

> （1) 请求方法是以下三种方法之一：
>
> - HEAD
> - GET
> - POST
>
> （2）**HTTP的头信息不超出以下几种字段**：（注意噢比如 `Authorization ` 字段就不行）
>
> - Accept
> - Accept-Language
> - Content-Language
> - Last-Event-ID
> - Content-Type：只限于三个值`application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain`（注意 `application/json` 不可以）

下面是一个例子，浏览器发现这次跨源 AJAX 请求是简单请求，就自动在头信息之中，添加一个`Origin`字段。

```http
GET /cors HTTP/1.1
Origin: http://api.bob.com
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```

上面的头信息中，`Origin`字段用来说明，本次请求来自哪个源（协议 + 域名 + 端口）。服务器根据这个值，决定是否同意这次请求。

+ 如果`Origin`指定的源，不在许可范围内，服务器会返回一个正常的HTTP回应。**浏览器发现**，这个回应的头信息没有包含`Access-Control-Allow-Origin`字段（详见下文），就知道出错了，从而抛出一个错误，被`XMLHttpRequest`的`onerror`回调函数捕获。注意，这种错误无法通过状态码识别，因为HTTP回应的状态码有可能是200。

+ 如果`Origin`指定的域名在许可范围内，服务器返回的响应，会多出几个头信息字段。

  ```http
  Access-Control-Allow-Origin: http://api.bob.com
  Access-Control-Allow-Credentials: true
  Access-Control-Expose-Headers: FooBar
  Content-Type: text/html; charset=utf-8
  ```

  + Access-Control-Allow-Origin：该字段是必须的。它的值要么是请求的 `Origin` 字段的值，要么是一个`*`，表示接受任意域名的请求。
  + 该字段可选。CORS请求时，`XMLHttpRequest`对象的`getResponseHeader()`方法只能拿到6个基本字段：`Cache-Control`、`Content-Language`、`Content-Type`、`Expires`、`Last-Modified`、`Pragma`。如果想拿到其他字段，就必须在`Access-Control-Expose-Headers`里面指定。上面的例子指定，`getResponseHeader('FooBar')`可以返回`FooBar`字段的值。

##### 复杂请求

除了简单请求之外的所有其他请求。

> 比如请求方法是`PUT`或`DELETE`，或者`Content-Type`字段的类型是`application/json`，或者带了别的字段比如 `Authorization ` 字段

非简单请求的 CORS 请求，会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求（preflight）。浏览器先询问服务器，当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些 HTTP 动词和头信息字段。只有得到肯定答复，浏览器才会发出正式的 `XMLHttpRequest` 请求，否则就报错。

下面是一段浏览器的JavaScript脚本。

```javascript
var url = 'http://api.alice.com/cors';
var xhr = new XMLHttpRequest();
xhr.open('PUT', url, true);
xhr.setRequestHeader('X-Custom-Header', 'value');
xhr.send();
```

浏览器发现，这是一个非简单请求，就自动发出一个"预检"请求，要求服务器确认可以这样请求。下面是这个"预检"请求的HTTP头信息。

```http
OPTIONS /cors HTTP/1.1
Origin: http://api.bob.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Custom-Header
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```

"预检"请求用的请求方法是`OPTIONS`，表示这个请求是用来询问的。头信息里面，关键字段是`Origin`，表示请求来自哪个源。除了`Origin`字段，"预检"请求的头信息包括两个特殊字段。

+ Access-Control-Request-Method：该字段是必须的，用来列出浏览器的 CORS 请求会用到哪些 HTTP 方法，上例是 `PUT`。
+ 该字段是一个逗号分隔的字符串，指定浏览器 CORS 请求会额外发送的头信息字段，上例是 `X-Custom-Header`。

**服务器收到"预检"请求以后**，检查了`Origin`、`Access-Control-Request-Method`和`Access-Control-Request-Headers`字段以后，确认允许跨源请求，就可以做出回应。

```http
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:15:39 GMT
Server: Apache/2.0.61 (Unix)
Access-Control-Allow-Origin: http://api.bob.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: X-Custom-Header
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 0
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain
```

上面的HTTP回应中，关键的是`Access-Control-Allow-Origin`字段，表示`http://api.bob.com`可以请求数据。该字段也可以设为星号，表示同意任意跨源请求。

**如果服务器否定了"预检"请求，会返回一个正常的HTTP回应，但是没有任何CORS相关的头信息字段。这时，浏览器就会认定，服务器不同意预检请求，因此触发一个错误**，被`XMLHttpRequest`对象的`onerror`回调函数捕获。控制台会打印出如下的报错信息。

```http
XMLHttpRequest cannot load http://api.alice.com.
Origin http://api.bob.com is not allowed by Access-Control-Allow-Origin.
```

服务器回应的其他CORS相关字段如下。

```http
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: X-Custom-Header
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 1728000
```

+ Access-Control-Allow-Methods：该字段必需，它的值是逗号分隔的一个字符串，表明服务器支持的所有跨域请求的方法。注意，返回的是所有支持的方法，而不单是浏览器请求的那个方法。这是为了避免多次"预检"请求。
+ Access-Control-Allow-Headers：如果浏览器请求包括`Access-Control-Request-Headers`字段，则`Access-Control-Allow-Headers`字段是必需的。它也是一个逗号分隔的字符串，表明服务器支持的所有头信息字段，不限于浏览器在"预检"中请求的字段。

**一旦服务器通过了"预检"请求，以后每次浏览器正常的CORS请求，就都跟简单请求一样，会有一个`Origin`头信息字段。***服务器的回应，也都会有一个`Access-Control-Allow-Origin`头信息字段。

下面是"预检"请求之后，浏览器的正常CORS请求。

```http
PUT /cors HTTP/1.1
Origin: http://api.bob.com
Host: api.alice.com
X-Custom-Header: value
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```

上面头信息的`Origin`字段是浏览器自动添加的。下面是服务器正常的回应。

```http
Access-Control-Allow-Origin: http://api.bob.com
Content-Type: text/html; charset=utf-8
```

上面头信息中，`Access-Control-Allow-Origin`字段是每次回应都必定包含的。

#### JSONP

> JSONP 的工作原理是什么？ - 贺师俊的回答 - 知乎 https://www.zhihu.com/question/19966531/answer/13502030

CORS 与 JSONP 的使用目的相同，但是比 JSONP 更强大。

JSONP 只支持`GET`请求，CORS 支持所有类型的 HTTP请求。JSONP的优势在于支持老式浏览器，以及可以向不支持 CORS 的网站请求数据。

> 当需要通讯时，本站脚本创建一个<script>元素，地址指向第三方的API网址，形如：    
>
> <script src="http://www.example.net/api?param1=1&param2=2"></script>    
>
> 并提供一个回调函数来接收数据（函数名可约定，或通过地址参数传递）。第三方产生的响应为 json 数据的包装（故称之为 jsonp，即 json padding），形如：` callback({"name":"hax","gender":"Male"}) `。这样浏览器会调用 callback 函数，并传递解析后 json 对象作为参数。本站脚本可在 callback 函数里处理所传入的数据。

### 攻击

诱导其他用户触发陷阱，在其浏览器中运行恶意代码，篡改用户的 HTTP 请求，导致 Cookie 等个人信息泄露，登陆状态中的用户权限遭到恶意滥用。  
尤其需要注意的是，如果此时用户通过 VPN 等方式访问了某企业内网，则内网信息也会外泄。

+ 跨站脚本攻击（Cross-Site Scripting, XSS）  
通过存在安全漏洞的网站的用户的浏览器内运行非法 HTML 标签或 JS 进行攻击。如：
  + 利用虚假的输入表单骗取用户个人信息
  + 利用脚本窃取用户的 Cookie 值
  + 显示伪造的文章或图片  
  
  一般来说，都是通过在 URL 等处置入恶意 JS 代码，向外泄露了信息。
  ![](11-1.png)
  ![](11-2.png)
  ![](11-3.png)

  需要指出的是，就算不触发这些代码，也有可能仅仅是把 Cookie 存放在 local storage 中，就被恶意 JS 读取泄露。

+ 会话固定攻击
![](11-4.png)

+ 跨站点请求伪造（Cross-Site Request Forgeries, CSRF）  
  [https://zhuanlan.zhihu.com/p/37293032](https://zhuanlan.zhihu.com/p/37293032)  
  挟制用户在当前已登录的Web应用程序上执行非本意的操作。攻击者通过一些技术手段欺骗用户的浏览器去访问一个自己曾经认证过的网站并运行一些操作（如发邮件，发消息，甚至财产操作如转账和购买商品）。由于浏览器曾经认证过，所以被访问的网站会认为是真正的用户操作而去运行。这利用了web中用户身份验证的一个漏洞：简单的身份验证只能保证请求发自某个用户的浏览器，却不能保证请求本身是用户自愿发出的。  

  跟跨站脚本攻击（XSS）相比，XSS 利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。

  > 假如一家银行用以运行转账操作的URL地址如下：http://www.examplebank.com/withdraw?account=AccoutName&amount=1000&for=PayeeName
  >
  > 那么，一个恶意攻击者可以在另一个网站上放置如下代码： 
  > ```
  > <img src="http://www.examplebank.com/withdraw?account=Alice&> amount=1000&for=Badman">
  > ```
  > 如果有账户名为Alice的用户访问了恶意站点，而她之前刚访问过银行不久，登录信息尚未过期，那么她就会损失1000资金。

  防御措施:
  + 检查Referer字段：HTTP头中有一个Referer字段，这个字段用以标明请求来源于哪个地址。在处理敏感数据请求时，通常来说，Referer字段应和请求的地址位于同一域名下。以上文银行操作为例，Referer字段地址通常应该是转账按钮所在的网页地址，应该也位于 www.examplebank.com 之下。而如果是CSRF攻击传来的请求，Referer字段会是包含恶意网址的地址，不会位于 www.examplebank.com 之下，这时候服务器就能识别出恶意的访问
  + 添加校验token：要求用户浏览器提供不保存在cookie中，并且攻击者无法伪造的数据作为校验。