![1730440141659](https://github.com/user-attachments/assets/b9ff01be-6642-43e7-a723-9336a2924352)# 新增数据POST

## 目录
- [新增数据](#新增数据)
- [新增子数据和同时新增父子数据](#新增子数据和同时新增父子数据)
- [使用DTO新增数据](#使用DTO新增数据)
- [使用AutoMapper新增数据](#使用AutoMapper新增数据)
- [使用内建函数匹配新增数据](#使用内建函数匹配新增数据)


## 新增数据

简单新增News表的数据：
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
1. 新增测试，返回200说明执行成功了：

![1730439985010](https://github.com/user-attachments/assets/3bd7f071-b481-4e60-b400-99980469d5eb)
2. 查询是否能找到这条记录：

![1730440045265](https://github.com/user-attachments/assets/01d7f97f-9ed7-40c5-a09c-6cc562fcb8e4)

在实际应用中，某些数据（如`UpdateDateTime`、`UpdateEmployeeId`、`InsertDateTime`、`InsertEmployeeId`和`Click`）不适合由用户手动输入，因此通常会进行默认设置。这种默认设置的优化可以通过以下方式实现：
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
1. 新增测试，分别给`UpdateDateTime`、`UpdateEmployeeId`、`InsertDateTime`、`InsertEmployeeId`和`Click`字段传入值：

![1730440077774](https://github.com/user-attachments/assets/ff4494aa-3835-41c9-a3ce-568b5f3469f3)

2. 查询新增的这条数据，可以看到`Click`字段的值是系统默认设置的：

![1730440141659](https://github.com/user-attachments/assets/993f70bd-644d-452f-aab4-add4107d0a79)

## 新增子数据和同时新增父子数据

#### 新增子数据

* 创建控制器`NewsFilesController.cs`，简单新增`NewsFiles`表的数据：
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
* 新增数据，并查看返回结果：

![1730440180800](https://github.com/user-attachments/assets/d7d0b6ce-9a47-455e-9421-aff5efcb76c3)
![1730440187686](https://github.com/user-attachments/assets/fc58265e-ed3f-4ec3-985c-c0cabe2d425a)
 
 #### 同时新增父子数据
1. 父子数据表**存在**外键的情况下，代码如下：
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
2. 父子数据表**不存在**外键的情况下，代码如下:
 
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
3. 新增测试，并查看返回结果
![1730440229061](https://github.com/user-attachments/assets/bd4377cc-0c0e-4e6d-940b-e6ae81b70124)
![1730440235848](https://github.com/user-attachments/assets/aab5ca13-9949-4756-b287-28091d4e24a8)

## 使用DTO新增数据
在处理新增数据请求时，部分字段是系统内部自动生成或与业务逻辑紧密关联的（如创建时间、数据状态等），不需要用户手动传入。这些字段如果暴露给用户传入，可能会引发数据安全和一致性问题。为此，我们使用`DTO` 模式，将前端输入参数与后台模型进行分离，确保参数传递更加符合规范并提高数据安全性。

1. 在NewsFilesDto.cs文件中新增类NewsFilesPostDto，只保留需要传入的字段属性。
```C#
public class NewsFilesPostDto
{
    public string Name { get; set; } = null!;
    public string Path { get; set; } = null!;
    public string Extension { get; set; } = null!;
}
```

2. 在NewsDto.cs文件中新增类NewsPostDto，只保留需要传入的字段属性。
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

3. 将传入的参数类型改为NewsPostDto，其他无需修改（无关联外键方法，这里不再对关联外键的方式进行赘述。）
```C#
[HttpPost]
public void Post([FromBody] NewsPostDto value)
{
    //代码内容同上，无需修改。
}
```

## 使用AutoMapper新增数据

1. 在`NewsProfile.cs`中配置`AutoMapper`，并设置默认值：
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
2. 在`NewsFilesProfile.cs`中配置`AutoMapper`：
```C#
CreateMap<NewsFilesPostDto, NewsFiles>();
```
3. 使用AutoMapper转换数据，提交修改。
```C#
[HttpPost("autoMapper")]
public void PostAutoMapper([FromBody] NewsPostDto value)
{
    var map = _mapper.Map<News>(value);

    _webContext.News.Add(map);
    _webContext.SaveChanges();
}
```
4. 新增测试：
![1730440310993](https://github.com/user-attachments/assets/6ebcdacc-5c04-4103-8463-fc542ea0cc90)
![1730440316426](https://github.com/user-attachments/assets/d0133fde-ca45-49db-82b3-effeea66f82c)

## 使用内建函数匹配新增数据

可以使用内建函数`.CurrentValues()`来获取实体的当前状态并进行比较或更新。效果等同于AutoMapper。

**这里需要注意的是，** 子类不一样的情况下，不会自动新增子数据，像本例中，子数据的类在`News`中为`NewsFiles`，在`NewsPostDto`中为`NewsFilesPostDto`，这种情况下，需要自行分步处理，处理方式如下。

* 优化后的代码：
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

* 新增测试：
![1730440377622](https://github.com/user-attachments/assets/f4aca09a-6138-440e-918e-1ca0a11a950d)
![1730440382795](https://github.com/user-attachments/assets/0b59c9c9-25f4-4212-ad8f-cf40210bb00a)
