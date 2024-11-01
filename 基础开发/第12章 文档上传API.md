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

![1730441455425](https://github.com/user-attachments/assets/8cc7181b-1345-4f75-8021-74694fbbc5b2)
![1730441461520](https://github.com/user-attachments/assets/ffdabe7d-9c0e-4e00-aad3-53ebad50873d)

2. 在项目文件夹下新建`wwwroot`文件夹存放上传的文档：
![1730441467409](https://github.com/user-attachments/assets/edffcddf-dee0-4567-8d85-7311e82183b0)

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
![1730441479082](https://github.com/user-attachments/assets/82dd5371-514e-47db-8d7f-d673a58c959b)
![1730441484330](https://github.com/user-attachments/assets/e8576f68-cbbb-4b09-9f9b-465730adceaf)

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

* 文档上传测试：
![1730441502638](https://github.com/user-attachments/assets/69af8305-aab6-41e2-95f8-5f05d198228e)

4. 如果需要在浏览器中查看文档，则需要在Program.cs中新增配置：
```C#
app.UseStaticFiles();
```
![1730441517838](https://github.com/user-attachments/assets/de2656cb-c562-49de-852c-a2b0286f6f6f)

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
![1730441532635](https://github.com/user-attachments/assets/f069f00a-962d-4806-ab3d-6d5cca4fe7a0)
![1730441538434](https://github.com/user-attachments/assets/ebe8d2a3-2c8c-4813-a677-d3ab2029a65c)

## 使用ModelBinder
——使用 `ModelBinder` 处理 `FormData` 的 `JSON` 字符串并反序列化为对应类对象

假设需要同时上传新闻信息和相关文档的存档。

1. 在NewsController.cs中新增测试代码：

```C#
[HttpPost("UploadFiles")]
public ActionResult POSTALL([FromForm]NewsPostDto value, IFormFileCollection files)
{
    return Ok(value);
}
```

2. 接口请求测试返回了验证错误信息。尽管添加了` [FromForm] `标签，程序仍无法自动将接收到的 `JSON` 字符串转换为相应的类。

![1730441552485](https://github.com/user-attachments/assets/54aaa0fc-b6c9-418b-bba4-783e7d06e34a)

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

![1730441558563](https://github.com/user-attachments/assets/3edeb849-c658-4b00-af4a-b706f49f7095)

* 这里**需要注意**的是，需要确保 `NewsPostDto` 的属性命名与 `JSON` 字段相符，包括大小写。

4. 进一步优化下，可以使用`ModelBinder`将 HTTP 请求的内容（如查询字符串、表单数据和路由数据）绑定到控制器操作方法的参数或模型上。
* 创建`Binders`文件夹，并新建`FormDataJsonBinder.cs`并继承`IModelBinder`。
![1730441569106](https://github.com/user-attachments/assets/6d7f0e9e-fba6-4be2-b418-5bd0b72a26e4)

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
5. 在 `NewsController.cs` 的 `POST` 方法中使用 `FormDataJsonBinder`。
```
[HttpPost("UploadFiles")]
public ActionResult POSTALL([FromForm][ModelBinder(BinderType = typeof(FormDataJsonBinder))] NewsPostDto value, IFormFileCollection files)
{
    return Ok(value);
}
```

* 测试：
![1730441587319](https://github.com/user-attachments/assets/c7be90ac-f28e-4459-9155-33e27c520522)

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
![1730441635564](https://github.com/user-attachments/assets/0b6b1de7-4b64-4ea9-92a5-af444cbe26e4)
![1730441640770](https://github.com/user-attachments/assets/506af375-112f-47b2-b562-43d9f646f5df)

#### 小结
通过使用 `ModelBinder`，可以灵活地处理请求数据，适应不同的业务需求。无论是使用内置的 `ModelBinder` 还是创建自定义的，`ModelBinder` 都能帮助你有效地将 `HTTP` 请求转换为可用的对象模型。
