# Filter

## 目录
- [什么是Filter？](#什么是Filter？)
- [过滤器的类型](#过滤器的类型)
- [过滤器执行顺序和工作流程](#过滤器执行顺序和工作流程))
- [AuthorizationFilter的自定义权限验证](#AuthorizationFilter的自定义权限验证)
- [ResourceFilter之文件大小和扩展名检查](#ResourceFilter之文件大小和扩展名检查)
- [ActionFilter之日志记录](#ActionFilter之日志记录)
- [ResultFilter之统一返回格式记录](#ResultFilter之统一返回格式记录)


## 什么是Filter？
Filter（过滤器）是一种用于执行控制器操作或处理请求的过程中插入自定义逻辑的机制。它们可以用于多种目的，包括身份验证、授权、缓存、异常处理和记录等。过滤器提供了一种集中处理请求和响应的方式，从而简化了代码的维护和重用。

## 过滤器的类型

#### 1.授权过滤器（Authorization Filters）

* 在执行控制器操作之前检查用户的身份和权限。
* 例如，可以用来确保用户已登录并具备访问特定资源的权限。

#### 2.资源过滤器（Resource Filters）

* 在控制器执行前后执行，用于缓存或资源初始化。
* 例如，可以在执行操作前获取某些数据并将其存储在HttpContext.Items中。

#### 3.操作过滤器（Action Filters）

* 在控制器操作执行前和执行后进行处理。
* 可以用于记录日志、修改模型、执行验证等。
* 例如，记录请求的开始和结束时间。

#### 4.异常过滤器（Exception Filters）

* 在控制器操作中发生未处理异常时执行。
* 可以用于处理特定类型的异常并生成自定义的错误响应。
*
#### 5.结果过滤器（Result Filters）

* 在操作结果生成之后但在结果写入响应之前执行。
* 可用于修改结果，如在响应中添加自定义头部。

## 过滤器执行顺序和工作流程
![1730442277101](https://github.com/user-attachments/assets/d30ada1c-6685-4732-9ad0-4189b0431fc1)

### 1. 请求处理管道

当 HTTP 请求到达 ASP.NET Core 应用时，它会经过一系列的中间件和过滤器。图中显示的流程是请求处理的顺序，从中间件到各个过滤器。

### 2. 中间件

- **Middleware**: 在请求进入控制器之前，首先经过中间件。中间件可以执行各种操作，例如身份验证、日志记录、异常处理等。中间件在请求管道中按顺序执行。

### 3. 过滤器的类型和顺序

#### - **Authorization Filter（授权过滤器）**
   - 在请求进入控制器之前，首先执行授权过滤器。这些过滤器用于验证用户的身份和权限，确保只有具备访问权限的用户才能继续。

#### - **Resource Filter（资源过滤器）**
   - 紧接着执行资源过滤器。这些过滤器可以处理资源的初始化或缓存逻辑。

#### - **Exception Filter（异常过滤器）**
   - 在资源过滤器之后，异常过滤器负责处理在执行控制器动作时可能抛出的异常。如果出现异常，这些过滤器将提供处理逻辑并返回相应的错误响应。

### 4. 模型绑定

- **Model Binding（模型绑定）**:
   - 模型绑定将请求数据（如查询参数、表单数据等）绑定到控制器方法的参数。这是请求处理中的一个重要步骤，确保数据能够正确传递到控制器。

### 5. 操作过滤器

- **Action Filter (Before)**:
   - 在执行控制器的动作方法之前，执行操作过滤器。这些过滤器可以用于日志记录、输入验证、修改输入参数等。

- **Action Execution（动作执行）**:
   - 实际执行控制器的动作方法。此步骤处理业务逻辑，并返回结果。

- **Action Filter (After)**:
   - 在动作执行后，再次执行操作过滤器。这些过滤器可以用于结果处理、清理资源等。

### 6. 结果过滤器

- **Result Filter**:
   - 在控制器动作返回结果后，执行结果过滤器。这些过滤器允许在结果生成之前修改结果，如添加自定义头部或进行日志记录。

### 7. 结果执行

- **Result Execution**:
   - 最后，将处理结果转换为 HTTP 响应并发送回客户端。

## AuthorizationFilter的自定义权限验证

1. 创建一个名为 `Filters` 的文件夹，在其中新建 `NewsAuthorizationFilter.cs` 文件并继承`IAuthorizationFilter`。
![1730442296053](https://github.com/user-attachments/assets/7c575a09-3b79-4262-9672-884f6ea179cc)

```C#
using Microsoft.AspNetCore.Mvc.Filters;

namespace WebAPITest.Filters
{
    public class NewsAuthorizationFilter : IAuthorizationFilter
    {
        public void OnAuthorization(AuthorizationFilterContext context)
        {
            
        }
    }
}
```
2. 在`OnAuthorization`方法中可以实现自定义的 Cookie 验证、Token 验证等功能。以下是一个简单的 Token 验证示例。
* 在 `Dtos` 文件夹下新建 `ReturnJson.cs` 文件，用于设置统一的返回格式。
```C#
namespace WebAPITest.Dtos
{
    public class ReturnJson
    {
        public dynamic Data { get; set; }
        public int HttpCode {  get; set; }
        public string ErrorMessage { get; set; }
    }
}
```
* 简单Token自定义验证示例：
```C#
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.Extensions.Primitives;
using WebAPITest.Dtos;

namespace WebAPITest.Filters
{
    public class NewsAuthorizationFilter : IAuthorizationFilter
    {
        public void OnAuthorization(AuthorizationFilterContext context)
        {
            bool tokenFlag = context.HttpContext.Request.Headers.TryGetValue("Authorization", out StringValues outValue);
            if (tokenFlag)
            {
                if (outValue != "123")
                {
                    context.Result = new JsonResult(new ReturnJson()
                    {
                        Data = "test1",
                        HttpCode = 401,
                        ErrorMessage = "没有登录"
                    });
                }
            }
            else
            {
                context.Result = new JsonResult(new ReturnJson()
                {
                    Data = "test1",
                    HttpCode = 401,
                    ErrorMessage = "没有登录"
                });
            }
        }
    }
}
```

3. 可以在 `Program.cs` 中全局注册过滤器，或者在控制器或操作方法上应用过滤器。
* **局部注册**：
 注释掉之前配置的内置验证功能：
![1730442315061](https://github.com/user-attachments/assets/6eae2243-ae95-48f9-9565-e17e2bd680fc)

引入自定义的验证功能：
```C#
[TypeFilter(typeof(NewsAuthorizationFilter))]
[HttpGet]
public ActionResult<IEnumerable<NewsDto>> Get([FromQuery] NewsParameter value){}
```
测试传入`Authorization`：
![1730442323714](https://github.com/user-attachments/assets/610ebded-34c3-4dc4-b415-c6380b305be7)

测试不传入`Authorization`：
![1730442333042](https://github.com/user-attachments/assets/3f037469-5d94-46f9-a8cc-7ef9776e457a)

测试传入`Authorization`，且值为`123`：
![1730442343290](https://github.com/user-attachments/assets/ba2f4174-5f6d-41f3-a24e-e74baa2d35e7)

让`NewsAuthorizationFilter`同时继承`Attribute`和`AuthorizationFilter`，这样在使用时可以直接作为标签来应用。
```C#
public class NewsAuthorizationFilter : Attribute, IAuthorizationFilter
{
}
```
```C#
[NewsAuthorizationFilter]
[HttpGet]
public ActionResult<IEnumerable<NewsDto>> Get([FromQuery] NewsParameter value)
{
}
```

* **全局注册**：
```C#
builder.Services.AddMvc(option => {
    option.Filters.Add(new NewsAuthorizationFilter()); 
});
```
需要注意的是，由于使用了自定义的 Token 验证，`LoginController.cs` 中的内置标签 `[AllowAnonymous]` 将不再生效。因此，在自定义的 Token 验证中，需要添加对 `[AllowAnonymous]` 标签的判断。
```C#
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.Extensions.Primitives;
using WebAPITest.Dtos;

namespace WebAPITest.Filters
{
    public class NewsAuthorizationFilter : Attribute, IAuthorizationFilter
    {
        public void OnAuthorization(AuthorizationFilterContext context)
        {
            bool tokenFlag = context.HttpContext.Request.Headers.TryGetValue("Authorization", out StringValues outValue);
            //添加对[AllowAnonymous]标签的判断
            var ignore = (from a in context.ActionDescriptor.EndpointMetadata
                          where a.GetType() == typeof(AllowAnonymousAttribute)
                          select a).FirstOrDefault();
            if (ignore == null)
            {
                if (tokenFlag)
                {
                    if (outValue != "123")
                    {
                        context.Result = new JsonResult(new ReturnJson()
                        {
                            Data = "test1",
                            HttpCode = 401,
                            ErrorMessage = "没有登录"
                        });
                    }
                }
                else
                {
                    context.Result = new JsonResult(new ReturnJson()
                    {
                        Data = "test2",
                        HttpCode = 401,
                        ErrorMessage = "没有登录"
                    });
                }
            }
        }
    }
}
```

4. 假设现在需要在过滤器中读取数据库并传入Roles参数。
```C#
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.Extensions.Primitives;
using WebAPITest.Dtos;
using WebAPITest.Models;

namespace WebAPITest.Filters
{
    public class NewsAuthorizationFilter : Attribute, IAuthorizationFilter
    {
        public string Roles = "";
        public void OnAuthorization(AuthorizationFilterContext context)
        {
            WebContext webContext = (WebContext)context.HttpContext.RequestServices.GetService(typeof(WebContext));
            //验证逻辑代码
	    }
     }
}
```
```C#
[NewsAuthorizationFilter(Roles = "aaaa")]
[HttpGet]
public ActionResult<IEnumerable<NewsDto>> Get([FromQuery] NewsParameter value)
{
}
```

## ResourceFilter之文件大小和扩展名检查

1. 在`Filters`文件夹下新建 FileLimitAttribute.cs 文件并继承`Attribute`和`IResultFilter`。
```C#
using Microsoft.AspNetCore.Mvc.Filters;

namespace WebAPITest.Filters
{
    public class FileLimitAttribute : Attribute, IResultFilter
    {
        public void OnResultExecuted(ResultExecutedContext context)
        {
            //在结果执行之后触发,可以在这个方法中执行一些逻辑，比如清理资源、记录执行时间、处理异常等。
        }

        public void OnResultExecuting(ResultExecutingContext context)
        {
            //在结果执行之前触发,可以在这个方法中添加自定义逻辑，以便在控制器动作结果被执行之前做一些处理，比如修改结果或记录日志等。
        }
    }
}

```
2. 添加简单的验证逻辑代码：
```C#
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using WebAPITest.Dtos;

namespace WebAPITest.Filters
{
    public class FileLimitAttribute : Attribute, IResultFilter
    {
        public int Size = 100000;
        public void OnResultExecuted(ResultExecutedContext context)
        {
            
        }

        public void OnResultExecuting(ResultExecutingContext context)
        {
            var files = context.HttpContext.Request.Form.Files;
            foreach (var file in files)
            {
                if (file.Length > (1024 * 1024 * Size))
                { 
                    context.Result = new JsonResult(new ReturnJson()
                    {
                        Data = "test1",
                        HttpCode = 400,
                        ErrorMessage = "文件过大"
                    });
                }
                if (Path.GetExtension(file.FileName) != ".jpg")
                {
                    context.Result = new JsonResult(new ReturnJson()
                    {
                        Data = "test2",
                        HttpCode = 400,
                        ErrorMessage = "只能上传jpg图片格式"
                    });
                }
            }
        }
    }
}
```
3. 在`NewsFilesController.cs`的`Post`方法中引入标签`[FileLimit]`：
```C#
[FileLimit]
[HttpPost]
public ActionResult Post(List<IFormFile> files,[FromForm]Guid Newsid)
{
}
```
也可以设置上传的文件大小：
```C#
[FileLimit(Size = 1)]
[HttpPost]
public ActionResult Post(List<IFormFile> files,[FromForm]Guid Newsid)
{
}
```
4. 测试文件上传验证功能：

![1730442366639](https://github.com/user-attachments/assets/a7677e0c-445d-468b-a586-eac20172291b)
![1730442378397](https://github.com/user-attachments/assets/5296a18f-4c62-4761-9cdb-506fad03e5e4)

## ActionFilter之日志记录

1. 在`Filters`文件夹下新建 NewsActionFilter.cs 文件并继承`IActionFilter`。
```C#
using Microsoft.AspNetCore.Mvc.Filters;

namespace WebAPITest.Filters
{
    public class NewsActionFilter : IActionFilter
    {
        public void OnActionExecuted(ActionExecutedContext context)
        {
            //在动作方法执行之后触发，可以在这个方法中执行一些后处理逻辑，比如记录执行时间、处理异常、修改响应结果等。
        }

        public void OnActionExecuting(ActionExecutingContext context)
        {
            //在动作方法执行之前触发，可以在这个方法中执行一些预处理逻辑，比如验证输入参数、记录日志或执行安全检查等。
        }
    }
}
```

2. 添加简单的日志记录代码：
```C#
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Filters;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using WebAPITest.Models;

namespace WebAPITest.Filters
{
    public class NewsActionFilter : IActionFilter
    {
        private readonly IWebHostEnvironment _env;
        public NewsActionFilter(IWebHostEnvironment env)
        {
            _env = env;
        }

        public void OnActionExecuted(ActionExecutedContext context)
        {
            string rootRoot = _env.ContentRootPath + @"\Log\";
            //判断文件夹是否存在，不存在则创建
            if (!Directory.Exists(rootRoot))
            {
                Directory.CreateDirectory(rootRoot);
            }

            var employeeName = context.HttpContext.User.FindFirst(ClaimTypes.Email);
            var path = context.HttpContext.Request.Path;
            var method = context.HttpContext.Request.Method;
            string text = "结束时间：" + DateTime.Now.ToString("yyyy/MM/dd HH:mm:ss") + " path:" + path + " method" + method + " user:" + employeeName;
            File.AppendAllText(rootRoot + DateTime.Now.ToString("yyyyMMdd") + ".txt", text);
        }

        public void OnActionExecuting(ActionExecutingContext context)
        {
            string rootRoot = _env.ContentRootPath + @"\Log\";
            //判断文件夹是否存在，不存在则创建
            if (!Directory.Exists(rootRoot))
            {
                Directory.CreateDirectory(rootRoot);
            }

            var employeeName = context.HttpContext.User.FindFirst(ClaimTypes.Email);
            var path = context.HttpContext.Request.Path;
            var method = context.HttpContext.Request.Method;
            string text = "开始时间：" + DateTime.Now.ToString("yyyy/MM/dd HH:mm:ss")+" path:"+path + " method"+method+" user:"+employeeName;
            File.AppendAllText(rootRoot + DateTime.Now.ToString("yyyyMMdd") + ".txt",text);
        }
    }
}
```
3. 验证结果：

![1730442391455](https://github.com/user-attachments/assets/c4cfc539-5e5e-459b-bdb9-db7ab11b048c)
![1730442400145](https://github.com/user-attachments/assets/52a64dfc-4ddb-437e-9bee-fc2c7c8281ba)
![1730442408438](https://github.com/user-attachments/assets/7c5bcff2-dfcd-4a90-a6dd-5dc2a9f9217c)

4. 在ASP.NET Core中使用`ActionFilter`进行日志记录有许多好处，以下是一些主要的**优点**：

* **分离关注点**

通过使用`ActionFilter`进行日志记录，可以将日志逻辑从业务逻辑中分离出来。这种分离有助于保持代码的清晰性和可维护性，允许开发人员专注于核心业务功能而不是日志实现。

* **一致性**

在`ActionFilter`中实现日志记录可以确保在整个应用程序中使用一致的日志格式和策略。无论哪个控制器或动作被调用，日志记录的行为都是相同的，有助于减少重复代码。

 * **中央管理**

集中管理日志记录逻辑使得在需要修改日志记录行为时只需更改一个地方。例如，如果需要改变日志级别或日志格式，只需在`ActionFilter`中修改，而不需要在每个控制器中单独修改。

* **灵活性**

通过`ActionFilter`，可以根据特定条件动态决定是否记录日志。例如，可以根据用户角色、请求类型或其他上下文信息选择性地记录日志。

* **自动记录请求和响应**

`ActionFilter`可以在请求开始前和响应结束后自动记录相关信息。这种能力使得捕获完整的请求-响应周期变得容易，便于后续的调试和分析。

* **简化测试**

将日志记录逻辑放入`ActionFilter`中，使得在测试控制器时可以更容易地模拟和验证日志记录行为。这样，可以独立测试业务逻辑而不干扰日志实现。

 * **增强安全性**

可以在`ActionFilter`中记录请求的信息（如IP地址、用户代理等），以便在发生安全事件时进行审计。这有助于检测和分析潜在的安全威胁。

## ResultFilter之统一返回格式记录
1. 在`Filters`文件夹下新建 NewsResultFilter.cs 文件并继承`IResultFilter`。
```C#
using Microsoft.AspNetCore.Mvc.Filters;

namespace WebAPITest.Filters
{
    public class NewsResultFilter : IResultFilter
    {
        public void OnResultExecuted(ResultExecutedContext context)
        {
            //在结果执行之后触发。你可以在这个方法中进行一些后续处理，比如清理资源、记录执行结果或处理异常。
        }

        public void OnResultExecuting(ResultExecutingContext context)
        {
            //在结果执行之前触发。这个方法可以用于执行一些前置逻辑，比如记录日志、修改响应内容或设置响应头等。
        }
    }
}
```
2. 简单代码示例：
```C#
namespace WebAPITest.Dtos
{
    public class ReturnJson
    {
        public dynamic Data { get; set; }
        public int HttpCode { get; set; }
        public string ErrorMessage { get; set; }
        public dynamic Error { get; set; }
    }
}
```
```C#
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using WebAPITest.Dtos;

namespace WebAPITest.Filters
{
    public class NewsResultFilter : IResultFilter
    {
        public void OnResultExecuted(ResultExecutedContext context)
        {
            
        }

        public void OnResultExecuting(ResultExecutingContext context)
        {
            var contextResult = context.Result as ObjectResult;
            if (context.ModelState.IsValid)
            {
                context.Result = new JsonResult(new ReturnJson()
                {
                    Data = contextResult.Value
                });
            }
            else
            {
                context.Result = new JsonResult(new ReturnJson()
                {
                    Error = contextResult.Value
                });
            }
            
        }
    }
}
```
3. 全局注册：
```C#
builder.Services.AddMvc(option => {
    option.Filters.Add(new AuthorizeFilter());
    option.Filters.Add(typeof(NewsActionFilter));
    option.Filters.Add(new NewsResultFilter());
});
```
4. 测试返回格式：

![1730442434681](https://github.com/user-attachments/assets/cfb2082b-323f-4a03-9d80-bd28eefcc51f)
![1730442444954](https://github.com/user-attachments/assets/b6a01815-35b1-4c6d-9ba4-baae6e5ac17f)

5. 通过`ResultFilter`实现统一的返回格式记录有多种优点，主要包括以下几点：

* **一致性**

通过统一的返回格式，可以确保应用程序中的所有响应都遵循相同的结构。这种一致性使得前端和后端的交互更为顺畅，前端开发人员只需处理一种格式，无需处理不同接口的不同返回结构。

* **可维护性**

当需要更改返回格式时，只需在一个地方（即`ResultFilter`）进行修改，所有使用该过滤器的控制器和动作将自动应用此更改。这减少了代码重复并提高了维护效率。

* **增强的可读性**

统一的返回格式使得API的输出更易于阅读和理解。开发人员和API用户可以迅速识别和解析返回数据，提高开发效率。

* **更好的错误处理**

在统一的返回格式中，可以预定义错误响应的结构，这有助于在出现错误时提供更清晰的信息。通过一致的错误格式，用户能够更容易理解发生了什么错误以及如何解决。

* **便于文档生成**

使用统一的返回格式可以简化API文档的生成过程。许多文档工具（如Swagger/OpenAPI）能够更轻松地生成文档，因为它们可以假设所有响应遵循相同的格式。

* **集中处理日志**

通过在`ResultFilter`中记录返回数据，可以集中处理所有响应的日志记录。这使得监控和分析API使用情况变得更加简便，有助于快速发现潜在的问题。

* **适应性和扩展性**

统一的返回格式为未来的扩展提供了灵活性。例如，可以很容易地添加新的字段（如状态码、时间戳等）而不影响现有的API使用者。

