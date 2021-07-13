密码实践
======================

首先声明一点：**哈希和加密是两件不同的事情**。这是一个普遍的误解。然而在大多数时候，我们通常会以不正确的方式来交替使用哈希和加密，但它们是不同的概念，它们各自的用途也不一样。

哈希是由（哈希）函数从源数据生成的字符串或数字：

```go
hash := F(data)
```

哈希具有固定的长度，其值随输入的微小变化而变化很大（但是仍然可能发生冲突）。一个好的哈希算法不会允许哈希的结果可逆[^1]。 MD5 是最流行的哈希算法，但从安全性的方面来说 [BLAKE2][4] 被认为是最强大和最灵活的。

Go 官方补充库（ x ）提供 [BLAKE2b [5]（或仅 BLAKE2）和 [BLAKE2s][6] 实现：前者针对 64 位平台进行了优化，后者针对 8 位到 32 位平台进行了优化。如果 BLAKE2 不能用，那么 SHA-256 是正确的选择。

请注意，慢是加密哈希算法所需要的。随着时间的推移，计算机变得越来越快，这意味着随着时间的推移，攻击者可以尝试越来越多的潜在密码（参见 [Credential Stuffing][7] 和 [Brute-force Attack][8]）。为了反击，哈希函数应该_固有地慢_，例如多迭代 10000 次。为什么要慢呢？这样可以增加攻击者每一次攻击的时间，从而减少攻击者可以尝试的密码数量。

```go
package main

import "fmt"
import "io"
import "crypto/md5"
import "crypto/sha256"
import "golang.org/x/crypto/blake2s"

func main () {
        h_md5 := md5.New()
        h_sha := sha256.New()
        h_blake2s, _ := blake2s.New256(nil)
        io.WriteString(h_md5, "Welcome to Go Language Secure Coding Practices")
        io.WriteString(h_sha, "Welcome to Go Language Secure Coding Practices")
        io.WriteString(h_blake2s, "Welcome to Go Language Secure Coding Practices")
        fmt.Printf("MD5        : %x\n", h_md5.Sum(nil))
        fmt.Printf("SHA256     : %x\n", h_sha.Sum(nil))
        fmt.Printf("Blake2s-256: %x\n", h_blake2s.Sum(nil))
}
```

输出

```
MD5        : ea9321d8fb0ec6623319e49a634aad92
SHA256     : ba4939528707d791242d1af175e580c584dc0681af8be2a4604a526e864449f6
Blake2s-256: 1d65fa02df8a149c245e5854d980b38855fd2c78f2924ace9b64e8b21b3f2f82
```
**注意**：要运行源代码示例，你需要运行 `go get golang.org/x/crypto/blake2s`。

另一方面，加密使用密钥把数据转换成可变长度的数据：
```go
encrypted_data := F(data, key)
```

与哈希不同的是，我们可以通过 `encrypted_data` 解密出原始数据的内容：
```go
data := F⁻¹(encrypted_data, key)
```

只要你需要通信或存储敏感数据，你或其他人稍后需要访问这些数据以进行进一步处理，就应该使用加密。一个“简单”的加密用例是 HTTPS - 安全的超文本传输协议。

在对称密钥加密方面，AES 是事实上的标准。该算法与许多其他对称密码类似，可以以不同的模式实现。你会注意到在下面的代码示例中，使用了 GCM（伽罗华计数器模式），而不是更流行的（至少在密码学代码示例中）CBC/ECB。 GCM和CBC/ECB的主要区别在于前者是**认证**密码模式，即在加密阶段之后，在密文中添加一个认证标签，然后在**之前*进行验证* 对消息进行解密，确保消息未被篡改。
另外一种是通过公钥和私钥对的公钥加密或非对称加密技术，而在大多数情况下，公钥加密的性能低于对称密钥加密。因此比较常见的用例是用非对称加密来传输对称加密的密钥，这样后续就可以用对称加密来提高性能了。

AES 是 1990 年代的技术，Go 的作者们已经开始实现和支持更现代的对称加密算法，这些算法也提供身份验证，例如 chacha20poly1305。

Go 中另一个有趣的包是 x/crypto/nacl。这是对 Daniel J. Bernstein 博士的 NaCl 库的引用，这是一个非常流行的现代密码学库。 Go 中的 nacl/box 和 nacl/secretbox 是 NaCl 抽象的实现，用于为两个最常见的用例发送加密消息：

* 使用公钥密码术（nacl/box）在两方之间发送经过身份验证的加密消息
* 使用对称（又名密钥）密码术在两方之间发送经过身份验证的加密消息

如果它们适合你的用例，最好使用这些抽象之一而不是直接使用 AES。

下面的示例说明了使用基于 [AES-256][9]（*256 位/32 字节*）的密钥进行加密和解密，并明确分离了加密、解密和秘密生成等问题。 `secret` 方法是一个方便的选项，有助于生成一个秘密。源代码示例取自 [here][10]，但略有修改。

```go
package main

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "io"
    "log"
)

func encrypt(val []byte, secret []byte) ([]byte, error) {
    block, err := aes.NewCipher(secret)
    if err != nil {
        return nil, err
    }

    aead, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }

    nonce := make([]byte, aead.NonceSize())
    if _, err = io.ReadFull(rand.Reader, nonce); err != nil {
        return nil, err
    }

    return aead.Seal(nonce, nonce, val, nil), nil
}

func decrypt(val []byte, secret []byte) ([]byte, error) {
    block, err := aes.NewCipher(secret)
    if err != nil {
        return nil, err
    }

    aead, err := cipher.NewGCM(block)
    if err != nil {
        return nil, err
    }

    size := aead.NonceSize()
    if len(val) < size {
        return nil, err
    }

    result, err := aead.Open(nil, val[:size], val[size:], nil)
    if err != nil {
        return nil, err
    }

    return result, nil
}

func secret() ([]byte, error) {
    key := make([]byte, 16)

    if _, err := rand.Read(key); err != nil {
        return nil, err
    }

    return key, nil
}

func main() {
    secret, err := secret()
    if err != nil {
        log.Fatalf("unable to create secret key: %v", err)
    }

    message := []byte("Welcome to Go Language Secure Coding Practices")
    log.Printf("Message  : %s\n", message)

    encrypted, err := encrypt(message, secret)
    if err != nil {
        log.Fatalf("unable to encrypt the data: %v", err)
    }
    log.Printf("Encrypted: %x\n", encrypted)

    decrypted, err := decrypt(encrypted, secret)
    if err != nil {
        log.Fatalf("unable to decrypt the data: %v", err)
    }
    log.Printf("Decrypted: %s\n", decrypted)
}
```

```bash
Message  : Welcome to Go Language Secure Coding Practices
Encrypted: b46fcd10657f3c269844da5f824511a0e3da987211bc23e82a9c050a2be287f51bb41dd3546742442498ae9fcad2ce40d88625d1840c11096a55cb4f217382befbeb636e479cfecfcd3a
Decrypted: Welcome to Go Language Secure Coding Practices
```

请注意，你应该**建立并利用有关如何管理加密密钥的策略和流程**，保护**主密码免遭未经授权的访问**。 话虽如此，你的加密密钥不应硬编码在源代码中（如本例中所示）。

[Go 的加密包][1] 收集了常见的加密常量，但也有实现了加密算法的包，例如 [crypto/md5][2] 之一。

大多数现代密码算法已经在 https://godoc.org/golang.org/x/crypto 下实现，因此开发人员应该关注那些而不是 [`crypto/*` 包][1] 中的实现。

---

[^1]: Rainbow table attacks are not a weakness on the hashing algorithms.
[^2]: Consider reading the [Authentication and Password Management][3] section about "_strong one-way salted hashes_" for credentials.

[1]: https://golang.org/pkg/crypto/
[2]: https://golang.org/pkg/crypto/md5/
[3]: ../authentication-password-management/README.md
[4]: https://blake2.net/
[5]: https://godoc.org/golang.org/x/crypto/blake2b
[6]: https://godoc.org/golang.org/x/crypto/blake2s
[7]: https://www.owasp.org/index.php/Credential_stuffing
[8]: https://www.owasp.org/index.php/Brute_force_attack
[9]: https://en.wikipedia.org/wiki/Advanced_Encryption_Standard
[10]: http://www.inanzzz.com/index.php/post/f3pe/data-encryption-and-decryption-with-a-secret-key-in-golang
