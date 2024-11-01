# JWT身份验证

## 目录
- [什么是JWT?](#什么是JWT?)
- [基本结构](#基本结构)
- [JWT的工作原理](#JWT的工作原理)
- [使用JWT](#使用JWT)
- [JWT的优点和缺点](#JWT的优点和缺点)


## 什么是JWT?
JWT(**JSON Web Token**)是一种基于JSON格式的令牌，用于在不同系统之间安全地传递信息，常用于身份验证和授权场景。尤其是在无状态的HTTP请求中，JWT是非常常见的用于实现WebAPI身份认证的机制。

## 基本结构
JWT由三部分组成，使用点（ `.` ）分割：

**1. Header（头部）**
* 通常包含令牌的类型（JWT）和所使用的签名算法（如`HMAC SHA256` 或 `RSA`）。
* 示例：
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
**2. Payload（有效负载）**

   - 有效负载部分包含声明（Claims），这些声明是关于用户及其他数据的信息。声明可以分为三类：
     - **注册声明**: 预定义的一些标准声明，如 `iss`（发行者）、`exp`（过期时间）、`sub`（主题）等。
     - **公共声明**: 可以自定义，使用 URI 作为声明名称。
     - **私有声明**: 自定义的声明，供特定应用程序使用。
   - 示例：
```json
{
    "sub": "1234567890",
    "name": "John Doe",
    "admin": true,
    "exp": 1672444870
}
```
**3. Signature（签名）**

  - 签名部分是通过编码的头部和有效负载以及密钥生成的，以确保令牌的完整性和真实性。可以使用 `HMAC SHA256` 或 `RSA` 等算法来生成签名。
   - 示例：
```scss
HMACSHA256(
       base64UrlEncode(header) + "." +
       base64UrlEncode(payload),
       secret)
```
## JWT的工作原理

1. **用户登录**：用户通过输入凭据（如用户名和密码）进行身份验证。
2. **生成JWT**：服务器验证凭据后，生成JWT，并将其返回给客户端。
3. **存储JWT**：客户端可以将JWT存储在本地存储或`Cookie`中。
4. **发送JWT**：客户端在后续请求中，将JWT添加到`HTTP`头中（通常是`Authorization`头，格式为`Bearer<token>`）。
5. **验证JWT**：服务器收到请求后，验证JWT的签名和有效性。如果有效，则允许访问请求的资源，否则，拒绝访问。 

## 使用JWT

1. 首先，我们需要在`appsettings.json`中配置一些必要的内容：
```json
"JWT": {
  "KEY":"ASDZXASDHAUISDHASDOHAHSDUAHDSASDZXASDHAUISDHASDOHAHSDUAHDS",
  "Issuer": "news.com",
  "Audience": "lynn"
}
```
2. 在`LoginController.cs`中创建一个新的API接口`JWTLogin`，用于生成`JWT Token`。
```C#
private readonly WebContext _webContext;
private readonly IConfiguration _configuration;

public LoginController(WebContext webContext, IConfiguration configuration)
{
    _webContext = webContext;
    _configuration = configuration;
}
```
```C#
[HttpPost("jwt")]
public string JWTLogin(LoginPostDto value)
{
    var user = (from a in _webContext.Employee
                where a.Account == value.Account
                && a.Password == value.Password
                select a).SingleOrDefault();
    if (user == null)
    {
        return "账号密码错误";
    }
    else
    {
        var claims = new List<Claim>
        {
            new Claim (JwtRegisteredClaimNames.Email,user.Account),
            new Claim("FullName",user.Name),
            new Claim(JwtRegisteredClaimNames.NameId,user.EmployeeId.ToString()),
            new Claim(ClaimTypes.Role,"administrator")
        };
        //从配置文件中获取 JWT 密钥（JWT:KEY），并使用该密钥创建一个对称安全密钥
        var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["JWT:KEY"]));
        // 创建 JWT
        var jwt = new JwtSecurityToken
            (
                issuer: _configuration["JWT:Issuer"],   //设置发行者，通常是生成该 JWT 的服务器。
                audience: _configuration["JWT:Audience"],   //设置受众，通常是接收此 JWT 的应用或服务。
                claims:claims,  //使用之前创建的 Claims。
                expires:DateTime.Now.AddMinutes(30),    //设置 JWT 的过期时间，此处设置为 30 分钟后。
                signingCredentials:new SigningCredentials(securityKey,SecurityAlgorithms.HmacSha256)    //设置签名凭据，使用安全密钥和 HMAC SHA256 算法进行签名。

            );
        var token = new JwtSecurityTokenHandler().WriteToken(jwt);  //使用 JwtSecurityTokenHandler 的 WriteToken 方法生成 JWT 字符串
        return token;
    }
}
```
**测试API：**
![d7e34c087b52aa8637e2be01cea31605.png](en-resource://database/802:1)
**验证token：**
![7f763548c6ed204ec8c0facf8198d30c.png](en-resource://database/804:1)
如图所示，PAYLOAD:DATA 部分显示我们设置的所有数据都正确地包含在生成的 JWT 中，这表明该 JWT 是有效的。

需要注意的是，这里只能放置一些非敏感信息，切勿包含机密数据，因为这些内容可以被反解码。

3. 添加必要的Nuget包`Microsoft.AspNetCore.Authentication.JwtBearer`
![6e5b5930f719619355e225725c5f417d.png](en-resource://database/808:1)

4. 在`Program.cs`中 配置JWT身份验证：
```C#
//JWT验证
//AddAuthentication: 此方法将身份验证服务添加到依赖注入容器中，并指定默认的身份验证方案。
//JwtBearerDefaults.AuthenticationScheme: 这是 JWT Bearer 身份验证的默认方案，表示该应用程序将使用 JWT 进行身份验证。
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>            //AddJwtBearer: 这是用于配置 JWT Bearer 身份验证的扩展方法。通过传入一个 options 参数，可以设置相关的参数来验证 JWT。
    {
        options.TokenValidationParameters = new TokenValidationParameters       //TokenValidationParameters: 该属性定义了一组参数，用于验证传入的 JWT。它包含多个重要的配置项。
        {
            ValidateIssuer = true,                                      //是否验证 JWT 的发行者。设置为 true 表示需要验证。
            ValidIssuer = builder.Configuration["JWT:Issuer"],          //指定有效的发行者，通常是生成 JWT 的服务器的标识。此值来自配置文件中的 JWT:Issuer。
            ValidateAudience = true,                                    //指示是否验证 JWT 的受众。设置为 true 表示需要验证。
            ValidAudience = builder.Configuration["JWT:Audience"],      //指定有效的受众，通常是接收 JWT 的应用程序或服务的标识。此值来自配置文件中的 JWT:Audience。
            ValidateLifetime = true,                                    //指示是否验证 JWT 的有效期。设置为 true 表示需要检查 JWT 是否过期。
            ClockSkew = TimeSpan.Zero,                                  //ClockSkew 表示在验证 JWT 的有效期（如 exp 声明）时允许的时间偏差。它允许验证过程在签发和接收 JWT 之间可能存在的时钟不同步问题。 当将 ClockSkew 设置为 TimeSpan.Zero 时，表示不允许任何时间偏差。
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["JWT:KEY"]))   //指定用于签名和验证 JWT 的密钥。这里使用 SymmetricSecurityKey，它是基于对称密钥的，确保只有使用同一密钥的服务器可以生成和验证 JWT。
        };
    });
```

5. 验证是否能够成功访问受权限控制的接口：

* 未添加头部信息：

![0cea0533a4bf511e729768cff6f96fb3.png](en-resource://database/812:1)

* 添加头部信息：

![cfe60c08bfc898c7d706826f78c132c4.png](en-resource://database/810:1)
如图所示，在 Headers 中设置 Authorization 值，格式为 "Bearer " 后接一个空格，再加上获得的 JWT，即可成功访问 API。

## JWT的优点和缺点

#### 优点

- **无状态**: JWT 是无状态的，服务器不需要存储会话信息，减少了服务器的负担。
- **跨域支持**: 可以方便地在不同域之间传递 JWT，适合微服务架构。
- **自包含**: JWT 包含所有用户信息，不需要查询数据库来获取用户信息。

#### 缺点

- **安全性**: 如果密钥泄露，JWT 将不再安全，需要妥善管理密钥。
- **不可撤销**: 一旦发放，JWT 不能被撤销，除非使用黑名单或设置过期时间。
- **大小问题**: JWT 的大小可能会比简单的会话 ID 大，这在某些情况下可能会影响性能。

## JWT 登录后获取用户信息
**获取方式同”第14章-获取登录用户信息与内置或自定义实现的讨论-内置身份验证“**，这里不再赘述。