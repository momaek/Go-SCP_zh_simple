验证
==========

在验证检查中，为了保证用户输入的数据是预期的，需要使用一系列条件来验证用户的输入。

**重要:** 如果验证失败，那么必须要拒绝用户的输入。

这不仅是从安全的角度出发，而且从数据一致性和完整性的角度来看也是非常重要的，因为数据通常用在各种系统和应用程序中。

这篇文章列出了在使用 Go 开发 Web 程序的时候需要注意的安全风险。

## 用户交互

在应用程序中，任何一个允许用户输入的地方都存在潜在的安全风险，安全问题不仅仅是发生在被恶意代理攻击的时候，还可能是由于人为的错误输入导致的（据统计，大部分非法数据是由于人为错误引起的）。在 Go 中有几种方法来防范这些问题。

Go 的官方库里面包含有处理这些问题的方法，在处理字符串的时候，我们可以使用下面的包：

* `strconv` 包可以可以把字符串转换成其他类型的数据
    * [`Atoi`](https://golang.org/pkg/strconv/#Atoi)
    * [`ParseBool`](https://golang.org/pkg/strconv/#ParseBool)
    * [`ParseFloat`](https://golang.org/pkg/strconv/#ParseFloat)
    * [`ParseInt`](https://golang.org/pkg/strconv/#ParseInt)
* `strings` 包包含可以处理字符串的很多方法
    * [`Trim`](https://golang.org/pkg/strings/#Trim)
    * [`ToLower`](https://golang.org/pkg/strings/#ToLower)
    * [`ToTitle`](https://golang.org/pkg/strings/#ToTitle)
* [`regexp`][4] 包支持使用正则表达式处理自定义格式数据，例如 [^1]
* [`utf8`][9] 包实现了一堆函数来处理 UTF-8 编码的文本，包含 rune 和 UTF-8 字节序列之间的转换等等

  验证 UTF-8 编码的字符:
    * [`Valid`](https://golang.org/pkg/unicode/utf8/#Valid)
    * [`ValidRune`](https://golang.org/pkg/unicode/utf8/#ValidRune)
    * [`ValidString`](https://golang.org/pkg/unicode/utf8/#ValidString)

  编码 UTF-8 字符:
    * [`EncodeRune`](https://golang.org/pkg/unicode/utf8/#EncodeRune)

  解码 UTF-8:
    * [`DecodeLastRune`](https://golang.org/pkg/unicode/utf8/#DecodeLastRune)
    * [`DecodeLastRuneInString`](https://golang.org/pkg/unicode/utf8/#DecodeLastRuneInString)
    * [`DecodeRune`](https://golang.org/pkg/unicode/utf8/#DecodeLastRune)
    * [`DecodeRuneInString`](https://golang.org/pkg/unicode/utf8/#DecodeRuneInString)


**注意**: `Forms`在 Go 里面是被当成有`String`值类型的`Maps`。

确保数据合法的其他技术包括:

* _白名单_ - 在条件允许的情况下，使用白名单的方式来验证用户的输入，参见 [验证 - 移除标签][1](Strip tags)
* _边界检查_ - 检查数据和数字的长度
* _转义字符_ - 特殊字符需要做转义，比如单引号 "'"
* _数值检查_ - 如果输入的是数值，那么需要检查是否溢出
* _检查空字节_ - `(%00)`
* _检查换行符_ - `%0d`, `%0a`, `\r`, `\n`
* _检查路径改变符_ - `../` or `\\..`
* _检查扩展的 UTF-8_ - 检查特殊字符的替代表示，

**注意**: 需要确保 HTTP 请求和相应头只包含 ASCII 字符。

在 Go 中现存的处理安全问题的第三方包:

* [Gorilla][6] - 在 Web 安全中使用最多的包，它支持`websockets`,`cookie sessions`,`RPC`等等。

* [Form][7] - 把`url.Values`转换成 Go 值或者把 Go 值转换成`url.Values`,支持嵌套`Array`和全`map`

* [Validator][8] - 验证`Struct`和`Field`，包括`Cross Field`，`Cross Struct` ，`Map`还有`Slice`和`Array`

## 重定向

Go 语言在内部处理了重定向，意思就是除非你在服务端指定，否则在有重定向的时候 Go 将直接重定向到目标页面。这样会有安全问题，恶意代理可以绕过服务器端的数据验证过程，直接将数据提交到重定向的目标页面。

因此，开发者需要特别关注这种情形下的数据验证，以确保提供的是预期的合法数据。

## 文件操作

任何时候操作文件(读或者写)都需要验证检查，因为大多数情况下的文件操作就是直接操作的用户数据。

其他文件检查步骤包括 "File existence check"，以验证文件名是否存在。

添加文件信息在文档的[文件管理][2]部分，错误处理在[错误处理][3]部分。

## 数据源

在任何时候，数据从一个受信任的源传递到一个不太可靠的源都需要做完整性检查，这样可以保证数据没有被篡改，我们收到的也就是预期的数据。其他数据源检查的方法包括：

* _跨系统一致性检查_
* _Hash 统计_
* _参照完整性_

**注意:** 在现代关系型数据库中，如果主键字段中的值不受数据库内部机制的约束，那么它们应该被验证。

* _唯一性验证_
* _Table look up check_

## Post-validation Actions

根据数据验证的最佳实践，输入验证只是数据验证的第一步，因此后验证行为是很有必要的。
后验证会根据上下文变化，主要分为以下三类：

## Enforcement Actions

存在几种类型的 Enforcement Actions，为了更好的保护我们的应用程序和数据。

其中一种是通知用户提交的数据未能符合要求，必须要修改提交的数据以符合要求的条件。

另外一种就是直接在服务端修改用户提交的数据而不通知用户，这最适用于具有交互式使用的系统。

**注意:** 第二种方式仅仅是用于普通的修改（如果修改敏感数据的话，可能会造成数据被截断，最终导致数据丢失）。

* **建议操作** - 这个通常允许输入不变的数据，但数据来源通常知道数据有问题。这个最适合没有交互的系统。
* **验证操作** - 验证操作是建议操作的一个特例。在这个例子里面，用户输入的数据会被建议要如何修改，用户可以接受这些变更，或者保留他原来的输入。

一个简单的例子是填写账单地址的表单，当用户填完的时候，系统会提供该账户输入地址的建议，用户可以采纳建议，或者按照自己原来的输入数据来提交。

---

[^1]: 在自己写正则之前先看看[OWASP Validation Regex Repository][5]

[1]: sanitization.md
[2]: ../file-management/README.md
[3]: ../error-handling-logging/README.md
[4]: https://golang.org/pkg/regexp/
[5]: https://www.owasp.org/index.php/OWASP_Validation_Regex_Repository
[6]: https://github.com/gorilla/
[7]: https://github.com/go-playground/form
[8]: https://github.com/go-playground/validator
[9]: https://golang.org/pkg/unicode/utf8/
