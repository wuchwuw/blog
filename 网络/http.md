## DNS

浏览器DNS缓存
操作系统DNS缓存
host文件
配置的非权威域名服务器(8.8.8.8 114.114.114.114)

递归或者迭代查询  根域名服务器 -> 顶级域名服务器 -> 权威域名服务器 ->

## 状态码

1xx: 信息
2xx: 响应成功
3xx: 资源已经转移
4xx: 客户端报文格式出错
5xx: 服务器出错

## 请求报文

### 请求行

协议 方法

### 实体

相关字段

Accept: 告诉服务器能接收什么格式的数据

Content-Type: 请求响应都可以使用，用来指定实体类型

Accept-Encoding: 浏览器支持的压缩格式

Content-Encoding: 服务器返回的实体的压缩格式

## 大文件传输

数据压缩： gzip、br等，Accept-Encoding、Content-Encoding

分块传输：在响应报文里指定Transfer-Encoding: chunked

范围请求：
- 在响应头中指定Accept-Ranges: bytes表示支持范围请求， Content-Range: bytes x-y/length表示返回的片段
- 在请求头中指定Range: bytes=x-y

多段范围请求：
- 请求头中Range指定多个返回Range: bytes=x-y, x-y
- 响应头中指定Content-Type: multipart/byteranges; boundary=00000000001
  通过boundary分隔多段数据

## 连接

长连接: Connection: keep-alive

长连接关闭方式：
- Connection: close
- Nginx相关： keepalive_timeout多少时间内无数据发送时关闭  keepalive_requests处理了多少请求后关闭
- 通用字段Keep-Alive: timeout=value

并发连接：发起多个长连接提高效率，浏览器为6-8个

## 重定向

相关字段：

- 响应头中指定Location: path 指定跳转链接可指定绝对或相对路径

相关状态码

- 301 永久重定向，资源的URI永久改变

- 302 临时重定向，资源的URI暂时改变

## Cookie

有状态的http请求，值为key=value

相关属性:

domain: 指定哪些主机可以接受Cookie

path: 指定主机下哪些路径可以接受Cookie

SameSite：跨站请求是是否发送Cookie (None, Strict, Lax)

Max-Age: 设置失效时间，单位秒

Expires：设置过期时间

HttpOnly: 只在服务端使用，客户端不能获取、修改

Secure: 只能用https发送

## 缓存

相关字段:

Cache-Control:

- no-store: 不缓存
- no-cache: 使用缓存前校验是否过期
- must-revalidate：不过期就可以使用，过期就去服务器验证
- max-age: 设置缓存过期时间
- if-Modified-Since Last-modified
- If-None-Match ETag(有强弱之分，弱ETag(前面带有W/)只要求语义不变)

## 代理

- 匿名代理：完全隐匿被代理的机器，外界只看到代理服务器
- 透明代理：完全透明开放，外界既知道代理，也知道客户端
- 正向代理：代理客户端向服务器发送请求
- 反向代理：代理服务端响应客户端的请求

部分功能:

- 负载均衡：把访问请求均匀分散到多台机器，实现访问集群化；
- 内容缓存：暂存上下行的数据，减轻后端的压力；
- 安全防护：隐匿 IP, 使用 WAF 等工具抵御网络攻击，保护被代理的机器；
- 数据处理：提供压缩、加密等额外的功能。

相关字段

- Via: proxy1, proxy2 (经过的所有主机名)
- X-Real-IP: 响应报文中显示客户端ip
- X-Forwarded-For: 所有代理客户端ip

## HTTPS

对称加密

## HTTP/2

- 头部压缩：建立字典，用索引号表示重复的字符串，压缩头部
- 二进制：headers body采用二进制格式，
- 流：使用流的形式传输，解决了队头阻塞问题，实现了多路复用，一个流对应http1的一次请求-应答
  通过id排序二进制帧
- 安全：增强安全性

## CDN

基于缓存代理，通过全局负载均衡（Global Sever Load Balance）一般简称为 GSLB寻找最佳节点