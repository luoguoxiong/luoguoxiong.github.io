# CSRF 跨站请求伪造

## 一个典型的 CSRF 攻击有着如下的流程：

* 受害者登录 a.com，并保留了登录凭证（Cookie）。
* 攻击者引诱受害者访问了 b.com。
* b.com 向 a.com 发送了一个请求：a.com/act=xx。浏览器会默认携带 a.com 的 Cookie。
* a.com 接收到请求后，对请求进行验证，并确认是受害者的凭证，误以为是受害者自己发送的请求。
* a.com 以受害者的名义执行了 act=xx。
* 攻击完成，攻击者在受害者不知情的情况下，冒充受害者，让 a.com 执行了自己定义的操作。

## CSRF 分析
可以看到，csrf 能成功的原因在于：借助浏览器 cookie 的机制，非法使用了用户的 cookie。

但也可以看出来，攻击者无法读取和操作 cookie，只是发起伪造请求，带上了 cookie。

## CSRF 防范策略
根据分析：

1. 禁用跨域请求

2. 携带一些本域下才会生成的字段

要防止字段信息保存在 cookie 中，所以可以保存在 session 中。在渲染完页面后，利用 js 脚本在表单提交中新增默认字段，或者在请求中 url 中携带字段。

对于大型网站，session 维护会有成本。可以使用一些通用的分布式 token 设计，服务端不需要存储 session，只需要对 token 的合法性进行一次计算验证即可。