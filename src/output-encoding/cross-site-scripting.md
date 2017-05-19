XSS - 跨站脚本攻击
==========================

大多数开发者已经听说过 XSS，但是其中的大部分人从没有尝试使用 XSS 去攻击 Web 程序。

跨站脚本攻击在2003年就已经在 [OWASP Top 10][0]，到现在依然是一个常见的漏洞。
在 [2013 version][1] 有相当详细的XSS介绍：攻击维度，安全薄弱点，技术影响还有商业影响等。

简单来说就是：

> 在把输入数据输出到页面之前，如果你能不确保所有用户提供的输入被正确转义，或者不通过服务器端输入验证来验证它是否安全，那么你的 Web 程序将受到威胁。 ([source][1])

Go，就像任何其他多用途编程语言一样，尽管 `html/ template` 包的文档很清楚，但还是有很多东西会混淆而且让你更加容易受到XSS的攻击。你在网上可以很简单的找到使用 `net/ http` 和 `io` 包写的 “hello world” 的例子，但是你没有意识到你已经受到了 XSS 攻击。

思考一下下面这段代码：

```go
package main

import "net/http"
import "io"

func handler (w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, r.URL.Query().Get("param1"))
}

func main () {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

这个代码片段创建了一个 HTTP Server，跑在 `8080` 端口(`main()`)，处理到根目录的请求(`/`)。

请求处理函数 `handler()` 从查询参数里面获取到 `param1` 参数的值，然后写到响应流里面(`w`)。

由于 `Content-Type HTTP` 响应头没有明确定义，所以将使用遵循 [WhatWG spec][5] 规范的 Go 默认值 `http.DetectContentType`。

因此，如果 `param1` 等于 "test"，那么 HTTP 的响应头 `Content-Type` 会被设置成 `text/plain`。

![Content-Type: text/plain][content-type-text-plain]

但是如果 `param1` 等于 "&lt;h1&gt;"，那么 `Content-Type` 将会被设置成 `text/html`。

![Content-Type: text/html][content-type-text-html]

到这里，你可能会认为如果 `param1` 等于任何一个 HTML tag，那么 `Content-Type` 都会被设置成 `text/html`，事实上不是这样的。当 `param1` 等于 "&lt;h2&gt;", "&lt;span&gt;" 或者 "&lt;form&gt;" 的时候，`Content-Type` 的值是 `plain/text` 而不是 `text/html`。

现在，我们让 `param1` 等于 `<script>alert(1)</script>`。

根据 [WhatWG spec][5] 规范，如果 HTTP 响应头 `Content-Type` 的值是 `text/html`，那么 `param1` 的值将会被渲染，这就是 XSS - 跨站脚本攻击。 

![XSS - Cross-Site Scripting][cross-site-scripting]

在与 Google 谈论这种情况后，他们告诉我们说：

> 实际上，自动设置 content-type，打印 html 是很方便而且有意义的，我们希望开发者自己可以使用 html/template 包做适当的转义。

Google 表示，开发者有责任清洗和保护自己的代码。我们完全同意，但是允许自动设置 `Content-Type` 而不是用 `text/plain` 作为默认的，在以安全性为优先的语言里面这不是最好的做法。

我们需要清楚一点，`text/plain` 和(或) [text/template][6] 包并不会让你远离 XSS，因为他不会清洗用户的输入。

```go
package main

import "net/http"
import "text/template"

func handler(w http.ResponseWriter, r *http.Request) {
        param1 := r.URL.Query().Get("param1")

        tmpl := template.New("hello")
        tmpl, _ = tmpl.Parse(`{{define "T"}}{{.}}{{end}}`)
        tmpl.ExecuteTemplate(w, "T", param1)
}

func main() {
        http.HandleFunc("/", handler)
        http.ListenAndServe(":8080", nil)
}
```

使 `param1` 等于"&lt;h1&gt;"，那么 `Content-Type` 将会被设置成 `text/html`，这点会让你受到 XSS 的威胁。

![XSS while using text/template package][text-template-xss]

把 [text/template][6] 包替换成 [html/template][2] 包的话，你已经开始安全前行了。

```go
package main

import "net/http"
import "html/template"

func handler(w http.ResponseWriter, r *http.Request) {
        param1 := r.URL.Query().Get("param1")

        tmpl := template.New("hello")
        tmpl, _ = tmpl.Parse(`{{define "T"}}{{.}}{{end}}`)
        tmpl.ExecuteTemplate(w, "T", param1)
}

func main() {
        http.HandleFunc("/", handler)
        http.ListenAndServe(":8080", nil)
}
```

当 `param1` 等于 "&lt;h1&gt;" 时，不仅仅是 `Content-Type` 的值会被设置成 `text/plain`

![Content-Type: text/plain while using html/template package][html-template-plain-text]

而且 `param1` 的值也会被正确的编码，然后返回给浏览器。

![No XSS while using html/template package][html-template-noxss]

[exploit-of-a-mom]: images/exploit-of-a-mom.png
[content-type-text-plain]: images/text-plain.png
[content-type-text-html]: images/text-html.png
[cross-site-scripting]: images/xss.png
[text-template-xss]: images/text-template-xss.png
[html-template-plain-text]: images/html-template-plain-text.png
[html-template-noxss]: images/html-template-text-plain-noxss.png

[0]: https://www.owasp.org/index.php/Category:OWASP_Top_Ten_Project
[1]: https://www.owasp.org/index.php/Top_10_2013-A3-Cross-Site_Scripting_(XSS)
[2]: https://golang.org/pkg/html/template/
[3]: https://golang.org/pkg/net/http/
[4]: https://golang.org/pkg/io/
[5]: https://mimesniff.spec.whatwg.org/#rules-for-identifying-an-unknown-mime-typ
[6]: https://golang.org/pkg/text/template/
