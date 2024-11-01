# 获取数据GET

## 目录
- [获取数据列表](#获取数据列表)
- [获取指定数据](#获取指定数据)
- [通过关键字获取数据](#通过关键字获取数据)
- [参数化并过滤接收值](#参数化并过滤接收值)
- [From之常用来源标签功能用法介绍](#From之常用来源标签功能用法介绍)
- [使用AutoMapper自动对应DTO栏位](#使用AutoMapper自动对应DTO栏位)
- [同时获取父子数据](#同时获取父子数据)
- [DbContext获取数据库数据时的注意事项](#DbContext获取数据库数据时的注意事项)
## 获取数据列表

简单获取News表中所有数据：
```C#
[HttpGet]
public IEnumerable<News> Get()
{
return _webContext.News;
}
```
![1730439105443](https://github.com/user-attachments/assets/1d29965d-4a48-4eb8-9aee-b8cd896deb83)

在实际应用中，我们往往不希望将所有字段都暴露给外部使用者。一些字段可能只是内部记录或用于系统维护的用途，并不适合对外展示。
 
比如这个例子中，我希望隐藏`InsertDateTime、UpdateDateTime` 和 `Enable` 字段，因为它们都是内部使用的资料，没必要提供给外部使用者查看。那么就可以有以下写法：
```C#
[HttpGet]
public IEnumerable<object> Get()
{
var result = from a in _webContext.News
                select new
                {
                    a.NewsId,
                    a.Title,
                    a.Content,
                    a.StartDateTime,
                    a.EndDateTime,
                    a.Click
                };
return result;
}
```
然而，这种写法会带来一个可读性和维护性的问题。具体来说，如果我们没有明确定义数据的类型结构，会大大降低代码的可维护性。假如同样的字段过滤逻辑在多个地方重复使用，却没有定义统一的类型，那么日后的维护和扩展将会更加困难。

虽然在项目初期定义类型会增加一些工作量，但从长期来看，明确的类型定义对于代码的可读性、可靠性和一致性是非常必要的。

为了解决这个问题，我们可以创建`NewsDto`类来定义输出的数据结构。为了保持项目结构清晰，我们还可以在项目中建立一个`Dtos`文件夹，专门存放这些数据传输对象（DTO）。接着，我们可以在`NewsDto`中，定义出需要返回的字段，以明确哪些数据是外部可见的。通过这种方式，我们可以在多个地方复用相同的DTO，确保代码的一致性和易维护性。
![1730439135869](https://github.com/user-attachments/assets/75086a3f-fa46-4b8d-8ff1-ed138b9842d6)
```C#
namespace WebAPITest.Dtos
{
    public class NewsDto
    {
        public Guid NewsId { get; set; }
        public string Title { get; set; }
        public string Content { get; set; }
        public DateTime StartDateTime { get; set; }
        public DateTime EndDateTime { get; set; }
        public int Click { get; set; }
    }
}
```

因此，我们可以对原有的API代码进行改写，使其符合新的DTO结构。
```C#
[HttpGet]
public IEnumerable<NewsDto> Get()
{
    var result = from a in _webContext.News
                    select new NewsDto
                    {
                        NewsId = a.NewsId,
                        Title = a.Title,
                        Content = a.Content,
                        StartDateTime = a.StartDateTime,
                        EndDateTime = a.EndDateTime,
                        Click = a.Click
                    };
    return result;
}
```
当然，如果表的字段很多，并且多个地方都需要将数据转换成DTO进行展示，那么在每个地方手动构造DTO会导致代码冗余，显得不够简洁。
为了解决这个问题，我们可以将DTO转换逻辑封装成一个通用的函数，以提升代码的可读性和复用性。
``` C#
private static NewsDto GetNewsDto(News news)
{
    return new NewsDto()
    {
        NewsId = news.NewsId,
        Title = news.Title,
        Content = news.Content,
        StartDateTime = news.StartDateTime,
        EndDateTime = news.EndDateTime,
        Click = news.Click
    };
}
```
在NewsController控制器中使用DTO转换函数：
```C#
[HttpGet]
public IEnumerable<object> Get()
{
    return _webContext.News.Select(a=> GetNewsDto(a)).ToList();
}
```
通常建议在执行 `ToList()` 方法时调用转换函数和其他自定义函数，这种做法有助于降低出错的可能性。

通过这样的操作，可以确保在数据完全从数据库中检索出来之后，再进行进一步的处理和转换，从而提高整体代码的稳定性和可靠性。

## 获取指定数据

根据传入id获取News表的指定数据：
```C#
[HttpGet("{id}")]
public News Get(Guid id)
{
    return _webContext.News.Find(id);
}
```

然而，这里存在一个问题，我们应该使用 DTO，因此需要调整一下我们的写法。
```C#
[HttpGet("{id}")]
public NewsDto Get(Guid id)
{
    var result = (from a in _webContext.News
                    where a.NewsId == id
                    select new NewsDto
                    {
                        Title = a.Title,
                        Content = a.Content,
                        NewsId = a.NewsId,
                        StartDateTime = a.StartDateTime,
                        EndDateTime = a.EndDateTime,
                        Click = a.Click
                        }).Single(); //只获取单条数据
    return result;
}
```
这样就能获取到指定的单条数据。

![1730439208876](https://github.com/user-attachments/assets/a283b5be-e815-47e9-8d37-6aa11bfa4bfd)

不过，当找不到指定的 ID 时，这会导致错误。

![1730439222018](https://github.com/user-attachments/assets/8164056b-84eb-4092-83b1-471f0c6c5d35)

问题出在使用 `.Single() `方法，因为它假设一定会返回一条记录，如果没有找到，则会导致程序出错。因此，我们需要换一种写法。
```C#
[HttpGet("{id}")]
public NewsDto Get(Guid id)
{
    var result = (from a in _webContext.News
                    where a.NewsId == id
                    select new NewsDto
                    {
                        Title = a.Title,
                        Content = a.Content,
                        NewsId = a.NewsId,
                        StartDateTime = a.StartDateTime,
                        EndDateTime = a.EndDateTime,
                        Click = a.Click
                        }).SingleOrDefault(); //只获取单条数据，但是找不到时会返回null
    return result;
}
```
这样就算没获取到数据也不会报错了。

![1730439245737](https://github.com/user-attachments/assets/7de6204d-b99f-4670-90d3-fb16d4f14df9)

通常在获取指定数据的API中，会使用`.SingleOrDefault()`方法，因为它允许返回`null`，以处理未找到记录的情况。而使用`.Single()`方法则适用于那些你认为必须返回一条记录的场景。如果没有找到记录，意味着前面的逻辑有问题，因此使用`.Single()`可以阻止程序继续执行。

## 通过关键字获取数据
通常，我们会使用`GET`方法通过`URL`传递需要过滤的关键字。在`URL`的末尾加上`?`，并且使用`&`符号来连接多个参数。例如：`api/news?id=xxxx&id2=xxxx`。

在这个例子中，id时字段名称，而xxxx则是对应的参数。

有了这些基础知识后，我们可以开始构建我们的API参数。将针对`News`数据表的`Title`、`Content`和`StartDateTime`进行过滤。因此，URL可以写成`api/news?Title=xxxx&Content=xxxx&StartDateTime=xxxx`。

>接下来，在GET方法中，添加接收参数的代码。
```C#
[HttpGet]
public IEnumerable<NewsDto> Get(string? title, string? content, DateTime? startDateTime)
{
    var result = from a in _webContext.News
                 select new NewsDto
                 {
                     Title = a.Title,
                     Content = a.Content,
                     NewsId = a.NewsId,
                     StartDateTime = a.StartDateTime,
                     EndDateTime = a.EndDateTime,
                     Click = a.Click
                 };

    return result;
}
```
这里传入的参数命名大小写不敏感，视为等效处理。

在该函数的参数定义中标明对并的字段名称和类型，其中在类型后加上`?`表示该参数为可选参数。如果调用时未传递该参数，则其默认值为`null`，这让我们在实现逻辑时可以灵活判断和处理。

这里需要注意的是，如果项目启用了可空引用类型支持（**建议开启**）（通常在 `csproj` 中启用），并且参数可能为`null`，应明确标注为`string?`以消除编译器警告，并表明设计意图。
![image](https://github.com/user-attachments/assets/45da4801-cdd8-4853-8a01-eab1de1aa1fd)

* **我们将捞取全部资料，并逐步根据条件进行过滤。**

1. 过滤空白或空值的标题（Title）：首先检查每条数据的Title是否为空或空值，如果是，则直接过滤掉。
2. 依条件进行进一步过滤：
```C#
if (!string.IsNullOrWhiteSpace(title))
{
    result = result.Where(a => a.Title == title);
}
```
3. 接下来判断`content`是否为空或空值， 并对其进行过滤。这里的写法采用了`a.Content.Contains(content)`，与`title`的处理方式不同——`title`需要完全匹配，而`content`只要部分匹配即可满足规则。不过实际上，`title`的匹配逻辑通常也会使用部分匹配，这里为了演示效果，故使用了`==`完全匹配作为示例。
```C#
if (!string.IsNullOrWhiteSpace(content))
{
    result = result.Where(a => a.Content.Contains(content));
}
```
4. 最后，判断startDateTime是否不为null，若不为null，则进行过滤。这里的过滤逻辑仅匹配日期部分。

```C#
if (startDateTime != null)
{
   result = result.Where(a => a.StartDateTime.Date == ((DateTime)startDateTime).Date);
}
```
* 测试title参数：

![1730439302668](https://github.com/user-attachments/assets/44413bc6-a31f-43d0-ae63-3fcf829c6e7e)

* 测试多个参数：

![1730439316443](https://github.com/user-attachments/assets/731f24e7-27f9-46a1-b6d7-60aa4c9c3983)

* 完整的代码：
```C#
[HttpGet]
public IEnumerable<NewsDto> Get(string? title, string? content, DateTime? startDateTime)
{
    var result = from a in _webContext.News
                 select new NewsDto
                 {
                     Title = a.Title,
                     Content = a.Content,
                     NewsId = a.NewsId,
                     StartDateTime = a.StartDateTime,
                     EndDateTime = a.EndDateTime,
                     Click = a.Click
                 };
    if (!string.IsNullOrWhiteSpace(title))
    {
        result = result.Where(a => a.Title == title);
    }

    if (!string.IsNullOrWhiteSpace(content))
    {
        result = result.Where(a => a.Content.Contains(content));
    }
    if (startDateTime != null)
    {
        result = result.Where(a => a.StartDateTime.Date == ((DateTime)startDateTime).Date);
    }
    return result;
}
```
这样，当没有传入参数的时候，依然会正常获取全部数据，有参数的时候则进行过滤。

## 参数化并过滤接收值

假设，GET方法若要新增更多参数，如endDateTime和click，且未来可能有更多字段，用简单的逐个参数传递的方式会导致代码臃肿，降低可维护性。为了提升代码的可读性和可维护性，我们可以将它封装成单独的一个类。

新建一个Parameters文件夹，存放参数类文件，并新建一个NewsParameter.cs。

建完后，当前项目的目录结构如下：

![1730439331699](https://github.com/user-attachments/assets/85e8cc2c-6db5-45db-9b3c-daa111e0aa0b)

* 封装参数：
```C#
namespace WebAPITest.Parameters
{
    public class NewsParameter
    {
        public string? Title { set; get; }
        public string? Content { set; get; }
        public DateTime? StartDateTime { set; get; }
        public DateTime? EndDateTime { get; set; }
        public int? Click { get; set; }
    }
}
```
* 引用：
```C#
[HttpGet]
public IEnumerable<NewsDto> Get(NewsParameter value)
{
}
```
这样的代码结构就更加清晰和可维护了。将参数封装成类，能有效减少代码冗余，并确保未来当字段或参数发送变更时，维护成本最低。

而在GET请求中使用[FromQuery]注解时一个关键步骤，它可以确保控制器能够正确接收和绑定传入的查询参数。
``` C#
[HttpGet]
public IEnumerable<NewsDto> Get([FromQuery]NewsParameter value)
{
    var result = from a in _webContext.News
                 select new NewsDto
                 {
                     Title = a.Title,
                     Content = a.Content,
                     NewsId = a.NewsId,
                     StartDateTime = a.StartDateTime,
                     EndDateTime = a.EndDateTime,
                     Click = a.Click
                 };
    if (!string.IsNullOrWhiteSpace(value.Title))
    {
        result = result.Where(a => a.Title == value.Title);
    }

    if (!string.IsNullOrWhiteSpace(value.Content))
    {
        result = result.Where(a => a.Content.Contains(value.Content));
    }
    if (value.StartDateTime != null && value.EndDateTime != null)
    {
        result = result.Where(a => a.StartDateTime.Date >= ((DateTime)value.StartDateTime).Date && a.EndDateTime.Date <= ((DateTime)value.EndDateTime).Date);
    }
    if (value.Click != null)
    {
        result = result.Where(a => a.Click >= value.Click);
    }
    return result.ToList();
}
```
* 测试新增参数：

![1730439345993](https://github.com/user-attachments/assets/646b2258-c49d-43d2-9b95-db8c13894e48)

接下来，进行一些调整。目前，传参时需要填写`startDateTime=2024-09-28&endDateTime=2024-09-29`，这种方式显得繁琐。因此，可以将其简化为一个字段，例如：`Time=[2024-09-28,2024-09-29]`，不仅简化了参数结构，还提升了可读性。在`NewsParameter.cs`中添加如下变量，并在其中进行处理。
``` C#
private string? _time;
public string? Time
{
    get { return _time; }
    set
    {
        Regex regex = new Regex(@"\[(\d{4}-\d{2}-\d{2}),(\d{4}-\d{2}-\d{2})\]");
        if (regex.Match(value).Success)
        {
            string[] dates = value.Trim('[',']').Split(',');
            StartDateTime = DateTime.Parse(dates[0]).Date;
            EndDateTime = DateTime.Parse(dates[1]).Date;
        }
        _time = value;
    }
}
```
当接收到`Time`值时，将通过正则表达式进行验证，确保其格式符合如：`[2024-09-28,2024-09-29]`。如果符合这一规则，将分别将`“，”`左侧的时间赋值给`StartDateTime`，`“，”`右侧的时间赋值给`EndDateTime`。因此，处理后的结果是`StartDateTime=2024-09-28`和`EndDateTime=2024-09-29`。这样，可以有效的从格式为`Time=[2024-09-28,2024-09-29]`的字符串中提取出开始时间和结束时间。

* 执行结果如下：

![1730439357543](https://github.com/user-attachments/assets/061de64b-9bd6-4e86-95e4-f3f2798fd581)

将处理Time的逻辑放在NewsParameter.cs中可以增强模块的内聚力，并降低与其他部分的耦合性。这样做的好处是，所有与参数相关的操作都集中在一个地方，便于未来的维护和修改。如果需要对参数处理逻辑进行更新或调整，只需要修改NewsParameter.cs文件。这种做法提升了代码的清晰度和可管理性。

## From之常用来源标签功能用法介绍

#### 什么是From？
**From属性是用来指定从哪里获取参数值的绑定源。**

#### 基本功能介绍

| 标签 |描述  |例子 |
| --- | --- |--- |
| * `[FromBody]`  |用于从请求的正文中获取数据，常用于处理JSON或XML格式的复杂数据结构。  |例如，POST请求正文中包含的JSON对象可以被绑定到一个模型上。|
| * `[FromForm]`  |用于从POST请求的表单数据获取参数。  |当表单被提交时，可以使用此标签来获取表单中的数据。从form-data中接收参数。|
| * `[FromQuery] ` |用于从URL的查询字符串获取参数。 | 如果API请求地址为`https://localhost:7240/api/news?click=23`，则可以使用`[FromQuery] int click`来获取这个参数。|
| * `[FromRoute] ` |用于从路由数据获取参数。 | 如果在路由模板中定义了参数，例如`[HttpGet("users/{id}")]`，则可以使用`[FromRoute] int id`来获取这个id。 |
|`[FromServices]`  |这不是用来绑定客户端发送的数据，而是用于从依赖注入容器中自动解析服务。  | 可以在控制器方法的参数中使用此标签来直接注入服务。|
|``[FromHeader]``|用于从HTTP请求头中获取数据 | 如果需要访问某些特定的请求头信息，如Authorization，可以使用此标签。|

常用到的是`[FromBody]、[FromForm]、[FromQuery]、[FromRoute]`。

在ASP.NET Core中，数据绑定标签（如`From`标签）其实是有默认行为的，通常情况下，不需要显示地使用这些标签，系统会根据参数的位置和类型自动推断数据的来源。这样可以简化控制器方法的代码。

但是如果是像上面将传入的参数封装成类后，系统默认会从请求体（`Body`）中获取这些参数，这是使用` [FromBody] `属性的行为。如果我们需要从`URL`的查询字符串（`Query String`）中获取参数，就必须明确指定使用` [FromQuery] `属性。这样，我们就可以确保方法能正确地从`URL`获取参数，而不是错误地尝试从请求体中解析它们。这个明确的指定是必要的，尤其是在参数可能来自不同来源的情况下，以避免错误和混淆。

## 使用AutoMapper自动对应DTO栏位

这里将介绍一个名为AutoMapper的实用工具包，它在使用数据传输对象（DTO）时显得尤为重要。

通常，我们需要手动将各个字段从源数据映射到DTO，当字段数量较少时，这种方法尚可管理，但一旦字段增多，手动映射的工作量巨大，容易出错且效率低下。

通过使用AutoMapper，可以自动化这一转换过程，显著提高开发效率并减少错误，从而使代码更加简洁和可维护。

#### 安装和使用

1. 需要通过NuGet包管理器安装AutoMapper库，选择`AutoMapper`。

![1730439367096](https://github.com/user-attachments/assets/8028236f-a802-4332-902f-1a337bd957a3)

2. 在Program.cs文件中，添加以下配置：
```C#
builder.Services.AddAutoMapper(typeof(Program));
```
这将注册AutoMapper服务并与当前启动类关联。

3. 创建一个名为Profiles的文件夹，用于集中存放所有的AutoMapper配置文件。这有助于代码的组织和维护，使不同对象映射的配置更易于查找和管理，确保项目结构更清晰。同时，将每个映射配置定义为独立的Profile类，能够保持单一职责原则，让不同类型的映射逻辑各自独立，避免混乱。

这样做的好处是可以把对象映射的配置管理集中化，提高代码的可读性和可维护性，特别是在项目规模较大、涉及多种映射关系时，更能体现其优势。

![1730439373986](https://github.com/user-attachments/assets/ffe58a4b-56aa-472b-ad62-7cfd3a69612c)

4. 创建一个名为`NewsProfile.cs`的类，并让它继承自`AutoMapper`提供的`Profile`类。

```C#
using AutoMapper;
using WebAPITest.Dtos;
using WebAPITest.Models;

namespace WebAPITest.Profiles
{
    public class NewsProfile:Profile
    {
        public NewsProfile()
        {
            CreateMap<News, NewsDto>();
        }
    }
}
```
在`NewsProfile.cs`中添加`CreateMap<News, NewsDto>();`的代码片段，可以明确地告诉`AutoMapper`，`News`和`NewsDto`之间存在映射关系。

假设`AutoMapper`映射过程中，有部分参数需要进行额外处理（如将 `News` 对象中的 `StartDateTime` 属性映射到 `NewsDto` 的 `StartDateTime` 属性时，仅获取日期部分），可以使用`CreateMap<News, NewsDto>()`中配置`.ForMember()`方法。
```C#
CreateMap<News, NewsDto>()
    .ForMember(dest=>dest.StartDateTime,            //处理后的字段名
    opt=>opt.MapFrom(src=>src.StartDateTime.Date))  //数据来源及处理
    .ForMember(dest => dest.EndDateTime,
    opt => opt.MapFrom(src => src.EndDateTime.Date));
```

5. 依赖注入 `AutoMapper` 实例，保证在当前类中可以方便地使用 `AutoMapper` 的映射功能：

```C#
private readonly WebContext _webContext;

private readonly IMapper _mapper;

public NewsController(WebContext webContext,IMapper mapper)
{
    _webContext = webContext; 
    _mapper = mapper;
}
```

6. 使用`AutoMapper` 的映射功能：
```C#
[HttpGet]
public IEnumerable<NewsDto> Get([FromQuery]NewsParameter value)
{
    var result = from a in _webContext.News select a;
    if (!string.IsNullOrWhiteSpace(value.Title))
    {
        result = result.Where(a => a.Title == value.Title);
    }

    if (!string.IsNullOrWhiteSpace(value.Content))
    {
        result = result.Where(a => a.Content.Contains(value.Content));
    }
    if (value.StartDateTime != null && value.EndDateTime != null)
    {
        result = result.Where(a => a.StartDateTime.Date >= ((DateTime)value.StartDateTime).Date && a.EndDateTime.Date <= ((DateTime)value.EndDateTime).Date);
    }
    if (value.Click != null)
    {
        result = result.Where(a => a.Click >= value.Click);
    }
    return _mapper.Map<IEnumerable<NewsDto>>(result); //如果是单条数据，写法为：_mapper.Map<NewsDto>(result);
}
```
这样，AutoMapper 就能够自动将 News 的属性转换为 NewsDto，这有助于减少手动编写映射代码的工作量，提高开发效率。

## 同时获取父子数据

如何在数据库查询中同时提取父对象及其关联的子对象数据呢？

以News表为例，每个新闻可能包含多个上传的文件`NewsFiles`。

以下将展示如何将这两者的关联数据一同查询出来。

1. 为`NewsFiles`创键一个对应的DTO。`NewsFilesDto`将用于表示从数据库获取的上传文件数据，使其可以方便地传递到前端或者其他层。
```C#
namespace WebAPITest.Dtos
{
    public class NewsFilesDto
    {
        public Guid NewsFilesId { get; set; }
        public Guid NewsId { get; set; }
        public string Name { get; set; } = null!;
        public string Path { get; set; } = null!;
        public string Extension { get; set; } = null!;
    }
}
```
2. 在`NewsDto.cs`中新增属性`NewsFiles`用于展示上传的文件数据。
```C#
namespace WebAPITest.Dtos
{
    public class NewsDto
    {
        public Guid NewsId { get; set; }
        public string Title { get; set; }
        public string Content { get; set; }
        public DateTime StartDateTime { get; set; }
        public DateTime EndDateTime { get; set; }
        public int Click { get; set; }
        public virtual ICollection<NewsFilesDto> NewsFiles { get; set; }
    }
}
```
3. 在`News.cs`中新增属性`NewsFiles`用于关联上传的文件数据。
```C#
using System;
using System.Collections.Generic;

namespace WebAPITest.Models;

public partial class News
{
    public Guid NewsId { get; set; }
    public string Title { get; set; } = null!;
    public string Content { get; set; } = null!;
    public DateTime StartDateTime { get; set; }
    public DateTime EndDateTime { get; set; }
    public DateTime UpdateDateTime { get; set; }
    public int UpdateEmployeeId { get; set; }
    public DateTime InsertDateTime { get; set; }
    public int InsertEmployeeId { get; set; }
    public bool Enable { get; set; }
    public int Click { get; set; }
    public virtual ICollection<NewsFiles> NewsFiles { get; set; } = new List<NewsFiles>();
} 
```
4. 获取上传文件数据：
* 将NewsFiles的DTO转换逻辑封装成一个通用的函数
```C#
private static NewsFilesDto GetNewsFilesDto(NewsFiles files)
{
    return new NewsFilesDto()
    {
        NewsFilesId = files.NewsFilesId,
        NewsId = files.NewsId,
        Name = files.Name,
        Path = files.Path,
        Extension = files.Extension
    };
}
```
* 修改GetNewsDto函数，新增附件属性。
private static NewsDto GetNewsDto(News news)
{
    return new NewsDto()
    {
        NewsId = news.NewsId,
        Title = news.Title,
        Content = news.Content,
        StartDateTime = news.StartDateTime,
        EndDateTime = news.EndDateTime,
        Click = news.Click,
        NewsFiles = news.NewsFiles.Select(a => GetNewsFilesDto(a)).ToList()
    };
}

* **如果实体之间存在外键关系**，那么要将父子数据一并查询出来是非常简单的。只需要在查询时使用`.Include()`方法来包含关联的子数据。

```C#
var result = _webContext.News
    .Include(a=>a.NewsFiles)
    .Select(a=>a);
```

将实体对象转换为DTO对象。

```C#
return result.Select(a => GetNewsDto(a)).ToList();
```

完整的代码：
```C#
[HttpGet]
public IEnumerable<NewsDto> Get([FromQuery]NewsParameter value)
{
    var result = _webContext.News
        .Include(a=>a.NewsFiles)
        .Select(a=>a);

    if (!string.IsNullOrWhiteSpace(value.Title))
    {
        result = result.Where(a => a.Title == value.Title);
    }

    if (!string.IsNullOrWhiteSpace(value.Content))
    {
        result = result.Where(a => a.Content.Contains(value.Content));
    }
    if (value.StartDateTime != null && value.EndDateTime != null)
    {
        result = result.Where(a => a.StartDateTime.Date >= ((DateTime)value.StartDateTime).Date && a.EndDateTime.Date <= ((DateTime)value.EndDateTime).Date);
    }
    if (value.Click != null)
    {
        result = result.Where(a => a.Click >= value.Click);
    }
    return result.Select(a => GetNewsDto(a)).ToList();
}
```

* **如果实体之间不存在外键关系**,那么就直接使用LINQ获取数据即可，同时在获取的过程中完成DTO转换。

```C#
[HttpGet]
public IEnumerable<NewsDto> Get([FromQuery] NewsParameter value)
{
    var result = _webContext.News.Select(a => a);

    var files = _webContext.NewsFiles.ToList();

    if (!string.IsNullOrWhiteSpace(value.Title))
    {
        result = result.Where(a => a.Title == value.Title);
    }

    if (!string.IsNullOrWhiteSpace(value.Content))
    {
        result = result.Where(a => a.Content.Contains(value.Content));
    }
    if (value.StartDateTime != null && value.EndDateTime != null)
    {
        result = result.Where(a => a.StartDateTime.Date >= ((DateTime)value.StartDateTime).Date && a.EndDateTime.Date <= ((DateTime)value.EndDateTime).Date);
    }
    if (value.Click != null)
    {
        result = result.Where(a => a.Click >= value.Click);
    }

    var temp = (from a in result.ToList()
                select new NewsDto 
                {
                    NewsId = a.NewsId,
                    Title = a.Title,
                    Content = a.Content,
                    StartDateTime = a.StartDateTime,
                    EndDateTime = a.EndDateTime,
                    Click = a.Click,
                    Enable = a.Enable,
                    NewsFiles = (from b in files
                                 where b.NewsId == a.NewsId
                                 select new NewsFilesDto
                                 {
                                     NewsFilesId = b.NewsFilesId,
                                     NewsId = b.NewsId,
                                     Name = b.Name,
                                     Path = b.Path,
                                     Extension = b.Extension
                                 }).ToList()
                });
    return temp;
}
```

返回结果：
![1730439424360](https://github.com/user-attachments/assets/7452ad48-5f1c-4c89-befa-06cb6faf8f8a)

## DbContext获取数据库数据时的注意事项

这里主要是了解下关于GridView.RowDataBound的陷阱。

#### GridView.RowDataBound 的基本用法

* RowDataBound 事件在每一行数据绑定时触发，这使得开发者可以在数据行渲染时进行自定义操作，例如改变行的样式或内容。

#### 常见性能问题

* 在循环中重复读取数据库。

#### 问题分析
* 在 RowDataBound 中，如果每一行都从数据库中查询数据，尤其是在有两三百个单位的情况下，可能会导致显著的性能下降。因为每一次查询都会涉及到数据库连接、查询执行等开销，因此在处理大量数据时，响应时间会大幅增加。

#### 解决方案

* **预先加载数据**：将所有需要的数据一次性从数据库中读取到内存中，然后在 RowDataBound 中直接使用这些数据。这种方法可以大大减少数据库的访问次数，提高性能。
* **使用缓存**：如果数据变化不频繁，可以考虑使用缓存机制，将数据库中的数据缓存到内存中，减少重复的数据库查询。

#### 数据量平衡
>在处理大量数据时需要注意内存的使用：

* **分页加载**：对于大量数据（例如上万条），可以考虑分页加载，将数据分成小块进行处理。这样可以在不消耗过多内存的情况下，仍然提供良好的用户体验。
* **动态加载**：结合前端技术（如 `AJAX`）实现动态加载，根据用户滚动或操作来加载更多数据，从而避免一次性加载过多数据。

#### 与 Web API 的关系
使用 GridView.RowDataBound 的例子可以引申到 Web API 的开发中：

* 在使用 Web API 时，避免在处理请求时进行重复的数据库查询，尤其是在循环中（如`foreach`）。可以通过在请求处理逻辑中先查询所有需要的数据，然后进行处理，来提高性能。


#### 结论

**在开发中，性能优化是一个重要的考虑因素。通过合理设计数据读取和处理的流程，能够有效提高应用的响应速度和用户体验。**



