数据清洗
============

数据清洗是指删除或者更换提交的数据的过程。处理数据时，在经过适当的验证检查后，通常采取的加强数据安全性的一个步骤就是数据清洗。
最常见的清洗方法有:

## 把 `<` 转为 entity

在内置的包 `html` 里面有两个方法是用来做数据清洗的，一个是用来转换 HTML 中的特殊字符，另外一个是把特殊字符转换成 HTML。
`EscapeString()` 接受一个字符串，返回一个转换后的字符串。比如：`<` 转换成 `&lt;`。注意，这个函数只会转换这五个字符 `<`,`>`,`&`,`'` 还有 `"`。
相反的，`UnescapeString()` 则是将上述五个字符转换回去。

## 移除所有 tags

尽管 `html/template` 包有一个叫 `stripTags()` 的函数，但是它不是公开的。再加上也没有其他的内置的包支持_移除所有tags_，所以我们只有使用第三方库，或者你把整个函数拷贝出来使用。

下面列出的第三方库可以达成上述目的：

* https://github.com/kennygrant/sanitize
* https://github.com/maxwells/sanitize

## 移除换行符，tab 和多余的空格

这个 `text/template` 包和这个 `html/template` 包都有一个通过分隔符 `-` 来移除模版中的空格的方法。

执行以下模版：

```
{{- 23}} < {{45 -}}
```

将输出：

```
23<45
```

**注意**: 如果 `-` 没有跟在 {{ 后面，或是 }} 前面时，它就会被认为是附加在数值上的。

以下：

```
{{ -3 }}
```

会变成：

```
-3
```

## URL 请求路由

在 `net/http` 包中，有一个处理 HTTP 请求路由的框架，叫 `ServeMux`。它用来将请求对应到注册好的 pattern，然后调用对应的 handler 来处理这个请求。此外，它另外一个主要目的是处理 URL 的路由，把包含有 `.`、`..` 或有反复的斜杠的重定向到一个等效，但是更加赶紧的 URL。

一个简单的例子如下:

```go
func main() {
  mux := http.NewServeMux()

  rh := http.RedirectHandler("http://yourDomain.org", 307)
  mux.Handle("/login", rh)

  log.Println("Listening...")
  http.ListenAndServe(":3000", mux)
}
```

第三方库:

* [Gorilla Toolkit - MUX][1]

[1]: http://www.gorillatoolkit.org/pkg/mux
