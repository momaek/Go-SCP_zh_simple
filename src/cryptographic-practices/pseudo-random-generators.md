伪随机发生器
========================

在 OWASP Secure Coding Practices 中，您会发现一个看似非常复杂的准则：“_所有随机数、随机文件名、随机 GUID 和随机字符串应使用加密模块批准的随机数生成器生成，当这些随机值是预期的不可猜测_”，所以让我们讨论“随机数”。

密码学依赖于一些随机性，但为了正确起见，大多数编程语言提供的是开箱即​​用的伪随机数生成器：例如，[Go 的 math/rand][1] 也不例外。

您应该仔细阅读文档，当它指出“_顶级函数，例如 Float64 和 Int，使用默认共享源，每次运行程序时都会生成 **确定性序列** 值。_”（[源][2])

这到底是什么意思呢？让我们来看看：


```go
package main

import "fmt"
import "math/rand"

func main() {
    fmt.Println("Random Number: ", rand.Intn(1984))
}
```

Running this program several times will lead exactly to the same
number/sequence, but why?

```bash
$ for i in {1..5}; do go run rand.go; done
Random Number:  1825
Random Number:  1825
Random Number:  1825
Random Number:  1825
Random Number:  1825
```

因为 [Go 的 math/rand][1] 是一个确定性的伪随机数生成器。与许多其他类似，它使用称为种子的来源。该种子 ** 完全** 负责确定性伪随机数生成器的随机性。如果它是已知的或可预测的，那么生成的数列也会发生同样的情况。

通过使用 [math/rand Seed 函数][3]，我们可以很容易地“修复”这个例子，为每个程序执行获得五个不同的预期值。但是因为我们在加密实践部分，我们应该遵循 [Go 的 crypto/rand 包] [4]。

```go
package main

import "fmt"
import "math/big"
import "crypto/rand"

func main() {
    rand, err := rand.Int(rand.Reader, big.NewInt(1984))
    if err != nil {
        panic(err)
    }

    fmt.Printf("Random Number: %d\n", rand)
}
```


您可能会注意到运行 [crypto/rand][4] 比 [math/rand][1] 慢，但这是意料之中的，因为最快的算法并不总是最安全的。 Crypto 的兰特实施起来也更安全。这方面的一个例子是你 _CANNOT_ 种子加密/兰特，因为库为此使用操作系统随机性，防止开发人员误用。

```bash
$ for i in {1..5}; do go run rand-safe.go; done
Random Number: 277
Random Number: 1572
Random Number: 1793
Random Number: 1328
Random Number: 1378
```

如果您对如何利用此漏洞感到好奇，请想一想如果您的应用程序在用户注册时创建默认密码会发生什么，通过计算使用 [Go's math/rand][1] 生成的伪随机数的哈希值，如图所示在第一个例子中。

是的，您猜对了，您将能够预测用户的密码！

[1]: https://golang.org/pkg/math/rand/
[2]: https://golang.org/pkg/math/rand/#pkg-overview
[3]: https://golang.org/pkg/math/rand/#Seed
[4]: https://golang.org/pkg/crypto/rand/
