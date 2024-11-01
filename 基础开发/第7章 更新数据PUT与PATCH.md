# 更新数据PUT与PATCH

## 目录
- [使用PUT更新数据](#使用PUT更新数据)
- [使用DTO更新数据](#使用DTO更新数据)
- [使用AutoMapper更新数据](#使用AutoMapper更新数据)
- [使用内建函数匹配更新数据](#使用内建函数匹配更新数据)
- [使用PATCH局部更新数据](#使用PATCH局部更新数据)

## 使用PUT更新数据
在更新数据时，我们首先通过路由传递要更新的 ID 和内容，遵循 RESTful 规范：
```C#
 [HttpPut("{id}")]
 public void Put(Guid id, [FromBody] News value)
 {
 }
```
路由中包含要更新的` Guid id`，而 `value` 是用户传递的更新数据。
通常情况下，我们可以使用简单的方法进行更新：
```C#
 [HttpPut("{id}")]
 public void Put(Guid id, [FromBody] News value)
 {
     _webContext.News.Update(value);
     _webContext.SaveChanges();
 }
```
但这通常不符合我们的需求，因此我们需要采用更精确的更新方式：
```C#
var update = (from a in _webContext.News
              where a.NewsId == id
              select a ).SingleOrDefault();
```
首先，获取要更新的记录。如果找到该记录，则更新其属性：
```C#
if (update != null)
{
update.UpdateDateTime = DateTime.Now;
update.UpdateEmployeeId = 1;

update.Title = value.Title;
update.Content = value.Content;
update.StartDateTime = value.StartDateTime;
update.EndDateTime = value.EndDateTime;
update.Enable = value.Enable;
_webContext.SaveChanges();
}
```
如果未找到记录，则不执行任何操作；如果记录存在，则进行更新。在上述代码中，系统会自动设置某些属性值，而 `update.Title = value.Title;` 则是使用用户传来的数据进行更新。最后，调用 `_todoContext.SaveChanges() `将修改保存至数据库。

* 完整代码示例：
```C#
[HttpPut("{id}")]
public void Put(Guid id, [FromBody] News value)
{
    var update = (from a in _webContext.News
                  where a.NewsId == id
                  select a ).SingleOrDefault();
    if (update != null)
    {
        update.Click = update.Click+1;
        update.UpdateDateTime = DateTime.Now;
        update.UpdateEmployeeId = 1;

        update.Title = value.Title;
        update.Content = value.Content;
        update.StartDateTime = value.StartDateTime;
        update.EndDateTime = value.EndDateTime;
        update.Enable = value.Enable;
        _webContext.SaveChanges();
    }
}
```
* 更新测试：

![1730440525725](https://github.com/user-attachments/assets/56f7e395-fdff-43a5-9f8b-56352967cd8d)
![1730440532709](https://github.com/user-attachments/assets/3152a7c1-226f-4370-90fe-3c70dae3cf5a)

通过这种方式，能够精确地更新目标记录，并确保只有在记录存在的情况下才会进行操作。

## 使用DTO更新数据
1. 在NewsDto.cs文件中新增类NewsPutDto，只保留需要传入的字段属性。
```C#
public class NewsPutDto
{
    public Guid NewsId { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public DateTime StartDateTime { get; set; }
    public DateTime EndDateTime { get; set; }
    public bool Enable { get; set; }
    public ICollection<NewsFilesPutDto> NewsFiles { get; set; } = new List<NewsFilesPutDto>();
}
```
2. 在NewsFilesDto.cs文件中新增类NewsFilesPutDto，只保留需要传入的字段属性。
```C#
public class NewsFilesPutDto
{
    public Guid NewsFilesId { get; set; }
    public Guid NewsId { get; set; }
    public string Name { get; set; } = null!;
    public string Path { get; set; } = null!;
    public string Extension { get; set; } = null!;
}
```
3. 将传入的参数类型改为NewsPostDto，其他无需修改。
```C#
[HttpPut("{id}")]
public void Put(Guid id, [FromBody] NewsPutDto value)
{
    //代码内容同上，无需修改。
}
```

将新增和修改操作分为两个 DTO 的主要目的是为了**实现不同的验证逻辑和结果反馈**。
这种做法带来了以下几个优势：

* **1.针对性验证**：新增和修改的业务需求和验证规则往往不同，使用不同的 DTO 可以针对性地设计验证逻辑，从而提高数据准确性。

* **2.清晰的接口设计**：分开 DTO 有助于接口的清晰性，使得调用者可以明确区分新增和修改操作，从而避免混淆。

* **3.灵活的结果反馈**：不同的操作可能需要返回不同的反馈信息，使用不同的 DTO 可以根据具体操作定制化反馈内容，提升用户体验。

* **4.易于维护和扩展**：当业务需求发生变化时，独立的 DTO 使得调整和扩展变得更加简单，不会影响到其他操作的逻辑。

## 使用AutoMapper更新数据

1. 在`NewsProfile.cs`中配置`AutoMapper`，并设置默认值：
```C#
CreateMap<NewsPutDto, News>()
    .ForMember(dest => dest.UpdateDateTime,
    opt => opt.MapFrom(src => DateTime.Now))
    .ForMember(dest => dest.UpdateEmployeeId,
    opt => opt.MapFrom(src => 1));
```
2. 在`NewsFilesProfile.cs`中配置`AutoMapper`：
```C#
CreateMap<NewsFilesPutDto, NewsFiles>();
```
3. 使用AutoMapper转换数据，提交修改。
```C#
[HttpPut("{id}/autoMapper")]
public void PutAutoMapper(Guid id, [FromBody] NewsPutDto value)
{

    var update = (from a in _webContext.News
                  where a.NewsId == id
                  select a).SingleOrDefault();
    if (update != null)
    {
        _mapper.Map(value, update);
        _webContext.SaveChanges();
    }
}
```
4. 更新测试：
![1730440596217](https://github.com/user-attachments/assets/8202a106-8b7e-4e2e-a386-2191fd1f885f)
![1730440601789](https://github.com/user-attachments/assets/a51a6d7e-faa0-43f9-9aae-7356fd8e1f33)

使用` _mapper.Map(value, update) `的语句可以实现将源对象 `value` (待更新的数据对象)的属性映射到目标对象 `update`(需要被更新的现有对象)。通过 AutoMapper 的映射功能，将 value 中的属性值复制到 update 中对应的属性上

## 使用内建函数匹配更新数据

* 优化后的代码：
```C#
[HttpPut("{id}")]
public void Put(Guid id, [FromBody] NewsPutDto value)
{
    var update = (from a in _webContext.News
                  where a.NewsId == id
                  select a ).SingleOrDefault();
    if (update != null)
    {
        update.UpdateDateTime = DateTime.Now;
        update.UpdateEmployeeId = 1;

        _webContext.Update(update).CurrentValues.SetValues(value);
        _webContext.SaveChanges();
    }
}
```
* 更新测试：

![1730440624924](https://github.com/user-attachments/assets/6385e274-ebc0-4003-93d5-1d09ed4c61bc)
![1730440631962](https://github.com/user-attachments/assets/5ec605f9-05f3-482c-94c9-77d17131f699)


## 使用PATCH局部更新数据
这里仅提供一个简单的介绍，更多详细用法请参考官方文档中的 JsonPatch 部分。

1. 通过NuGet安装程序包`Microsoft.AspNetCore.JsonPatch`。

![1730440668217](https://github.com/user-attachments/assets/791d497d-f13f-4b02-a935-6c31164236b7)

* 完整局部更新代码：
```C#
  [HttpPatch("{id}")]
 public void Patch(Guid id, [FromBody] JsonPatchDocument value)
 {
     var update = (from a in _webContext.News
                   where a.NewsId == id
                   select a).SingleOrDefault();
     if (update != null)
     {
         update.UpdateDateTime = DateTime.Now;
         update.UpdateEmployeeId = 1;

         value.ApplyTo(update);

         _webContext.SaveChanges();
     }
 }
```
2. 通过NuGet安装程序包`Microsoft.AspNetCore.Mvc.NewtonsoftJson`。

![1730440677861](https://github.com/user-attachments/assets/5fce3a34-c477-4f98-8ace-0cb445d5f240)

3. 在Program.cs中配置并注入相关依赖。
```C#
builder.Services.AddControllersWithViews().AddNewtonsoftJson();
```

* `AddControllersWithViews()`：添加 MVC 控制器和视图的服务。

* `AddNewtonsoftJson()`：启用 `Newtonsoft.Json` 作为 JSON 处理程序，使其可以用于序列化和反序列化 JSON 数据。

4. 局部更新数据：

![1730440688611](https://github.com/user-attachments/assets/5feb8583-54b6-42db-a81c-56d67334ce58)
![1730440694875](https://github.com/user-attachments/assets/b53eb501-2f25-4ef5-809b-8c97c8f00581)


