SQL 注入
=============

由于缺乏正确的输出编码，另一个常见的注入是 SQL 注入，主要是因为古董时代的不良做法：字符串拼接。

思考一下下面这个查询：

```go
customerId := r.URL.Query().Get("id")
query := "SELECT number, expireDate, cvv FROM creditcards WHERE customerId = " + customerId

row, _ := db.Query(query)
```
这简直就是作死。

当输入的是一个合法的`customerId`的时候，你可以正确的列出这个用户的信用卡，但如果`customerId`是`1 OR 1=1`会怎么样呢？

你的查询语句会变成这样：

```SQL
SELECT number, expireDate, cvv FROM creditcards WHERE customerId = 1 OR 1=1
```
然后你将会列出这张表的所有记录（`1=1`恒定为`true`）！

只有一种让你的数据库安全的办法：[Prepared Statements][1].

```go
customerId := r.URL.Query().Get("id")
query := "SELECT number, expireDate, cvv FROM creditcards WHERE customerId = ?"

stmt, _ := db.Query(query, customerId)
```
注意占位符`?`，你的查询语句应该是：

 * 可读
 * 较短
 * 安全

在 Prepared statements 中不同的数据库占位符也不同，例如

| MySQL | PostgreSQL | Oracle |
| :---: | :--------: | :----: |
| WHERE col = ? | WHERE col = $1 | WHERE col = :col |
| VALUES(?, ?, ?) | VALUES($1, $2, $3) | VALUES(:val1, :val2, :val3) |

你可以在本书的数据库安全章节中找到更加深入的介绍。

[1]: https://golang.org/pkg/database/sql/#DB.Prepare
