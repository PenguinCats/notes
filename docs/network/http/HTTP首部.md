# HTTP 首部

HTTP 报文首部必须存在于报文之中，用于客户端与服务器分别处理请求和响应，根据之前所述：

+ 请求报文首部含有：请求行、请求首部字段、通用首部字段、实体首部字体、其他。具体而言：  
  
  + 请求行：方法（GET/POST 等）、URL、HTTP版本
  + HTTP 首部字段：请求首部字段、通用首部字段、实体首部字段
  + 其他

+ 响应报文首部含有：状态行、响应首部字段、通用首部字段、实体首部字段、其他。具体而言：
  
  + 状态行：HTTP 版本、状态码
  
  + HTTP 首部字段：请求首部字段、通用首部字段、实体首部字段
  
  + 其他
    
    ## HTTP 首部字段
    
    HTTP 首部字段用于传递一些重要信息，包含了报文主题大小、语言、认证信息等内容。其是由许多许多 `key:value` 键值对组成的。
    
    ### 通用首部字段

+ Cache-Control：用于操作缓存的工作机制
  
  + 缓存请求指令
    + no-cache：强制向源服务器验证是否有效
    + no-store：机密信息，不缓存请求或响应的任何内容
    + max-age=[秒]：无需确认的缓存时间
    + max-stale=[秒]：接受已过期的响应
    + min-fresh=[秒]：期望在指定时间内的响应一直有效
    + no-transform：代理不可更改媒体类型
    + only-if-cached：从缓存获取资源
    + cache-extension：拓展字段，可以加入一些拓展指令，服务器并不一定能够处理。不能处理时，会忽略。
  + 缓存响应字段
    + public：该缓存公开，可向所有用户提供该缓存
    + private：仅向特定用户返回该缓存
    + no-cache：强制禁止缓存，且源服务器不再提供缓存有效性确认服务
    + no-store：禁止缓存请求或响应的任意部分。
    + s-maxage/maxage：源服务器在一段时间内不提供缓存有效性确认，且保证缓存在该段时间内有效
    + no-transform 请求和响应中，不允许缓存更改实体主体的媒体类型，**防止缓存或代理压缩图片等操作**
    + must-revalidate/proxy-revalidate：源服务器允许代理缓存，但是要求每次认证
    + cache-extension

+ Connecttion
  
  + 控制代理不再转发的首部字段：代理删除该字段再转发
  + 管理持久连接：客户端要求持久连接：`Connection: Kepp-Alive`，服务器明确表示断开连接：`Connection:close`。**持久连接中，每个连接可以处理多个请求-响应事务。HTTP/1.1 默认使用持久连接**

+ Date：HTTP 报文的时间和日期

+ Pragma：弃用

+ Trailer：在首部中事先说明报文主体中有哪些内容。

+ Transfer-Encoding：指定传输时的编码格式

+ Upgrade：检测 HTTP 及其他协议是否可使用更高的版本进行通信。参数值用来指定新的通信协议。

+ Via：追踪客户端与服务器之间的请求和响应的路径。不仅可用于追踪路径，还可用于防止回路转发。

+ Warning

+ Content-Length
  
  > 用了这么久HTTP, 你是否了解Content-Length? - Fundebug的文章 - 知乎
  > https://zhuanlan.zhihu.com/p/81955498
  
  `Content-Length`是HTTP消息长度, 用**十进制数字**表示的**八位字节的数目**, 是Headers中常见的一个字段. `Content-Length`应该是精确的, 否则就会导致异常。
  
  `Content-Length`首部指示出报文中实体主体的字节大小. 这个大小是包含了所有内容编码的, 比如, 对文本文件进行了`gzip`压缩的话, `Content-Length`首部指的就是压缩后的大小而不是原始大小。
  
  + 如果 Content-Length 比实际的长度大, 服务端/客户端读取到消息结尾后, 会等待下一个字节, 自然会无响应直到超时
  
  + 如果这个长度小于实际长度, 首次请求的消息会被截取, 比如参数为`param=piaoruiqing`, `Content-Length`为10, 那么这次请求的消息会被截取为: `param=piao`
  
  但如在请求处理完成前无法获取消息长度, 我们就无法明确指定`Content-Length`, 此时应该使用`Transfer-Encoding: chunked`
  
  > http协议为什么使用传输编码？例如分块编码 Transfer-encoding:chunked ? - 上古清茗的回答 - 知乎
  > https://www.zhihu.com/question/22983235/answer/41278345
  > 
  > 
  > 
  > 采用分块传输主要的考量是对于体积较大的文件（eg：图片、文件），服务器不能每次都先把完整的文件读完，拿到Content-length再按照Transfer-Encoding: identity的方式返回。
  > 
  > 这样效率太低了，采用分块的方式可以实现边读边传，分块压缩传输，大大提高传输效率。
  > 
  > 其他的好处是：
  > 
  > 1. HTTP分块传输编码允许服务器为动态生成的内容维持[HTTP持久链接](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/HTTP%25E6%258C%2581%25E4%25B9%2585%25E9%2593%25BE%25E6%258E%25A5)。通常，持久链接需要服务器在开始发送消息体前发送Content-Length消息头字段，但是对于动态生成的内容来说，在内容创建完之前是不可知的。[[1]](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/%25E5%2588%2586%25E5%259D%2597%25E4%25BC%25A0%25E8%25BE%2593%25E7%25BC%2596%25E7%25A0%2581%23cite_note-1)
  > 2. 分块传输编码允许服务器在最后发送消息头字段。对于那些头字段值在内容被生成之前无法知道的情形非常重要，例如消息的内容要使用[散列](https://www.zhihu.com/search?q=%E6%95%A3%E5%88%97&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A%2241278345%22%7D)进行签名，散列的结果通过HTTP消息头字段进行传输。没有分块传输编码时，服务器必须缓冲内容直到完成后计算头字段的值并在发送内容前发送这些头字段的值。
  > 3. HTTP服务器有时使用[压缩](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/%25E6%2595%25B0%25E6%258D%25AE%25E5%258E%258B%25E7%25BC%25A9) （[gzip](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/Gzip)或[deflate](https://link.zhihu.com/?target=http%3A//zh.wikipedia.org/wiki/DEFLATE)）以缩短传输花费的时间。分块传输编码可以用来分隔压缩对象的多个部分。在这种情况下，块不是分别压缩的，而是整个负载进行压缩，压缩的输出使用本文描述的方案进行分块传输。在压缩的情形中，分块编码有利于一边进行压缩一边发送数据，而不是先完成压缩过程以得知压缩后数据的大小。
  > 
  > 
  
  数据以一系列分块的形式进行发送. `Content-Length` 首部在这种情况下不被发送. 在每一个分块的开头需要添加当前分块的长度, 以十六进制的形式表示，后面紧跟着 `\r\n` , 之后是分块本身, 后面也是`\r\n`. 终止块是一个常规的分块, 不同之处在于其长度为0.
  
  ![preview](https://pic2.zhimg.com/v2-cbd57216a2cb582759334cfb299d4461_r.jpg)
  
  ### 请求首部字段
  
  ### 响应首部字段
  
  ### 实体首部字段
