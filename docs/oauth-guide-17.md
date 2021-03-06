# 使用PKCE保护移动应用程序

**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com)）**

用于代码交换的证明密钥（PKCE，发音为pixie）扩展描述了一种,公共客户端用于减轻拦截授权代码的威胁的技术。该技术涉及客户端首先创建秘钥，然后在交换访问令牌的授权码时再次使用该秘钥。这样，如果代码被截获，它将不能被使用，因为令牌请求依赖于初始秘钥。

完整规范以[RFC7636](https://tools.ietf.org/html/rfc7636)的形式提供。我们将在下面介绍协议的摘要。

### 授权请求

当原生应用程序开始授权请求时，客户端首先创建所谓的“代码验证程序” ，而不是立即启动浏览器。这是使用随机字符串A-Z，a-z，0-9和标点字符-._~（连字符，期间，下划线和波浪线）来进行加密，长字符43和128之间。

一旦应用程序生成了代码验证程序，它就会使用它来创建代码验证。对于可以执行SHA256哈希的设备，代码验证是代码验证程序的SHA256哈希的BASE64-URL编码字符串。允许无法执行SHA256哈希的客户端使用普通代码验证程序字符串作为验证。

现在客户端有一个代码验证字符串，它包含了一个参数，该参数指示用于生成验证的方法（plain或S256）以及授权请求的标准参数。这意味着完整的授权请求将包括以下参数。

> **response_type = code** - 表示您的服务器希望收到授权代码
> **client_id =** - 首次创建应用程序时收到的客户端ID
> **redirect_uri =** - 表示授权完成后返回用户的URL，例如org.example.app://redirect
> **state = 1234zyx** - 应用程序生成的随机字符串，稍后您将验证该字符串
> **code_challenge = XXXXXXXXX** - 如前所述生成的代码验证
> **code_challenge_method = S256** - plain或者S256，取决于验证是普通验证者字符串还是字符串的SHA256哈希。如果省略此参数，则服务器将采用plain。

授权服务器应识别code_challenge请求中的参数，并将其与其生成的授权码相关联。将其与授权码一起存储在数据库中，或者如果您使用自编码授权码，则可以将其包含在代码本身中。服务器正常返回授权码，并且不包括返回数据中的验证。

#### 错误响应

授权服务器可以要求公共客户端必须使用PKCE扩展。这实际上是允许公共客户端在不使用客户端密钥的情况下拥有安全授权流的唯一方法。由于对应于公共客户端，授权服务器应该知道特定客户端ID，因此它可以拒绝对不包含代码验证的公共客户端的授权请求。

如果授权服务器要求公共客户端使用PKCE，并且授权请求缺少代码验证，则服务器应返回错误响应，`error=invalid_request`并且应该使用`error_description`或者`error_uri`解释错误的性质。

### 授权码交换

本机应用程序将用授权码交换访问令牌。除了授权码请求中定义的参数之外，客户端还将发送`code_verifier`参数。完整的访问令牌请求将包括以下参数：

> **grant_type = authorization_code** - 表示此令牌请求的授权类型
> **code** - 客户端将发送它在重定向中获得的授权代码
> **redirect_uri** - 初始授权请求中使用的重定向URL
> **client_id** - 应用程序注册的客户端ID
> **code_verifier** - 应用程序最初在授权请求之前生成的PKCE请求的代码验证程序。

除了验证标准参数之外，授权服务器还将验证`code_verifier`请求中的内容。由于`code_challenge`和`code_challenge_method`用授权码相关联，因此服务器应该已经知道使用哪种方法（普通或SHA256）验证`code_verifier`。

如果方法是plain，那么授权服务器只需要检查提供的`code_verifier`是否与预期的`code_challenge`字符串匹配。

如果方法是S256，那么授权服务器应该使用提供的`code_verifier`, 并使用客户端最初使用的相同方法对其进行转换。这意味着计算其SHA256哈希值并对其进行base64-url编码，然后将其与存储的`code_challenge`字符串进行比较。

如果验证程序与期望值匹配，则服务器可以正常继续，发出访问令牌并进行适当响应。如果出现问题，则服务器会响应`invalid_grant`错误。

PKCE扩展不会添加任何新响应，因此即使授权服务器不支持，客户端也可以始终使用PKCE扩展。