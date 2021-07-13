输出编码
===============

虽然在 [OWASP SCP Quick Reference Guide][1] 中针对输出编码的就只有 6 小节，但是在 Web 开发中不正确的输出编码是相当普遍的，这就导致了一个常见的漏洞：[Injection][2]

在 Web 程序越来越丰富且复杂的同时，数据的来源也越多：用户，数据库，第三方库等。当你把这些数据都放在浏览器里面展示的时候，如果没有一个强大的输出编码策略，那么就有发生注入的风险。

当然，你或许已经知道在本章节中我们要处理的安全问题，但是你真是的知道它们是如何发生，并且如何避免吗？

[1]: https://www.owasp.org/images/0/08/OWASP_SCP_Quick_Reference_Guide_v2.pdf
[2]: https://www.owasp.org/index.php/Top_10_2013-A1-Injection
