验证和存储认证数据
==========================================

## 验证
----------

鉴于越来越多的用户帐户数据库在 Internet 上泄露，本节的关键主题是“身份验证数据存储”。当然，这不能保证一定会发生，但在这种情况下，如果正确存储身份验证数据，尤其是密码，则可以避免附带损害。

首先，让我们明确“_所有身份验证控制都应该安全的失败_”。 我们建议你阅读“身份验证和密码管理”的所有其他部分，因为它们涵盖了有关报告错误身份验证数据以及如何处理日志记录的建议。

另一项初步建议如下：
* 对于顺序身份验证实现（就像 Google 现在所做的那样），验证应该只在所有数据输入完成时发生，在受信任的系统（例如服务器）上。


## 安全存储密码：理论

现在让我们讨论存储密码。

你真的不需要存储密码，因为它们是由用户提供的（明文）。但是你需要在每次身份验证时验证用户是否提供相同的令牌。

因此，出于安全原因，你需要的是“单向”函数 `H`，因此对于每个密码 `p1` 和 `p2`，`p1` 与 `p2` 不同，`H(p1)` 是也不同于`H(p2)`[^1]。

这听起来或看起来像数学吗？

注意最后一个要求：`H` 不应该有这样的`H⁻¹(H(p1))` 等于`p1`的 `H⁻¹` 这个函数。这样的话就没办法通过 `H(p1)` 逆向推导出 `p1`，除非你尝试所有可能的 `p` 值。

如果“H”只是单向，那么帐户泄漏的真正问题是什么？

好吧，如果你知道所有可能的密码，你可以预先计算它们的哈希值，然后运行彩虹表攻击。

当然你已经知道，从用户的角度来看，密码很难管理。并且用户不仅能够重复使用密码，而且他们还倾向于使用易于记住的东西，所以他们的秘密是可以用某种方式猜测的。

我们怎样才能避免这种情况？

关键是：如果两个不同的用户提供相同的密码“p1”，我们应该存储不同的哈希值。这听起来可能是不可能的，但还是可以做到的 `salt`：一个伪随机 ** 每个用户密码唯一** 值，它被添加到 `p1` 之前，因此结果哈希计算如下：`H(salt + p1 )`。

因此，密码存储中的每个条目都存的是哈希值，并且“salt”本身以明文形式存储。

最后的建议。
* 避免使用过时的散列算法（例如 SHA-1、MD5 等）
* 阅读 [伪随机生成器部分][1]。

下面是一个简单的示例：
```go
package main

import (
    "crypto/rand"
    "crypto/sha256"
    "database/sql"
    "context"
    "fmt"
)

const saltSize = 32

func main() {
    ctx := context.Background()
    email := []byte("john.doe@somedomain.com")
    password := []byte("47;u5:B(95m72;Xq")

    // create random word
    salt := make([]byte, saltSize)
    _, err := rand.Read(salt)
    if err != nil {
        panic(err)
    }

    // let's create SHA256(salt+password)
    hash := sha256.New()
    hash.Write(salt)
    hash.Write(password)
    h := hash.Sum(nil)

    // this is here just for demo purposes
    //
    // fmt.Printf("email   : %s\n", string(email))
    // fmt.Printf("password: %s\n", string(password))
    // fmt.Printf("salt    : %x\n", salt)
    // fmt.Printf("hash    : %x\n", h)

    // you're supposed to have a database connection
    stmt, err := db.PrepareContext(ctx, "INSERT INTO accounts SET hash=?, salt=?, email=?")
    if err != nil {
        panic(err)
    }
    result, err := stmt.ExecContext(ctx, h, salt, email)
    if err != nil {
        panic(err)
    }

}
```

但是，这种方法有几个缺陷，不能在实际项目里使用。在这里展示只是为了用一个实际的例子来说明理论，下一节将解释如何在现实生活中正确地为密码加盐。

## 安全存储密码：实践

密码学中最重要的格言之一是：**永远不要公布自己的加密方式**，这样做可能会使整个应用程序处于危险之中。这是一个敏感而复杂的话题，我们期望的是密码学提供经过的工具和标准都是经过专家审查和批准的。因此，重要的是使用它们而不是试图重新造轮子。

在密码存储的情况下，[OWASP][2]推荐的哈希算法有[`bcrypt`][2]、[`PDKDF2`][3]、[`Argon2`][4]和[`scrypt` ][5]，这些是以健壮的方式处理散列和加盐密码。 Go 作者为密码学提供了一个扩展包，它不是标准库的一部分。它为大多数上述算法提供了健壮的实现。它可以使用`go get`下载：

```
go get golang.org/x/crypto
```

下面的例子展示了如何使用 `bcrypt` ，这对于大多数情况应该已经足够了。`bcrypt` 的优点是使用更简单，因此不易出错：

```go
package main

import (
    "database/sql"
    "context"
    "fmt"

    "golang.org/x/crypto/bcrypt"
)

func main() {
    ctx := context.Background()
    email := []byte("john.doe@somedomain.com")
    password := []byte("47;u5:B(95m72;Xq")

    // Hash the password with bcrypt
    hashedPassword, err := bcrypt.GenerateFromPassword(password, bcrypt.DefaultCost)
    if err != nil {
        panic(err)
    }

    // this is here just for demo purposes
    //
    // fmt.Printf("email          : %s\n", string(email))
    // fmt.Printf("password       : %s\n", string(password))
    // fmt.Printf("hashed password: %x\n", hashedPassword)

    // you're supposed to have a database connection
    stmt, err := db.PrepareContext(ctx, "INSERT INTO accounts SET hash=?, email=?")
    if err != nil {
        panic(err)
    }
    result, err := stmt.ExecContext(ctx, hashedPassword, email)
    if err != nil {
        panic(err)
    }
}
```

`Bcrypt` 还提供了一种简单而安全的方式来比较明文密码使用已经散列的密码：

```go

ctx := context.Background()

// credentials to validate
email := []byte("john.doe@somedomain.com")
password := []byte("47;u5:B(95m72;Xq")

// fetch the hashed password corresponding to the provided email
record := db.QueryRowContext(ctx, "SELECT hash FROM accounts WHERE email = ? LIMIT 1", email)

var expectedPassword string
if err := record.Scan(&expectedPassword); err != nil {
    // user does not exist

    // this should be logged (see Error Handling and Logging) but execution
    // should continue
}

if bcrypt.CompareHashAndPassword(password, []byte(expectedPassword)) != nil {
    // passwords do not match

    // passwords mismatch should be logged (see Error Handling and Logging)
    // error should be returned so that a GENERIC message "Sign-in attempt has
    // failed, please check your credentials" can be shown to the user.
}
```

如果对官方 `Bcrypt` 的参数或者选项不满意，那么还可以使用第三方的包

* [passwd][6] - 一个提供密码哈希和比较的 Go 包，它支持原始的 go bcrypt 实现、argon2、scrypt、参数屏蔽和密钥（不可破解）哈希。

[^1]: Hashing functions are the subject of Collisions but recommended hashing functions have a really low collisions probability

[1]: ../cryptographic-practices/pseudo-random-generators.md
[2]: https://www.owasp.org/index.php/Password_Storage_Cheat_Sheet
[3]: https://godoc.org/golang.org/x/crypto/bcrypt
[4]: https://github.com/p-h-c/phc-winner-argon2
[5]: https://godoc.org/golang.org/x/crypto/pbkdf2
[6]: https://github.com/ermites-io/passwd
