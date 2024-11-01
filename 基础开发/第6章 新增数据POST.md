# 新增数据POST

## 目录
- [新增数据](#新增数据)
- [新增子数据和同时新增父子数据](#新增子数据和同时新增父子数据)
- [使用DTO新增数据](#使用DTO新增数据)
- [使用AutoMapper新增数据](#使用AutoMapper新增数据)
- [使用内建函数匹配新增数据](#使用内建函数匹配新增数据)


## 新增数据

>简单新增News表的数据：
```C#
[HttpPost]
public void Post([FromBody] News value)
{
    _webContext.News.Add(value);
    _webContext.SaveChanges();
}
```
* 使用 `Add()` 方法将新数据添加到数据集中
* 使用 `SaveChanges() `方法将更改保存到数据库中。
>新增测试，返回200说明执行成功了：
![5bbac768fb411861b9568fef5bc8530e.png](en-resource://database/650:1)
>查询是否能找到这条记录：
![f9bea0ba32ca9ab2e667d9d9bebe84ff.png](en-resource://database/652:1)

>在实际应用中，某些数据（如`UpdateDateTime`、`UpdateEmployeeId`、`InsertDateTime`、`InsertEmployeeId`和`Click`）不适合由用户手动输入，因此通常会进行默认设置。这种默认设置的优化可以通过以下方式实现：
```C#
[HttpPost]
public void Post([FromBody] News value)
{
    News insert = new News()
    {
        Title = value.Title,
        Content= value.Content,
        StartDateTime= value.StartDateTime,
        EndDateTime= value.EndDateTime, 
        Enable = value.Enable,
        UpdateDateTime = DateTime.Now,
        UpdateEmployeeId = 1, 
        InsertDateTime = DateTime.Now,
        InsertEmployeeId = 1,
        Click = 0
    };
    _webContext.News.Add(insert);
    _webContext.SaveChanges();
}
```
>新增测试，分别给`UpdateDateTime`、`UpdateEmployeeId`、`InsertDateTime`、`InsertEmployeeId`和`Click`字段传入值：

![9016d31d6f08c7d0c98a899204f28950.png](en-resource://database/654:1)

>查询新增的这条数据，可以看到`Click`字段的值是系统默认设置的：

![276a6690b264a3771f4037d90bc482fe.png](en-resource://database/656:1)

## 新增子数据和同时新增父子数据

#### 新增子数据

>创建控制器`NewsFilesController.cs`，简单新增`NewsFiles`表的数据：
``` C#
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

        public NewsFilesController(WebContext webContext) 
        { 
            _webContext = webContext;
        }

        [HttpPost]
        public string Post(Guid NewsId,[FromBody] NewsFilesDto value)
        {
            if (!_webContext.News.Any(a => a.NewsId == NewsId))
            {
                return "找不到数据";
            }
            NewsFiles insert = new NewsFiles()
            {
                NewsId = NewsId,
                Name = value.Name,
                Path = value.Path,
                Extension = value.Extension
            };
            _webContext.NewsFiles.Add(insert);
            _webContext.SaveChanges();
            return "ok";
        }
    }
}

```
>新增数据，并查看返回结果：

![b20fec90ec04eb3f70d0b898f62f891c.png](en-resource://database/658:1)
![2b13ae51149e0de845083bad171fa891.png](en-resource://database/660:1)
 
 #### 同时新增父子数据
 * 父子数据表**存在**外键的情况下，代码如下：
 
```C#
[HttpPost]
public void Post([FromBody] News value)
{
    News insert = new News()
    {
        Title = value.Title,
        Content = value.Content,
        StartDateTime = value.StartDateTime,
        EndDateTime = value.EndDateTime,
        Enable = value.Enable,
        UpdateDateTime = DateTime.Now,
        UpdateEmployeeId = 1,
        InsertDateTime = DateTime.Now,
        InsertEmployeeId = 1,
        Click = 0,
        NewsFiles = value.NewsFiles //直接获取子数据
    };
    _webContext.News.Add(insert); //会根据配置好的外键同时新增父子数据
    _webContext.SaveChanges();
}
```
 * 父子数据表**不存在**外键的情况下，代码如下:
 
``` C#
 [HttpPost]
public void Post([FromBody] News value)
{
    News insert = new News()
    {
        Title = value.Title,
        Content = value.Content,
        StartDateTime = value.StartDateTime,
        EndDateTime = value.EndDateTime,
        Enable = value.Enable,
        UpdateDateTime = DateTime.Now,
        UpdateEmployeeId = 1,
        InsertDateTime = DateTime.Now,
        InsertEmployeeId = 1,
        Click = 0
    };
    _webContext.News.Add(insert);
    _webContext.SaveChanges();      //这一步必须做，提交数据库后，父数据才会生成NewsId

    foreach (var temp in value.NewsFiles)
    {
        NewsFiles insert2 = new NewsFiles()
        {
            NewsId = insert.NewsId, //insert在父数据新增后会返回NewsId
            Name = temp.Name,
            Path = temp.Path,
            Extension = temp.Extension
        };
        _webContext.NewsFiles.Add(insert2);
    }
    _webContext.SaveChanges();      //统一提交数据库
}
```
>新增测试，并查看返回结果
![e48f859506c50879878fbc38152134c0.png](en-resource://database/662:1)
![21d8fa87255c5fd160b0087c29e35cd3.png](en-resource://database/676:1)

## 使用DTO新增数据
>在处理新增数据请求时，部分字段是系统内部自动生成或与业务逻辑紧密关联的（如创建时间、数据状态等），不需要用户手动传入。这些字段如果暴露给用户传入，可能会引发数据安全和一致性问题。为此，我们使用`DTO` 模式，将前端输入参数与后台模型进行分离，确保参数传递更加符合规范并提高数据安全性。

>在NewsFilesDto.cs文件中新增类NewsFilesPostDto，只保留需要传入的字段属性。
```C#
public class NewsFilesPostDto
{
    public string Name { get; set; } = null!;
    public string Path { get; set; } = null!;
    public string Extension { get; set; } = null!;
}
```

>在NewsDto.cs文件中新增类NewsPostDto，只保留需要传入的字段属性。
```C#
public class NewsPostDto
{
    public string Title { get; set; }
    public string Content { get; set; }
    public DateTime StartDateTime { get; set; }
    public DateTime EndDateTime { get; set; }
    public bool Enable { get; set; }
    public int Click { get; set; }
    public ICollection<NewsFilesPostDto> NewsFiles { get; set; } = new List<NewsFilesPostDto>();
}
```
>将传入的参数类型改为NewsPostDto，其他无需修改（无关联外键方法，这里不再对关联外键的方式进行赘述。）
```C#
[HttpPost]
public void Post([FromBody] NewsPostDto value)
{
    //代码内容同上，无需修改。
}
```

## 使用AutoMapper新增数据

>1、在`NewsProfile.cs`中配置`AutoMapper`，并设置默认值：
```C#
CreateMap<NewsPostDto, News>()
.ForMember(dest=>dest.UpdateDateTime,
opt=>opt.MapFrom(src=>DateTime.Now))
.ForMember(dest => dest.UpdateEmployeeId,
opt => opt.MapFrom(src => 1))
.ForMember(dest => dest.InsertDateTime,
opt => opt.MapFrom(src => DateTime.Now))
.ForMember(dest => dest.InsertEmployeeId,
opt => opt.MapFrom(src => 1))
.ForMember(dest => dest.Click,
opt => opt.MapFrom(src => 0));
```
>2、在`NewsFilesProfile.cs`中配置`AutoMapper`：
```C#
CreateMap<NewsFilesPostDto, NewsFiles>();
```
>3、使用AutoMapper转换数据，提交修改。
```C#
[HttpPost("autoMapper")]
public void PostAutoMapper([FromBody] NewsPostDto value)
{
    var map = _mapper.Map<News>(value);

    _webContext.News.Add(map);
    _webContext.SaveChanges();
}
```
4、新增测试：

![abdd88b4f63e4d852bc306a5b1f6dfad.png](en-resource://database/664:1)
![28ab11700e29ae4b31e8ac1c11053a99.png](en-resource://database/674:1)

## 使用内建函数匹配新增数据

>可以使用内建函数`.CurrentValues()`来获取实体的当前状态并进行比较或更新。效果等同于AutoMapper。

> **这里需要注意的是，** 子类不一样的情况下，不会自动新增子数据，像本例中，子数据的类在`News`中为`NewsFiles`，在`NewsPostDto`中为`NewsFilesPostDto`，这种情况下，需要自行分步处理，处理方式如下。

>优化后的代码：
```C#
[HttpPost]
public void Post([FromBody] NewsPostDto value)
{
    News insert = new News()
    {
        UpdateDateTime = DateTime.Now,
        UpdateEmployeeId = 1,
        InsertDateTime = DateTime.Now,
        InsertEmployeeId = 1,
        Click = 0
    };
    _webContext.News.Add(insert).CurrentValues.SetValues(value);
    _webContext.SaveChanges();      //这一步必须做，提交数据库后，父数据才会生成NewsId

    foreach (var temp in value.NewsFiles)
    {
        _webContext.NewsFiles.Add(new NewsFiles()
        {
            NewsId = insert.NewsId,
        }).CurrentValues.SetValues(temp) ;
    }
    _webContext.SaveChanges();      //统一提交数据库
}
```

>新增测试：

![6d9a87d54b9297b07a7f6f1a57885613.png](en-resource://database/668:1)
![4b5e980be658fe47fe5a40a7125c9c55.png](en-resource://database/672:1)