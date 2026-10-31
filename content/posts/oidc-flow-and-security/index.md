---
title: "OIDC 详解：授权码流程，以及它如何保证登录安全"
date: 2026-10-31T20:00:00+08:00
draft: false
slug: "oidc-flow-and-security"
description: "从 OAuth 2.0 与 OIDC 的区别讲起，逐步拆解授权码 + PKCE 流程，再逐一说明 state、nonce、PKCE、ID Token 校验等机制各自防御什么攻击。"
tags: ["OIDC", "OAuth 2.0", "PKCE", "身份认证", "Web 安全"]
categories: ["安全"]
---

很多应用的“用 XX 账号登录”背后都是 OpenID Connect（OIDC）。整个过程看起来只是几次页面跳转，但每一步带的参数、每一处校验，都对应着一类真实存在的攻击。本文先讲清 OIDC 和 OAuth 2.0 的关系，再按时间顺序拆解目前最佳实践一致推荐的授权码 + PKCE 流程，最后逐一说明它靠哪些机制保证安全。

<!--more-->

文中的示例约定如下：客户端应用（RP）是 `https://app.example.com`，OpenID 提供方（OP）是 `https://op.example.com`，API 是 `https://api.example.com`，RP 的 `client_id` 是 `web-app`。流程以“有后端的 Web 应用”为例，它属于机密客户端（confidential client），能在服务器上安全保存自己的凭证。内容依据 OpenID Connect Core 1.0、RFC 7636、RFC 9700 和 RFC 10017 等规范。

## 一、先分清 OAuth 2.0 和 OIDC

OAuth 2.0 解决的是授权：用户同意之后，客户端拿到一个 access token，用它去调用某个 API。它在设计上并不回答“当前用户是谁”。

早年不少网站直接拿 OAuth 做登录：拿到 access token 后调一下“获取用户信息”接口，就认定用户已经登录。这里有个根本问题：access token 是给 API 看的，客户端无法确认这个 token 是不是签发给自己的。攻击者可以把其他应用拿到的 token 塞进来冒充用户，这类攻击叫令牌替换（token substitution）。

OIDC 在 OAuth 2.0 之上补了一层身份层。最核心的是 ID Token：一个由 OP 签名的 JWT，写明了“哪个用户、在什么时候、为哪个客户端完成了认证”。此外，OIDC 还标准化了 `openid` 等 scope、用户信息声明（claims）、UserInfo 端点和 Discovery 自动发现机制。

两种令牌的区别值得先记住：

| | ID Token | Access Token |
|---|---|---|
| 给谁看 | 客户端（RP） | 资源服务器（API） |
| 回答的问题 | 用户是谁、何时以何种方式完成了认证 | 持有者可以访问哪些资源 |
| 格式 | 规范规定为签名的 JWT | 规范不限定，可以是不透明字符串或 JWT |
| 受众 `aud` | RP 的 `client_id` | 目标 API |
| 正确用法 | RP 验证后建立登录会话，不要发给 API | 放进 `Authorization: Bearer` 调用 API，RP 不应据此判断用户身份 |

## 二、角色、端点和两条通道

一次 OIDC 登录涉及四方：

- **用户 / 浏览器**（User Agent）：在 RP 和 OP 之间来回跳转。
- **客户端应用**（Relying Party，RP）：需要知道“用户是谁”的应用。
- **OpenID 提供方**（OpenID Provider，OP）：负责认证用户、签发令牌，比如企业的统一身份平台或各类社交登录服务。
- **资源服务器**（Resource Server）：接受 access token 的 API。

OP 对外提供若干端点。RP 一般不需要手动配置它们，而是读取 Discovery 文档 `https://op.example.com/.well-known/openid-configuration`：

```json
{
  "issuer": "https://op.example.com",
  "authorization_endpoint": "https://op.example.com/authorize",
  "token_endpoint": "https://op.example.com/token",
  "userinfo_endpoint": "https://op.example.com/userinfo",
  "jwks_uri": "https://op.example.com/jwks",
  "response_types_supported": ["code"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256", "ES256"],
  "code_challenge_methods_supported": ["S256"],
  "authorization_response_iss_parameter_supported": true
}
```

其中 `issuer` 是 OP 的身份标识，之后 ID Token 里的 `iss` 必须与它完全一致；`jwks_uri` 提供验证签名用的公钥。

理解 OIDC 安全设计的关键，是分清两条通道：

{{< figure src="oidc-channels.svg" alt="OIDC 的前端通道与后端通道示意图" caption="图 1　前端通道经过浏览器，后端通道是服务器之间直连" >}}

- **前端通道**：经浏览器重定向传递的数据都在 URL 里，会进入浏览器历史、服务器访问日志和 `Referer` 头，也可能被浏览器扩展、恶意脚本或手机上的恶意 App 截获，还可能被攻击者伪造。
- **后端通道**：RP 的服务器通过 TLS 直接访问 OP 的令牌端点。内容不经过浏览器，OP 的身份由 TLS 证书保证，RP 的身份由客户端凭证保证。

整套协议的设计原则可以概括为一句话：真正有价值的东西（令牌，以及能换到令牌的证明）只走后端通道；前端通道上只放一次性、短时效、单独拿到也没用的东西。

## 三、授权码 + PKCE 流程逐步拆解

先看全貌，下面按图中的编号逐步展开。

{{< figure src="oidc-sequence.svg" alt="OIDC 授权码流程（含 PKCE）时序图" caption="图 2　OIDC 授权码流程（含 PKCE）" >}}

### 第 1 步之后：RP 准备三个随机值

用户点击“登录”后，RP 在跳转之前先生成三个高熵随机值，保存在与这个浏览器绑定的服务端会话里：

```python
import base64, hashlib, secrets

def b64url(raw: bytes) -> str:
    return base64.urlsafe_b64encode(raw).rstrip(b"=").decode("ascii")

state = secrets.token_urlsafe(32)                # 防 CSRF
nonce = secrets.token_urlsafe(32)                # 绑定 ID Token
code_verifier = b64url(secrets.token_bytes(32))  # PKCE 原文，43 个字符
code_challenge = b64url(hashlib.sha256(code_verifier.encode("ascii")).digest())
```

按 RFC 7636 的要求，`code_verifier` 是 43 到 128 个字符的随机串，`code_challenge` 是它的 SHA-256 哈希再做 Base64URL 编码，也就是 `S256` 方法。可以用 RFC 7636 附录中的示例自测：`dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk` 计算后应得到 `E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM`。

### 第 2、3 步：跳转到授权端点

RP 返回 302，让浏览器带着授权请求访问 OP。为便于阅读，下面把参数分行写，实际是一行 URL：

```http
GET /authorize?response_type=code
  &client_id=web-app
  &redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback
  &scope=openid%20profile%20email
  &state=kq8nY3pT...
  &nonce=Zt2Wv9xQ...
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256 HTTP/1.1
Host: op.example.com
```

- `response_type=code`：使用授权码流程。
- `scope` 必须包含 `openid`。缺少它时这就不算 OIDC 请求，不能指望拿到 ID Token。
- `redirect_uri` 必须与在 OP 注册的值逐字符一致。
- `state`、`nonce`、`code_challenge` 是上一步生成的值；`code_verifier` 原文不会出现在这里。
- 还可以按需加上 `prompt`、`max_age`、`login_hint`、`acr_values` 等参数，控制是否强制重新登录、要求什么样的认证强度。

### 第 3、4 步之间：用户在 OP 登录并授权

用户的密码、MFA、通行密钥（passkey）只交给 OP，RP 从头到尾接触不到用户凭证。这本身就是一项重要的安全收益：认证策略集中在 OP 一处实施，RP 不必各自存储和校验密码。由应用直接收集用户名和密码的“密码模式”被 RFC 9700 禁止使用，原因之一就在这里。

如果用户之前已经同意过授权，OP 可能直接跳过同意页。

### 第 4、5 步：code 经浏览器回到 RP

```http
HTTP/1.1 302 Found
Location: https://app.example.com/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=kq8nY3pT...&iss=https%3A%2F%2Fop.example.com
```

`code` 只能用一次，有效期很短（RFC 6749 建议不超过 10 分钟），并且与 `client_id`、`redirect_uri` 和 `code_challenge` 绑定。`iss` 参数由 RFC 9207 定义，OP 在 Discovery 中声明支持时才会返回，用来防御多 OP 场景下的混淆攻击（见 4.8 节）。

RP 处理回调的顺序很重要：

1. 用 `state` 找到会话中保存的登录事务，比对一致后立即删除，保证一次性使用。不一致或找不到就直接拒绝。
2. 如果回调里带了 `iss`，确认它正是发起这次登录的 OP。
3. 如果回调里是 `error`（比如用户拒绝了授权），同样先校验 `state`，再展示错误。
4. 以上都通过之后，才去换令牌。

回调页面本身也要小心：不加载第三方资源，设置 `Referrer-Policy: no-referrer`，处理完立即重定向，避免带 code 的 URL 经 `Referer` 或日志泄露。

### 第 6 步：后端用 code 换令牌

```http
POST /token HTTP/1.1
Host: op.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <base64(client_id:client_secret)>

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

OP 收到请求后依次检查：

- 客户端认证是否通过（机密客户端）；
- `code` 是否存在、未过期、未被使用过，并且正是签发给这个客户端的；
- `redirect_uri` 是否与授权请求时的一致；
- `BASE64URL(SHA-256(code_verifier))` 是否等于授权请求里的 `code_challenge`。

任何一项不满足，都会返回 `invalid_grant` 等错误。如果同一个 code 被重复使用，RFC 6749 要求 OP 拒绝请求，并建议吊销之前用这个 code 签发的所有令牌。

单页应用、移动 App 这类公共客户端无法保存密钥，请求里没有 `Authorization` 头，只在请求体中带上 `client_id`，防止授权码被滥用就全靠 PKCE。机密客户端则建议优先使用 `private_key_jwt` 或 mTLS 这类非对称方式认证，它们比共享的 `client_secret` 更难泄露。

### 第 7 步：令牌响应

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6ImF0K2p3dCIsImtpZCI6IjIwMjYtMTAifQ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "8xLOxBtZp8...",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjIwMjYtMTAiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL29w..."
}
```

`Cache-Control: no-store` 是 RFC 6749 的要求，防止令牌被缓存。`refresh_token` 是可选的，取决于 OP 的配置和请求的 scope（例如 `offline_access`）。

### 第 7 步之后：验证 ID Token

签名的 ID Token 由三段 Base64URL 编码的内容组成：头部、载荷和签名。解码前两段：

```json
{ "alg": "RS256", "kid": "2026-10", "typ": "JWT" }
```

```json
{
  "iss": "https://op.example.com",
  "sub": "248289761001",
  "aud": "web-app",
  "exp": 1793462400,
  "iat": 1793458800,
  "auth_time": 1793458788,
  "nonce": "Zt2Wv9xQ...",
  "email": "alice@example.com",
  "email_verified": true
}
```

按 OIDC Core 第 3.1.3.7 节，RP 至少要做以下检查，任何一项失败都必须拒绝：

1. **签名**：按头部的 `kid` 从 `jwks_uri` 取对应公钥验签。`alg` 只接受事先约定的算法，拒绝 `none`，也不能让攻击者把 `RS256` 改成 `HS256` 骗过验签（算法混淆）。规范允许授权码流程用 TLS 服务器校验代替验签，因为 ID Token 是 RP 直接从令牌端点拿到的；但主流库都会验签，建议保留。
2. **`iss`**：必须与 Discovery 中的 `issuer` 完全一致。
3. **`aud`**：必须包含自己的 `client_id`；如果还包含 RP 不信任的其他受众，也要拒绝。
4. **`exp` 和 `iat`**：令牌未过期（可以容许几分钟的时钟偏差），签发时间也不能离现在太远。
5. **`nonce`**：必须与会话中保存的值一致，比对后删除。
6. **`auth_time` 和 `acr`**：如果授权请求用了 `max_age` 或 `acr_values`，检查认证时间和认证强度是否满足要求。

验证通过后，用 `iss` + `sub` 作为用户的唯一标识。OIDC Core 明确指出，只有这两个声明组合在一起，才是 RP 可以依赖的稳定用户标识。不要用 `email` 关联本地账户：邮箱可能变更，也可能根本没经过验证。确实要按邮箱合并账户时，至少要求 `email_verified` 为 `true`，并且只信任你确认会验证邮箱的 OP。

### 第 8 步：建立 RP 自己的会话

ID Token 验证通过后，RP 建立自己的登录会话，再把浏览器重定向回原页面：

- 登录成功后更换会话 ID，防止会话固定攻击。
- 会话 Cookie 设置 `Secure`、`HttpOnly` 和 `SameSite`，名称使用 `__Host-` 前缀，防止被子域名覆盖。
- 需要保留的令牌只保存在服务端会话里，不下发给浏览器。

有个容易踩的坑：保存登录事务（`state`、`nonce`、`code_verifier`）的 Cookie 必须能在回调时被带上。回调是从 OP 跨站跳回来的顶层导航，`SameSite=Strict` 的 Cookie 不会随之发送，所以这个 Cookie 应使用 `SameSite=Lax`。如果使用 `response_mode=form_post`，回调变成跨站 POST，连 `Lax` 的 Cookie 也不会发送，就需要 `SameSite=None; Secure`。

### 第 9、10 步：调用 API

```http
GET /api/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6ImF0K2p3dCIsImtpZCI6IjIwMjYtMTAifQ...
```

资源服务器要自己验证 access token：JWT 格式（RFC 9068）的令牌在本地验签，不透明令牌通过 OP 的内省端点（RFC 7662）查询。无论哪种方式，都要检查 `iss`、`exp`，确认 `aud` 是本 API，`scope` 覆盖这次操作。API 不应接受 ID Token 作为访问凭证。

access token 过期后，RP 用 `refresh_token` 向令牌端点换取新的令牌，用户无需重新登录。需要更多用户信息时，RP 还可以带着 access token 调用 UserInfo 端点。

### 只做登录：不调用 API 的情况

很多应用接入 OIDC 只是为了实现“用 XX 账号登录”，并不需要调用任何 API。这时流程到第 8 步就结束了，第 9、10 步可以省略，OP 只承担身份认证的角色：RP 验证 ID Token，用 `iss` + `sub` 找到或创建本地用户，再建立自己的会话。此后用户访问 RP，校验的都是 RP 自己的会话，与 OP 签发的令牌无关。

因此，OP 返回的令牌大多用不着保存：

- **`access_token`**：OAuth 2.0 规定令牌响应必须包含它，所以 OP 照样会返回，RP 用不到就直接丢弃。常见的例外是 ID Token 里的声明不够用（比如缺少邮箱或昵称），需要在登录时用它调用一次 UserInfo 端点，用完再丢。这时 UserInfo 响应中的 `sub` 必须与 ID Token 中的 `sub` 完全一致，否则不能使用这份响应（OIDC Core 第 5.3.2 节）。
- **`refresh_token`**：很多 OP 只在请求了 `offline_access` 时才会签发，不请求即可；即使拿到了，也可以不保存。
- **`id_token`**：验证通过后就可以丢弃。但如果要支持“退出 RP 时同时退出 OP”（RP-Initiated Logout），规范建议注销请求携带 `id_token_hint`，有些 OP 还要求必须携带，这时就把它保存在服务端会话里。

scope 也按需申请：只需要身份时，`openid` 就够了；要展示用户资料，再加上 `profile`、`email`。还要注意，省掉的只是第 9、10 步：`state`、`nonce`、PKCE 和 ID Token 的各项校验一项都不能少，它们保护的正是登录本身。

这样做的代价是，RP 的本地会话与用户在 OP 的会话从此脱钩：用户在 OP 被禁用、修改密码或者退出登录，RP 的会话都不会随之失效，而是一直持续到它自己过期。常见的应对方式有：

- 本地会话的有效期不要设得太长，到期后重新跳转到 OP 认证。如果用户在 OP 的会话还在，OP 通常会直接跳回 RP，用户几乎无感。
- 需要让会话及时失效的场景（比如员工离职后立即断开访问），可以接入 OP 的 Back-Channel Logout：由 OP 通知 RP 注销对应的会话，为此要在本地会话里记下 ID Token 中的 `sid`，或者 `iss` + `sub`。也可以保留 `refresh_token` 并定期刷新，OP 拒绝刷新，通常就意味着账户已被禁用或授权已被撤销。

## 四、OIDC 如何保证安全

OIDC 的安全性不靠某一个参数，而是几层机制叠加：

1. **通道隔离**：令牌只走后端通道，前端通道只放一次性的 code。
2. **会话绑定**：用 `state`、`nonce`、PKCE 三个随机值，把前端通道上的每一段跳转绑定到发起登录的那个浏览器会话。
3. **令牌自证**：用签名和 `iss`、`aud`、`exp` 等声明，证明 ID Token 由谁签发、发给谁、何时有效。
4. **纵深防御**：精确匹配回调地址、客户端认证、发送方约束令牌、安全地存储令牌。

先看三个随机值分别在哪里发出、在哪里校验：

{{< figure src="oidc-bindings.svg" alt="state、nonce、PKCE 的发出与校验位置" caption="图 3　三个随机值都随授权请求发出，分别在回调、令牌交换和 ID Token 验证时核对" >}}

### 4.1 令牌只走后端通道

OAuth 2.0 早期还有一种隐式流程（implicit flow）：OP 直接把 access token 放在重定向 URL 的片段（`#` 之后）里交给浏览器。令牌因此会暴露在浏览器历史、页面脚本和各种日志中，客户端也无法确认收到的令牌是不是发给自己的。RFC 9700 规定客户端不应再使用隐式流程；2026 年 8 月发布的 RFC 10017 进一步要求，浏览器应用必须使用授权码流程，不得使用隐式流程获取 access token。

在授权码流程里，经过前端通道的只有 code。它只能用一次、几分钟内过期、与客户端和回调地址绑定；配合 PKCE 之后，单独拿到它也换不到令牌。

### 4.2 PKCE：截获了 code 也换不到令牌

{{< figure src="oidc-pkce.svg" alt="PKCE 如何挫败授权码截获" caption="图 4　攻击者只截获到 code，拿不出匹配的 code_verifier" >}}

PKCE（Proof Key for Code Exchange，RFC 7636）最初是为移动应用设计的：手机上的恶意 App 可以注册与正规 App 相同的自定义 URL scheme，从而截获回调里的 code。现在它适用于所有类型的客户端，原理是把流程的前后两段绑在一起：

- 授权请求（前端通道）只携带 `code_challenge`，也就是 `code_verifier` 的哈希。哈希不可逆，看到它也算不出原文。
- 换令牌（后端通道）时必须提交 `code_verifier` 原文，OP 重新计算哈希并比对。
- 攻击者从日志、`Referer` 或恶意 App 截获的只有 code，拿不出匹配的 `code_verifier`，换令牌的请求就会被拒绝。

PKCE 同样能挡住授权码注入：攻击者把偷来的 code 塞进自己在 RP 上的登录会话，RP 会用这个会话里的 `code_verifier` 去换令牌，与 code 绑定的 `code_challenge` 对不上，交换失败。

使用 PKCE 时要注意几点：

- 只用 `S256`。`plain` 方式会让 `code_verifier` 原文出现在前端通道上，保护作用大打折扣。RFC 7636 规定，能使用 `S256` 的客户端必须使用它。
- OP 必须防范 PKCE 降级：授权请求里没有 `code_challenge`，令牌请求里却出现了 `code_verifier`，这样的请求应当拒绝。客户端可以检查 Discovery 中的 `code_challenge_methods_supported`，确认 OP 支持 PKCE。
- 按 RFC 9700，公共客户端必须使用 PKCE，机密客户端也推荐使用。最省心的做法是所有客户端都用。
- 移动端应该通过系统浏览器（如 ASWebAuthenticationSession、Custom Tabs）发起授权，不要用内嵌 WebView（RFC 8252）。

### 4.3 state：防登录 CSRF

登录 CSRF 的攻击过程是这样的：攻击者用自己的账号走完授权，拿到属于自己的 code 但不去兑换；然后诱导受害者的浏览器访问 `https://app.example.com/callback?code=<攻击者的 code>`。如果 RP 不做检查，受害者就会以攻击者的身份登录，之后保存的银行卡、上传的文件、搜索记录，都会进入攻击者的账户。

`state` 的防御方式很直接：它与发起登录的浏览器会话绑定，回调时 RP 拿它和会话里保存的值比对。受害者的会话里没有攻击者那次请求的 `state`，回调就会被拒绝。

RFC 9700 指出，如果已确认 OP 支持 PKCE，PKCE 本身也能防 CSRF；在 OIDC 中，`nonce` 也能起到同样的作用。但 `state` 仍然值得保留：它在回调时就能拒绝请求，不必等到换完令牌；还可以作为服务端登录事务的索引，比如记录登录后要回到哪个页面。注意不要把跳转地址这类可被篡改的数据直接放进 `state`。

### 4.4 nonce：把 ID Token 绑定到这一次登录

`state` 保护的是回调这一跳，`nonce` 保护的是 ID Token 本身。OP 会把授权请求里的 `nonce` 原样写进签名的 ID Token，RP 验证时与会话里保存的值比对：

- 攻击者想让 RP 接受一个不属于这次登录的 ID Token，比如注入别人的 code，或者重放旧令牌，令牌里的 `nonce` 都会与受害者会话中的值对不上。
- `nonce` 位于受签名保护的载荷里，无法篡改。

在授权码流程中，`nonce` 按规范是可选的，但强烈建议使用；在 ID Token 直接经前端通道返回的隐式流程和混合流程中，它是必需的。

### 4.5 ID Token 的签名和声明：谁签发、发给谁、何时有效

ID Token 的每一项校验都对应一类攻击：

- **签名**证明令牌由 OP 签发且未被篡改。RP 应缓存 JWKS，遇到不认识的 `kid` 时再刷新（注意限流），以支持 OP 轮换密钥。算法必须来自白名单，否则可能遭遇 `alg=none` 或 RS256/HS256 混淆攻击。
- **`iss`** 保证令牌来自预期的 OP，接入多个 OP 时尤其重要。
- **`aud`** 保证令牌是发给自己的。这正是 OIDC 比“拿 access token 做登录”安全的地方：用户在某个恶意应用上登录，恶意应用拿到的 ID Token 的 `aud` 是它自己，拿到你的应用来用会被拒绝。
- **`exp` 和 `iat`** 限制令牌的有效时间，缩小重放窗口。
- **`iss` + `sub`** 作为用户标识，避免因为依赖可变或未经验证的 `email` 而导致账户被接管。

签名验证、算法选择、时钟偏差这些细节，都应交给成熟的 OIDC 库处理，不要自己手写 JWT 解析和校验。

### 4.6 redirect_uri 精确匹配

如果 OP 只按前缀或通配符匹配 `redirect_uri`，攻击者就能构造授权请求，让 code 被送到自己控制的地址，比如某个被接管的子域名，或 RP 站点上的开放重定向。对公共客户端来说，这时 PKCE 也帮不上忙：攻击者可以从头发起自己的授权请求，用自己生成的 `code_verifier` 去兑换。

因此：

- OP 必须对 `redirect_uri` 做精确的字符串匹配。RFC 9700 只允许一个例外：原生应用的 localhost 回调地址可以使用任意端口。
- RP 和 OP 都不能存在开放重定向，也就是根据 URL 参数把浏览器跳转到任意地址的接口。
- 令牌请求里再次携带 `redirect_uri`，由 OP 核对它与授权请求时的一致。

### 4.7 客户端认证：证明来换令牌的就是这个 RP

机密客户端在令牌端点必须认证自己。这样即使 code 和 `code_verifier` 同时泄露，没有客户端凭证也换不到令牌。常见方式有：

- `client_secret_basic` 和 `client_secret_post`：使用共享密钥，最简单，但密钥一旦泄露就失去作用；
- `private_key_jwt`：用私钥签名一个 JWT 作为凭证，OP 只保存公钥；
- `tls_client_auth`：基于双向 TLS（RFC 8705）。

公共客户端（单页应用、移动 App）的代码会分发到用户手里，内置的密钥算不上秘密。RFC 10017 强烈建议，处理业务数据、敏感数据或个人数据的浏览器应用采用 BFF（Backend for Frontend）架构：由一个后端组件充当机密客户端，令牌只保存在后端，浏览器只持有 `HttpOnly` 会话 Cookie。

### 4.8 多个 OP 时的混淆攻击

如果 RP 同时接入多个 OP，比如同时支持两家的“一键登录”，就要防范混淆攻击（mix-up）：一个恶意或被攻破的 OP，可能诱使 RP 把从诚实 OP 拿到的 code 发到恶意 OP 的令牌端点，从而窃取 code。

RFC 9700 规定，能与多个 OP 交互的客户端必须防御混淆攻击。做法是：

- 在登录事务里记录这次是向哪个 OP 发起的；
- 回调时检查 RFC 9207 定义的 `iss` 参数，确认与记录一致；
- OP 不支持 `iss` 参数时，为每个 OP 使用不同的 `redirect_uri`。

### 4.9 令牌本身的保护

即使流程本身没有漏洞，令牌仍可能在传输或存储环节泄露，所以还需要几层止损措施：

- **全程 TLS**，令牌响应设置 `Cache-Control: no-store`。
- **access token 短有效期、最小 scope**，并把受众（`aud`）限定为具体的 API。
- **发送方约束**：DPoP（RFC 9449）或 mTLS（RFC 8705）把令牌绑定到客户端持有的密钥上，令牌被偷走也无法单独使用。
- **refresh token**：发给公共客户端时，必须每次使用后轮换，或者做发送方约束（RFC 9700）。启用轮换后，一旦发现已作废的旧令牌被再次使用，就说明令牌可能已经泄露，OP 应吊销当前有效的令牌。
- **浏览器端**：尽量不在 `localStorage` 里保存令牌，并用 CSP 等手段降低 XSS 风险。

## 五、攻击与防护速查表

| 攻击 | 攻击方式 | 主要防护 | 由谁检查 |
|---|---|---|---|
| 授权码截获 | 从日志、`Referer`、恶意 App 拿到 code | PKCE（S256）、code 一次性且短时效、客户端认证 | OP |
| 授权码注入 | 把偷来的 code 塞进攻击者自己的会话 | PKCE、`nonce` | OP、RP |
| 登录 CSRF | 让受害者以攻击者的身份登录 | `state`（或 PKCE、`nonce`） | RP |
| ID Token 重放或注入 | 重放旧令牌，或塞入不属于本次登录的令牌 | `nonce`、`exp`、`aud` | RP |
| 令牌伪造 | 自签 JWT、`alg=none`、算法混淆 | 验签、算法白名单、JWKS | RP |
| 令牌替换 | 用发给其他应用的令牌登录 | 校验 `aud`，不用 access token 判断身份 | RP |
| 重定向劫持 | 让 code 被送到攻击者控制的地址 | `redirect_uri` 精确匹配、消除开放重定向 | OP、RP |
| 混淆攻击（mix-up） | 接入多个 OP 时把 code 交给恶意 OP | `iss` 参数、每个 OP 独立的 `redirect_uri` | RP |
| 令牌泄露后被滥用 | 通过 XSS、日志、中间人窃取令牌 | TLS、短有效期、DPoP 或 mTLS、BFF | RP、API |
| 账户接管 | 按未经验证的邮箱关联账户 | 用 `iss` + `sub` 标识用户 | RP |

## 六、上线前检查清单

- [ ] 只使用授权码流程（`response_type=code`），不用隐式流程和密码模式
- [ ] 每次登录都生成新的 `state`、`nonce`、`code_verifier`，与浏览器会话绑定，用后即删
- [ ] `code_challenge_method` 固定为 `S256`，并确认 OP 支持
- [ ] 回调时先校验 `state`（以及 `iss`），再去换令牌
- [ ] 逐项验证 ID Token：签名、算法白名单、`iss`、`aud`、`exp` 和 `iat`、`nonce`
- [ ] 用 `iss` + `sub` 标识用户，不按邮箱关联账户
- [ ] 在 OP 注册精确的 HTTPS 回调地址，RP 上没有开放重定向
- [ ] 机密客户端优先用 `private_key_jwt` 或 mTLS 认证
- [ ] 令牌只留在后端，单页应用采用 BFF，浏览器只持有 `HttpOnly` 会话 Cookie
- [ ] 登录后更换会话 ID；会话 Cookie 设置 `Secure`、`HttpOnly`、`SameSite`
- [ ] access token 短有效期、最小 scope；refresh token 轮换或绑定发送方
- [ ] 用成熟的 OIDC 库处理协议细节，不手写 JWT 校验

## 结语

回头看整个流程，OIDC 的安全设计有一条清晰的主线：不信任浏览器这条通道。凡是经过它的数据，要么一次性、短时效，要么只是一个哈希，要么会在另一端被核对；真正的令牌只在服务器之间传递，并用签名和受众声明证明自己的来历。抓住这条主线，每个参数为什么存在、少了会出什么问题，也就清楚了。

截至本文写作时，OAuth 2.1 仍是 IETF 草案。它把授权码 + PKCE、回调地址精确匹配、移除隐式流程和密码模式这些做法收进了核心规范，按本文的流程实现，基本已经符合它的方向。本文没有展开的话题还有登出（RP-Initiated Logout、Back-Channel Logout），以及进一步保护授权请求的 PAR（RFC 9126）和 JAR（RFC 9101），感兴趣可以继续阅读。

## 参考资料

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html)
- [OpenID Connect RP-Initiated Logout 1.0](https://openid.net/specs/openid-connect-rpinitiated-1_0.html)
- [OpenID Connect Back-Channel Logout 1.0](https://openid.net/specs/openid-connect-backchannel-1_0.html)
- [RFC 6749：The OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749)
- [RFC 6750：Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750)
- [RFC 7636：Proof Key for Code Exchange（PKCE）](https://www.rfc-editor.org/rfc/rfc7636)
- [RFC 8252：OAuth 2.0 for Native Apps](https://www.rfc-editor.org/rfc/rfc8252)
- [RFC 9068：JWT Profile for OAuth 2.0 Access Tokens](https://www.rfc-editor.org/rfc/rfc9068)
- [RFC 9207：Authorization Server Issuer Identification](https://www.rfc-editor.org/rfc/rfc9207)
- [RFC 9449：DPoP](https://www.rfc-editor.org/rfc/rfc9449)
- [RFC 9700：Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700)
- [RFC 10017：OAuth 2.0 for Browser-Based Applications](https://www.rfc-editor.org/rfc/rfc10017)
- [The OAuth 2.1 Authorization Framework（IETF 草案）](https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/)
