# 通过Cookie实现身份验证

## 目录
- [实现Cookie认证](#实现Cookie认证)
- [身份验证和登录期限](#身份验证和登录期限)
- [登录期限](#登录期限)
- [获取登录用户信息与内置或自定义实现的讨论](#获取登录用户信息与内置或自定义实现的讨论)


## 实现Cookie认证
实现 Cookie 认证而不使用 ASP.NET Core Identity 可以通过自定义中间件和模型来完成。以下是一个基本的实现步骤：

#### 1.创建用户模型
首先，定义一个用户模型来存储用户的信息，如Employee表及其模型：
```C#
using System;
using System.Collections.Generic;

namespace WebAPITest.Models;

public partial class Employee
{
    public int EmployeeId { get; set; }

    public string Name { get; set; } = null!;

    public string Password { get; set; } = null!;

    public string Account { get; set; } = null!;
}
```

#### 2.配置服务
在 Program.cs 中配置 Cookie 认证服务：
```C#
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme).AddCookie(option =>
{
    option.LoginPath = new PathString("api/Login/NoLogin");
});
```
- **`builder.Services.AddAuthentication`**: 这部分代码注册了身份验证服务。
- **`CookieAuthenticationDefaults.AuthenticationScheme`**: 这是一个常量，表示默认的认证方案，这里指定为 Cookie 认证。这样，后续代码中对 Cookie 认证的设置将应用于默认的认证方案。
- **`.AddCookie(option => { ... })`**: 这是用于配置 Cookie 认证的扩展方法。传入的 `option` 参数是一个配置对象，允许你设置 Cookie 认证的相关选项。
- **`option.LoginPath = new PathString("api/Login/NoLogin");`**: 这行代码指定了当用户未经过身份验证时，应该重定向到的登录路径。也就是说，如果用户访问了需要身份验证的资源，但尚未登录，系统将自动将其重定向到 `api/Login/NoLogin` 这个路径。
```C#
//顺序必须一致
app.UseCookiePolicy();
app.UseAuthentication();
app.UseAuthorization();
```

这段代码的顺序非常重要，通常在 ASP.NET Core 应用程序的 `Configure` 方法中使用。整体流程如下：

1. **`UseCookiePolicy`**: 设置和应用 Cookie 的策略。
2. **`UseAuthentication`**: 处理请求中的身份验证信息，识别用户身份。
3. **`UseAuthorization`**: 基于用户身份，判断其是否有权限访问特定资源。

这样的中间件配置确保了用户在访问受保护资源时，能够按照正确的顺序进行身份验证和授权检查，从而实现安全访问控制。

#### 3.创建登录控制器
创建一个`LoginController.cs`控制器处理登录逻辑：
```C#
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using WebAPITest.Models;

namespace WebAPITest.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class LoginController : ControllerBase
    {
        private readonly WebContext _webContext;

        public LoginController(WebContext webContext)
        { 
            _webContext = webContext;
        }
        [HttpPost]
        public string login(LoginPostDto value)
        {
            var user = (from a in _webContext.Employee
                        where a.Account == value.Account
                        && a.Password == value.Password
                        select a).SingleOrDefault();
            if (user == null)
            {
                return "账号密码错误";
            }
            else {
                //创建一个 `Claim` 对象的列表，`Claim` 是一种表示用户特征或权限的对象。每个 `Claim` 包含一个类型和一个值。
                var claims = new List<Claim>
                {
                    new Claim(ClaimTypes.Name,user.Account),  //这是一个标准的 Claim 类型，表示用户的名称。在这里，`user.Account` 是用户的账户名。
                    new Claim("FullName",user.Name),    //这是一个自定义的 Claim，用于存储用户的全名（`user.Name`）。这里使用了字符串 `"FullName"` 作为 Claim 类型，可以在后续代码中通过这个类型来访问该信息。
                    //一般不这样用，通常是创建角色表获取角色，这里做简单示范
                    new Claim(ClaimTypes.Role,"administrator")
                };
                /*
                ClaimsIdentity: 这是一个表示用户身份的对象，包含了与该用户相关的所有 Claims。
                claims: 之前创建的 Claim 列表，包含用户的基本信息。
                CookieAuthenticationDefaults.AuthenticationScheme: 指定了身份验证的方案，这里使用了默认的 Cookie 认证方案。这个信息将用于后续的身份验证流程。
                */
                var claimsIdentity = new ClaimsIdentity(claims,CookieAuthenticationDefaults.AuthenticationScheme); 
                
                /*
                HttpContext.SignInAsync: 这是一个异步方法，用于将用户的身份信息（Claims）写入 Cookie。它会创建一个身份验证 Cookie，使用户在后续请求中保持登录状态。
                CookieAuthenticationDefaults.AuthenticationScheme: 指定使用的身份验证方案，与之前创建的 ClaimsIdentity 一致。
                new ClaimsPrincipal(claimsIdentity): 创建 ClaimsPrincipal 对象，该对象代表当前用户的主体信息。它封装了 ClaimsIdentity，并提供了访问用户相关 Claims 的方法。
                */
                HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, new ClaimsPrincipal(claimsIdentity));
                return "ok";
            }
        }

        [HttpGet("NoLogin")]
        public string noLogin()
        {
            return "未登入";
        }

        [HttpGet("NoAccess")]
        public string noAccess()
        {
            return "无权限";
        }

        [HttpDelete]
        public void logout()
        {
            HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
        }

        public class LoginPostDto
        {
            public string Password { get; set; }
            public string Account { get; set; }
        }
    }
}
```
整体而言，这段代码实现了以下几个关键功能：

1. **创建用户的身份信息**：通过 Claims 来表示用户的基本信息（如账户名和全名）。
2. **构建 ClaimsIdentity**：将 Claims 组合到一起，形成用户的身份信息。
3. **用户登录**：调用 `SignInAsync` 方法，将用户的身份信息写入 Cookie，使其在后续请求中被识别为已登录状态。

这段代码通常出现在用户登录成功后，确保用户能够持续访问需要身份验证的资源。

#### 4.保护其他控制器
在需要身份验证的控制器或单个`API`接口上应用 `[Authorize]` 特性。
  
```C#
  [Authorize]
  [Route("api/[controller]")]
  [ApiController]
  public class NewsController : ControllerBase
  {
  }
```
```C#
[Authorize]
[HttpGet]
public ActionResult<IEnumerable<NewsDto>> Get([FromQuery] NewsParameter value)
{
}
```

* 这样，当我们访问 `GET /api/news` 时，程序会自动重定向到 `/api/Login/NoLogin`。

![c2e7d84a59f70ab6ffd6062e63ff3682.png](en-resource://database/784:1)

* 在 `POST /api/login` 中输入正确的账户和密码（这里为了方便展示，不做密码加密处理）后，将收到一个“ok”的响应。

![5fae55a2418079951666bd6464773649.png](en-resource://database/786:1)

* 此次再访问 `GET /api/news` 时，即可获取到数据：

![348f88a9400e0da41f25aae1d9bb2652.png](en-resource://database/788:1)

* 执行 `DELETE /api/login` 后，用户将被登出。

![2d563a7c43bcc77b082e329789853f06.png](en-resource://database/790:1)

* 登出后再次访问 `GET /api/news` 时，程序会自动重定向到 `/api/Login/NoLogin`。

![2b1d6edab87bf9653820079781e3cea2.png](en-resource://database/792:1)

#### 总结
以上就是内置 Cookie 验证的简单示范。不过，还有一些小技巧需要分享。大多数 API 都需要登录验证，如果在每个 API 上都添加 `[Authorize]` 特性，会显得繁琐。因此，可以在 `Program.cs` 中设置全局应用，代码如下：
``` C#
builder.Services.AddMvc(option => {
    option.Filters.Add(new AuthorizeFilter()); 
});
```
但这样一来，登录 API 也无法使用，因此需要在登录 API 上方添加 `[AllowAnonymous]` 特性，代码如下：
```C#
[Route("api/[controller]")]
[ApiController]
[AllowAnonymous]
public class LoginController : ControllerBase
{
}
```
这样，该 API 就不会受到验证的限制。

## 身份验证和登录期限
ClaimTypes.Role 在身份验证和授权中的作用非常重要。通过`new Claim(ClaimTypes.Role,"administrator")` 进行角色声明和赋予管理员的身份（当然，通常情况下不是这样赋值的，而是通过角色表的方式获取用户角色，这边只做简单示例）。使用角色声明，可以轻松地管理用户的权限，并实现灵活的访问控制。

#### 用途

- **访问控制**: 通过将角色声明分配给用户，可以在授权时根据角色控制用户对资源的访问。例如，可以允许某些角色访问特定的 API 或页面。
- **角色管理**: 在应用程序中，可以使用角色来组织和管理用户，例如管理员、普通用户、编辑者等。


#### 示例

1. 在用户登录时添加角色：
```C#
new Claim(ClaimTypes.Role,"administrator")
```
2. 在控制器中使用角色进行授权：
```C#
[Authorize(Roles = "administrator")]
```
3. 配置`Program.cs`，访问被拒绝时可以重定向到其他网址：
```C#
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme).AddCookie(option =>
{
    //未登录时会自动导到这个网址
    option.LoginPath = new PathString("/api/Login/NoLogin");
    //没有权限时会自动导到这个网址
    option.AccessDeniedPath = new PathString("/api/Login/NoAccess");
});
```

* 测试接口授权为`[Authorize(Roles = "user")`]时：

![acb996acf9b97488fa98b2fa756ec785.png](en-resource://database/794:1)

* 测试接口授权为`[Authorize(Roles = "administrator")]`时：

![af76f8e11b739715820bb4e2b25aab52.png](en-resource://database/796:1)

4. 从角色表中获取多个角色权限简单步骤示例：
* 获取角色
```C#
var roles = (from a in _webContext.Roles
                where user.Roles.Contains(a.RoleId)
                select a).ToList();
```
* 循环将角色信息添加到 `Claims` 中：
```C#
foreach(var role in roles)
{
    claims.Add(new Claim(ClaimTypes.Role,role.Name));
}
```

#### 登录期限

**1. 全局设置**
在 `Program.cs` 中配置 Cookie 认证选项来指定登录期限，适用于整个应用程序的所有请求。

一旦配置，全局设置将影响所有使用该认证方案的控制器和 API。任何使用 Cookie 认证的请求都将遵循这些全局配置。

**优点**是保持了整个应用的一致性和统一管理。

```C#
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme).AddCookie(option =>
{
    //未登录时会自动导到这个网址
    option.LoginPath = new PathString("/api/Login/NoLogin");
    //没有权限时会自动导到这个网址
    option.AccessDeniedPath = new PathString("/api/Login/NoAccess");
    //设置登录有效期为 30 分钟
    option.ExpireTimeSpan = TimeSpan.FromMinutes(30);
});
```

- **`ExpireTimeSpan`**: 此属性设置 Cookie 的过期时间。在上面的示例中，Cookie 的有效期被设置为 30 分钟。超过此时间后，用户将需要重新登录。
  
- **`SlidingExpiration`**: 这个选项指示是否启用滑动过期。启用后，每次用户请求时，如果 Cookie 仍然有效，过期时间将被重置。例如，如果用户在过期前的 20 分钟内进行活动，Cookie 的过期时间将重新设置为 30 分钟。

**2. 在控制器中设置**
在控制器中设置是指在特定的控制器或操作方法中单独为 Cookie 认证配置选项。

这种设置只适用于指定的控制器或操作方法，不会影响其他控制器或请求。

**优点**是提供灵活性和细粒度控制，允许开发者根据不同的业务需求为特定控制器或操作定义不同的 Cookie 过期策略。

在LoginController.cs控制器中，当用户成功登录时，生成的 Cookie 将根据以下配置进行设置：
```C#
var authenticationProperties = new AuthenticationProperties()
{
    IsPersistent = true, // 如果为 true，Cookie 会在浏览器关闭后仍然有效
    ExpiresUtc = DateTimeOffset.UtcNow.AddMinutes(30) // 可选：设置 Cookie 的过期时间
};
HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, new ClaimsPrincipal(claimsIdentity),authenticationProperties);
return "ok";
```

## 获取登录用户信息与内置或自定义实现的讨论
在应用程序中，获取登录用户信息是身份验证和授权的重要环节。这可以通过使用内置的身份验证系统或自己定制解决方案来实现。以下是对这两种方式的详细介绍：
### 内置身份验证

获取登录用户信息通常涉及以下几个步骤：

1. **身份验证**：用户在系统中登录，系统会验证其凭据（如用户名和密码）。
2. **创建 Claims**：成功登录后，系统会创建包含用户信息的 Claims，例如用户名、角色等。
3. **存储 Claims**：将这些 Claims 存储在 ClaimsPrincipal 对象中，并通过 Cookie 或 Token 将其发送到客户端。
4. **获取信息**：在后续请求中，可以通过 `HttpContext.User` 来访问这些 Claims，从而获取当前登录用户的信息。

**示例：**

* 在控制器中可以直接通过 `HttpContext.User` 来获取信息：

```C#
var username = HttpContext.User.Identity.Name; // 获取用户名
var roles = HttpContext.User.Claims.Where(c => c.Type == ClaimTypes.Role).Select(c => c.Value); // 获取角色
```

* 在Service服务层获取信息：
在`Program.cs`中添加配置信息：
```C#
builder.Services.AddHttpContextAccessor();
```
在服务层：
```C#
private readonly IHttpContextAccessor _httpContextAccessor;
public AsyncService(IHttpContextAccessor httpContextAccessor)
{
    _httpContextAccessor = httpContextAccessor;
}
```

* 第一种获取方式：

```C#
var claim = _httpContextAccessor.HttpContext.User.Claims.ToList();
var Name = claim.Where(a=>a.Type == "FullName").First().Value;
```

* 第二种获取方式：

```C#
var Name = _httpContextAccessor.HttpContext.User.FindFirstValue("FullName");
var Account = _httpContextAccessor.HttpContext.User.FindFirstValue(ClaimTypes.Name);
```
### 自定义身份验证

在某些情况下，内置身份验证可能无法满足需求，这时可以考虑自定义身份验证。

**1. 自定义用户存储**
可以选择使用自定义的数据存储，例如数据库、外部 API 或文件系统，来管理用户信息。

**2. 自定义 Claims**
可以根据业务需求创建特定的 Claims，并将其添加到 ClaimsPrincipal 中。

**3. 自定义身份验证逻辑**
实现自定义的身份验证逻辑，例如通过 RESTful API 验证用户凭据，或实现多因素身份验证等。

### 总结

- **内置身份验证**：适合大多数应用场景，使用简单，易于维护。可以快速集成 Cookie 或 JWT 身份验证，获取用户信息也很方便。
  
- **自定义身份验证**：适合复杂场景或特殊需求，可以灵活地处理用户信息存储和身份验证逻辑，但实现成本较高，开发和维护工作量较大。

选择哪种方案取决于应用的具体需求、复杂性和团队的技术能力。无论使用哪种方式，确保用户信息的安全性和隐私性始终是开发者需要考虑的重要问题。