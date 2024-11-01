# 文档上传API

## 目录
- [基本文档上传](#基本文档上传)
- [文档上传与简单传值](#文档上传与简单传值)
- [使用ModelBinder](#使用ModelBinder)


## 基本文档上传

在之前第6章新增子数据时，我们进行了简单的 `NewsFiles` 表数据新增操作，但未实际处理文档上传。现在，改造该接口以接收上传的文档，并将文档存储到项目的文件夹中。

* 通过`IFormFile`可以读取客户端上传的文件，并将其处理为服务器端可以使用的格式。
```C#
[HttpPost]
public void Post(IFormFile file)
{
}
```
* 判断文档存取位置：
1. 存放在当前项目文件夹下
```C#
 [HttpPost]
 public void Post(IFormFile file)
 {
     if (file.Length > 0)
     {
         var fileName = file.FileName;
         using (var stream = System.IO.File.Create(fileName))
         { 
             file.CopyTo(stream); 
         }
     }
 }
```

**文档上传测试：**

![3fa9969eac85f73a95349fdea3378fed.png](en-resource://database/746:1)
![167c27e030c61c661b5351a07ea6a789.png](en-resource://database/748:1)

2. 在项目文件夹下新建`wwwroot`文件夹存放上传的文档：
![702b62a292f839670c8e5c59bfcdd45e.png](en-resource://database/750:1)

**完整代码：**
```C#
using Microsoft.AspNetCore.Mvc;
using WebAPITest.Dtos;
using WebAPITest.Models;

namespace WebAPITest.Controllers
{
    [Route("api/news/{NewsId}/UploadFile")]
    [ApiController]
    public class NewsFilesController : ControllerBase
    {
        private readonly WebContext _webContext;
        //IWebHostEnvironment 用于提供有关 web 主机环境的信息，包括应用程序的根路径、内容根路径以及当前环境（例如开发、测试或生产）等信息
        private readonly IWebHostEnvironment _webHostEnvironment;

        public NewsFilesController(WebContext webContext, IWebHostEnvironment webHostEnvironment) 
        { 
            _webContext = webContext;
            _webHostEnvironment = webHostEnvironment;
        }
        
        [HttpPost]
        public void Post(IFormFile file)
        {
            if (file.Length > 0)
            {
                string rootRoot = _webHostEnvironment.ContentRootPath + @"\wwwroot\";
                var fileName = file.FileName;
                using (var stream = System.IO.File.Create(rootRoot + fileName))
                { 
                    file.CopyTo(stream); 
                }
            }
        }
    }
}
```
**文档上传测试：**
![580171d0b7298c1ad3caa74f7f9aad46.png](en-resource://database/752:1)
![bec2d1090d2a447fa5feeb8652bdd460.png](en-resource://database/754:1)

3. 同时上传多个文件：
```C#
[HttpPost]
public void Post(List<IFormFile> files)
{
    string rootRoot = _webHostEnvironment.ContentRootPath + @"\wwwroot\";
    foreach (var file in files)
    {
        if (file.Length > 0)
        {
            var fileName = file.FileName;
            using (var stream = System.IO.File.Create(rootRoot + fileName))
            {
                file.CopyTo(stream);
            }
        }
    }
}
```
>`IFormFileCollection`、`IEnumerable<IFormFile>` 和 `List<IFormFile>` 都可以接收多个文档。

文档上传测试：
![d9333230ec3264eb8cfbac9b68fe34c5.png](en-resource://database/756:1)

4. 如果需要在浏览器中查看文档，则需要在Program.cs中新增配置：
```C#
app.UseStaticFiles();
```
![530b2820592a070fd50b9adc3e990143.png](en-resource://database/758:1)

## 文档上传与简单传值
简单获取传入的主键 ID，并将文件存储到对应 ID 的文件夹下，并将上传的文档数据存入NewsFiles表中。

1. 完整代码如下：
```C#
using Microsoft.AspNetCore.Mvc;
using WebAPITest.Dtos;
using WebAPITest.Models;

namespace WebAPITest.Controllers
{
    [Route("api/newsFiles")]
    [ApiController]
    public class NewsFilesController : ControllerBase
    {
        private readonly WebContext _webContext;

        private readonly IWebHostEnvironment _webHostEnvironment;

        public NewsFilesController(WebContext webContext, IWebHostEnvironment webHostEnvironment) 
        { 
            _webContext = webContext;
            _webHostEnvironment = webHostEnvironment;
        }

        [HttpPost]
        public ActionResult Post(List<IFormFile> files,[FromForm]Guid Newsid)
        {
            if (!_webContext.News.Any(a => a.NewsId == Newsid))
            {
                return NotFound("找不到数据");
            }
            string rootRoot = _webHostEnvironment.ContentRootPath + @"\wwwroot\UploadFiles\"+Newsid+"\\";
            //判断文件夹是否存在，不存在则创建
            if (!Directory.Exists(rootRoot))
            {
                Directory.CreateDirectory(rootRoot);
            }
            foreach (var file in files)
            {
                if (file.Length > 0)
                {
                    
                    var fileName = file.FileName;
                    using (var stream = System.IO.File.Create(rootRoot + fileName))
                    {
                        file.CopyTo(stream);
                        NewsFiles insert = new NewsFiles()
                        {
                            NewsId = Newsid,
                            Name = fileName,
                            Path = "/UploadFiles/" + Newsid + "/" + fileName,
                            Extension = Path.GetExtension(fileName)
                        };
                        _webContext.NewsFiles.Add(insert);
                    }
                }
            }
            _webContext.SaveChanges();
            return NoContent();
        }
    }
}
```
2. 文档上传测试：
![d720b8b4824c6b8dc8c93dc6f662d516.png](en-resource://database/760:1)
![183442a565bc3fc4c073d33df93e1ce3.png](en-resource://database/762:1)

## 使用ModelBinder
——使用 `ModelBinder` 处理 `FormData` 的 `JSON` 字符串并反序列化为对应类对象

>假设需要同时上传新闻信息和相关文档的存档。

1. 在NewsController.cs中新增测试代码：

```C#
[HttpPost("UploadFiles")]
public ActionResult POSTALL([FromForm]NewsPostDto value, IFormFileCollection files)
{
    return Ok(value);
}
```

2. 接口请求测试返回了验证错误信息。尽管添加了` [FromForm] `标签，程序仍无法自动将接收到的 `JSON` 字符串转换为相应的类。

![00634db4f270acfa9a4099ebdaa703e8.png](en-resource://database/764:1)

3. 所以可以通过反序列化 `JSON` 字符串，将其转换为相应的类。

* 示例代码：

```C#
[HttpPost("UploadFiles")]
public ActionResult POSTALL([FromForm]string value, IFormFileCollection files)
{
    NewsPostDto news = JsonSerializer.Deserialize<NewsPostDto>(value);
    return Ok(news);
}
```

* 测试：

![18b2220a8d74dd8f8c7f55868f826b72.png](en-resource://database/766:1)

* 这里**需要注意**的是，需要确保 `NewsPostDto` 的属性命名与 `JSON` 字段相符，包括大小写。

4. 进一步优化下，可以使用`ModelBinder`将 HTTP 请求的内容（如查询字符串、表单数据和路由数据）绑定到控制器操作方法的参数或模型上。
* 创建`Binders`文件夹，并新建`FormDataJsonBinder.cs`并继承`IModelBinder`。
![c6cd229e6aaac53e813235e11642db90.png](en-resource://database/768:1)
```C#
using Microsoft.AspNetCore.Mvc.ModelBinding;

namespace WebAPITest.Binders
{
    public class FormDataJsonBinder:IModelBinder
    {
    }
}
```
*  完整代码如下，具体内容可参考官方文档，这里不做赘述。
```C#
using Microsoft.AspNetCore.Mvc.ModelBinding;
using System.Text.Json;
using WebAPITest.Dtos;
using WebAPITest.Models;

namespace WebAPITest.Binders
{
    public class FormDataJsonBinder:IModelBinder
    {
        public Task BindModelAsync(ModelBindingContext modelBindingContext)
        {
            //检查 modelBindingContext 是否为 null，如果是，则抛出 ArgumentNullException
            if (modelBindingContext == null)
            { 
                throw new ArgumentNullException(nameof(modelBindingContext));
            }
            //从 modelBindingContext 中获取模型名称，并使用该名称从 ValueProvider 中检索相应的值。
            var modelName = modelBindingContext.ModelName;
            var valueProviderResult = modelBindingContext.ValueProvider.GetValue(modelName);
            //如果未找到值，则直接返回，表示未能绑定模型。
            if (valueProviderResult == ValueProviderResult.None)
            {
                return Task.CompletedTask;
            }
            //将模型值存入 ModelState，以便后续的验证和处理
            modelBindingContext.ModelState.SetModelValue(modelName, valueProviderResult);
            //获取 valueProviderResult 中的第一个值。
            var value = valueProviderResult.FirstValue;
            //如果获取的值为空或 null，则返回，表示未能绑定模型。
            if (string.IsNullOrEmpty(value))
            { 
                return Task.CompletedTask;
            }
            try
            {
                //关键代码，使用 JsonSerializer.Deserialize 方法将 JSON 字符串反序列化为指定类型（modelBindingContext.ModelType）
                object result = JsonSerializer.Deserialize(value,modelBindingContext.ModelType);
                modelBindingContext.Result = ModelBindingResult.Success(result);
            }
            catch (Exception ex)
            {
                modelBindingContext.Result = ModelBindingResult.Failed();
            }
            return Task.CompletedTask;
        }
    }
}
```
* 在 `NewsController.cs` 的 `POST` 方法中使用 `FormDataJsonBinder`。
```
[HttpPost("UploadFiles")]
public ActionResult POSTALL([FromForm][ModelBinder(BinderType = typeof(FormDataJsonBinder))] NewsPostDto value, IFormFileCollection files)
{
    return Ok(value);
}
```

* 测试：
![11d4d4ef71a0a3e1644b6296aa03177f.png](en-resource://database/770:1)

* 完整代码：
```C#
[HttpPost("UploadFiles")]
public ActionResult POSTALL([FromForm][ModelBinder(BinderType = typeof(FormDataJsonBinder))] NewsPostDto value, IFormFileCollection files)
{
    News insert = new News();
    _webContext.News.Add(insert).CurrentValues.SetValues(value);
    _webContext.SaveChanges();
    string rootRoot = _webHostEnvironment.ContentRootPath + @"\wwwroot\UploadFiles\" + insert.NewsId + "\\";
    if (!Directory.Exists(rootRoot))
    {
        Directory.CreateDirectory(rootRoot);
    }
    foreach (var file in files)
    {
        if (file.Length > 0)
        {

            var fileName = file.FileName;
            using (var stream = System.IO.File.Create(rootRoot + fileName))
            {
                file.CopyTo(stream);
                NewsFiles insertfile = new NewsFiles()
                {
                    NewsId = insert.NewsId,
                    Name = fileName,
                    Path = "/UploadFiles/" + insert.NewsId + "/" + fileName,
                    Extension = Path.GetExtension(fileName)
                };
                _webContext.NewsFiles.Add(insertfile);
            }
        }
    }
    _webContext.SaveChanges();
    return NoContent();
}
```
* 测试：
![7ddbc5af4c1c3d045dde04e758653e70.png](en-resource://database/772:1)
![c46840ff164edeedf2c30072515f85a9.png](en-resource://database/774:1)


#### 小结
通过使用 `ModelBinder`，可以灵活地处理请求数据，适应不同的业务需求。无论是使用内置的 `ModelBinder` 还是创建自定义的，`ModelBinder` 都能帮助你有效地将 `HTTP` 请求转换为可用的对象模型。