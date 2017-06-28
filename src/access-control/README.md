访问控制
==============

在谈到访问控制，首先要说的就是做访问控制的这个系统必须要是受信任的。
在这个例子（[Session Management][3]）中，我们在服务器端使用 JWT(JSON Web Tokens) 来生成会话 token.

```go
// create a JWT and put in the clients cookie
func setToken(res http.ResponseWriter, req *http.Request) {
    //30m Expiration for non-sensitive applications - OWASP
    expireToken := time.Now().Add(time.Minute * 30).Unix()
    expireCookie := time.Now().Add(time.Minute * 30)

    //token Claims
    claims := Claims{
        {...}
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signedToken, _ := token.SignedString([]byte("secret"))
```

我们可以使用这个 token 来验证用户，并启用我们的`Access Control`模块。

用来做权限控制的组件应该只有一个，而且是全站公用的。包含了调用外部授权服务的库。

万一失败时，访问控制应该安全的失败(In case of failure, access control should fail securely)。在 Go 中，我们可以使用`defer`来实现，具体细节可以参考[Error Logging][1]章节。 

如果程序不能读到自己的配置信息，那么所有对这个程序的请求都应该被拒绝。

针对每个请求都需要做权限验证，包含服务器端的脚本和客户端的一些技术(AJAX or Flash)

还有就是把一些特俗逻辑的代码跟其他代码区别开来也非常重要。

其他需要做防止未授权访问的资源有：

* 文件和其他资源
* 受保护的 URL 
* 受保护的功能
* 直接对象引用
* 服务
* 程序数据
* 用户和数据属性和政策信息

在提供的实例中，我们会对一个简单的直接对象引用进行测试。代码在[sample in the Session Management][2]。

实现这些访问控制时,重要的是要验证的服务器端实现和展现层表示访问控制规则匹配。

如果状态数据需要存在客户端，那么是有必要做加密和完整行检查来防止数据被篡改。

程序的逻辑流程必须要遵守业务规则。

在处理事务的时候，单个用户或者设备在单位时间内的交易数量必须要高于业务需要，但是也不能高到让用户可以进行 DoS 攻击。

仅仅使用`referer` HTTP header 是不足以验证权限的，这个应该是作为附加的验证条件而不是主要验证条件。

对于经过验证的会话，应用程序应该定期进行重新评估用户的授权，以验证用户的权限有没有改变。如果权限更改了，将用户记录下来并强制让他们重新认证。

用户的账户也需要有一种方式来审计他们，以便遵从安全准则(例如，禁用那么些超过密码过期时间30天也没有更换密码的账户)。

当用户的授权被取消时，应用程序还必须支持帐户的禁用和终止会话。(例如: 角色改变,就业状况,等等)。

当支持外部服务帐户和支持来自外部系统的连接的帐户时，这些帐户必须在可能的最低级别上运行。

[1]: /error-handling-logging/error-handling.md
[2]: URL.go
[3]: /session-management/README.md
