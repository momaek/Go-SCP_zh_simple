伪随机生成器
========================

在本书中，你会发现一些看起来特别复杂的示例："_为了让这些个随机数，随机文件名，随机 GUIDs 和随机字符串的值不可预测，我们应该使用通过加密模块认证的随机数生成器_"，所以接下来我们来讨论一下"随机数"。

密码学或多或少都依赖一些随机性，所以大多数的编程语言都提供了一些开箱即用的伪随机数生成器，例如 Golang 的 [math/rand][1] 这个库。

你在使用 Golang 的这个库的时候，需要仔细阅读它的文档，文档中声明了 **库级别的顶层函数，例如 Float64, Int 这些函数默认使用的是一个共享的源，这个源在程序每次运行的时候都会生成一个确定的序列值，这就导致了生成的随机数是可以预测的**([source][2])

看下面这段代码你就知道这个具体是什么意思了:

```go
package main

import "fmt"
import "math/rand"

func main() {
    fmt.Println("Random Number: ", rand.Intn(1984))
}
```

多运行几次这个段代码，我们得到的是完全相同的输出值。为什么呢？

```bash
$ for i in {1..5}; do go run rand.go; done
Random Number:  1825
Random Number:  1825
Random Number:  1825
Random Number:  1825
Random Number:  1825
```

因为 Golang 的 [math/rand][1] 是一个确定的伪随机生成器，与其他伪随机生成器一样，它使用种子来作为源，什么意思呢？这个种子是用来确认为随机算法的随机性的。如果种子是已知的或者可以预测的，那么生成的随机数也就是可以预测了。

不过我们可以用 [math/rand 的 Seed 函数][3] 来修复这个问题，从而每次运行都得到不同的输出值。由于目前我们在是密码实践的章节，所以让我们来看看 Golang 的 [crypto/rand][4]


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

你可能发现 [crypto/rand][4] 比 [math/rand][1] 要慢，不过这是预期的，因为最快的算法并不总是最安全的。 crypto 库的随机相比于 math 库的随机来说是一个安全性更好的实现。一个示例就是：你是不能指定 crypto/rand 的种子的，这个库用了操作系统的随机种子，目的是防止开发者误用。

```bash
$ for i in {1..5}; do go run rand-safe.go; done
Random Number: 277
Random Number: 1572
Random Number: 1793
Random Number: 1328
Random Number: 1378
```

如果你好奇这种为随机的漏洞会被怎么利用的话，那么请你想一想这样的场景：你的程序在用户注册的时候就为这个用户生成一个默认密码，而且每个用户的默认密码都是一样的。 会发生什么呢？ 

这个就是使用 Golang 的 [math/rand][1] 可能发生的事情 - 你可以猜到用户的密码！！


[1]: https://golang.org/pkg/math/rand/
[2]: https://golang.org/pkg/math/rand/#pkg-overview
[3]: https://golang.org/pkg/math/rand/#Seed
[4]: https://golang.org/pkg/crypto/rand/
