通信认证数据
==================================

在本节中，“通信”更加广义，包括用户体验 (UX) 和客户端-服务器通信。

不仅“_password 条目应该在用户的屏幕上被隐藏_”是正确的，而且“_remember me 功能应该被禁用_”也是如此。

您可以通过使用带有 `type="password"` 的输入框，并将 `autocomplete` 属性设置为 `off`[^1] 来完成这两项操作。

```html
<input type="password" name="passwd" autocomplete="off" />
```

身份验证凭据应仅通过加密连接 (HTTPS) 发送。电子邮件重置相关的临时密码可能可以不用 HTTPS 。

通常请求的 URL 由 HTTP 服务器（`access_log`）记录，其中包括查询字符串。为防止身份验证凭据泄漏到日志里面，应使用 HTTP `POST` 方法将数据发送到服务器。

```text
xxx.xxx.xxx.xxx - - [27/Feb/2017:01:55:09 +0000] "GET /?username=user&password=70pS3cure/oassw0rd HTTP/1.1" 200 235 "-" "Mozilla/5.0 (X11) ; Fedora; Linux x86_64; rv:51.0) Gecko/20100101 Firefox/51.0"
```

一个设计良好的用于身份验证的 HTML 表单如下所示：

```html
<form method="post" action="https://somedomain.com/user/signin" autocomplete="off">
    <input type="hidden" name="csrf" value="CSRF-TOKEN" />

    <label>用户名 <input type="text" name="username" /></label>
    <label>密码 <input type="password" name="password" /></label>

    <input type="submit" value="提交" />
</form>
```

在处理身份验证错误时，您的应用程序不应透露身份验证数据的哪一部分不正确。使用“无效的用户名和（或）密码”来替代“无效的用户名”或“无效的密码”：

```html
<form method="post" action="https://somedomain.com/user/signin" autocomplete="off">
    <input type="hidden" name="csrf" value="CSRF-TOKEN" />

    <div class="error">
        <p>无效的用户名和（或）密码</p>
    </div>

    <label>用户名 <input type="text" name="username" /></label>
    <label>密码 <input type="password" name="password" /></label>

    <input type="submit" value="提交" />
</form>
```

如果不这样做会暴露的信息如下：

* 谁已经注册过了：“无效的密码”表示用户名存在。
* 您的系统如何工作：“无效的密码”可能会揭示您的应用程序如何工作，首先查询数据库中的“用户名”，然后比较内存中的密码。

在[验证和存储部分][5]中提供了如何执行身份验证数据验证（和存储）的示例。

成功登录后，应通知用户上次成功或不成功的访问日期/时间，以便他可以检测和报告可疑活动。有关日志记录的更多信息可以在文档的 [`错误处理和日志记录`][4] 部分中找到。此外，还建议在检查密码时使用恒定时间比较功能，以防止计时攻击。后者包括分析具有不同输入的多个请求之间的时间差异。在这种情况下，“记录 == 密码”形式的标准比较将在第一个不匹配的字符处返回 false。提交的密码越接近，响应时间越长。通过利用这一点，攻击者可以猜测密码。请注意，即使记录不存在，我们也总是强制执行带有空值的 `subtle.ConstantTimeCompare` 以将其与用户输入进行比较。

---

[^1]: [How to Turn Off Form Autocompletion][1], Mozilla Developer Network
[^2]: [Log Files][2], Apache Documentation
[^3]: [log_format][3], Nginx log_module "log_format" directive

[1]: https://developer.mozilla.org/en-US/docs/Web/Security/Securing_your_site/Turning_off_form_autocompletion
[2]: https://httpd.apache.org/docs/1.3/logs.html#accesslog
[3]: http://nginx.org/en/docs/http/ngx_http_log_module.html#log_format
[4]: ../error-handling-logging/logging.md
[5]: ./validation-and-storage.md#storing-password-securely-the-practice
