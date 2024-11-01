# 使用ActionResult返回数据

## 目录
- [什么是ActionResult？](#什么是ActionResult？)
- [为什么要使用ActionResult？](#为什么要使用ActionResult？)
- [关于IActionResult和ActionResult的区别](#关于IActionResult和ActionResult的区别)
- [优化增删改查代码](#优化增删改查代码)

## 什么是ActionResult？

`ActionResult` 是 ASP.NET Core 中用于表示控制器方法返回结果的基类。它允许开发者在处理请求时返回不同类型的响应，从而实现灵活的HTTP响应。

## 为什么要使用ActionResult？

#### 1.灵活性
ActionResult 提供了多种返回类型的支持，包括：

* `JsonResult`：返回JSON格式的数据。

* `ViewResult`：返回视图，用于渲染HTML页面。

* `FileResult`：返回文件下载或流式文件。

* `StatusCodeResult`：返回特定的HTTP状态码。

* `ContentResult`：返回纯文本或其他内容类型。

这种灵活性允许开发者根据需要选择最合适的响应类型，简化了处理不同请求的逻辑。

#### 2.清晰的状态管理

使用 ActionResult 可以清晰地表示操作的结果和HTTP状态码。例如：

* 使用 `Ok() `返回 200 状态码，表示请求成功。

* 使用 `NotFound()` 返回 404 状态码，表示资源未找到。

* 使用 `BadRequest()` 返回 400 状态码，表示请求错误。

这样可以使客户端（例如前端应用）更容易理解服务器响应的状态，进而做出相应处理。

#### 3.支持模型状态验证
在ASP.NET Core中，ActionResult 可以与模型状态验证一起使用，这使得处理请求时更为简便。如果请求模型无效，可以直接返回 `BadRequest`，避免了手动检查和返回。

>代码示例：
```C#
public ActionResult UpdateUser(UserModel userModel)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState); // 自动处理无效状态
    }

    // 处理有效请求
}
```

#### 4.简化代码
使用 ActionResult 可以减少代码的复杂性和冗余。因为可以根据操作结果直接返回不同类型的响应，无需额外的条件逻辑来处理不同的返回情况。

>代码示例：
```C#
public ActionResult GetUser(int id)
{
    var user = FindUserById(id);
    if (user == null)
    {
        return NotFound(); // 清晰简洁的状态返回
    }
    return Ok(user); // 返回用户数据
}
```

#### 5.适应不同的客户端请求
ActionResult 使得服务器可以根据不同的客户端需求返回不同类型的数据。例如，移动应用可能更倾向于接收`JSON`，而网页应用可能需要完整的`HTML`视图。

#### 6.支持标准化API开发
在RESTful API开发中，标准的`HTTP`状态码和响应格式对于API的可用性至关重要。ActionResult 提供了与`HTTP`协议的良好集成，使得API的行为符合行业标准，增强了可读性和易用性。

#### 总结
使用 ActionResult 使得ASP.NET Core中的控制器方法更加灵活、清晰和易于维护，能够有效处理多种返回类型和`HTTP`状态，提供更好的用户体验和开发效率。这是构建健壮Web应用程序和API的重要手段。

## 关于IActionResult和ActionResult的区别


| IActionResult | ActionResult  |
| --- | --- |
| IActionResult 是一个接口，定义了一个控制器操作可以返回的结果的契约。它允许不同的实现类（如 `JsonResult`、`ViewResult`、`RedirectResult` 等）来实现这个接口，从而使得控制器返回各种不同类型的结果。 |  ActionResult 是一个具体的类，它实现了 IActionResult 接口。它提供了一些便捷的方法，如 `Ok()`、`NotFound()`、`BadRequest()` 等，用于生成常见的响应类型。|
|使用 IActionResult 可以实现更大的灵活性，允许控制器方法返回任何实现了IActionResult 接口的类型。这意味着可以在一个方法中使用不同的返回类型，而不需要明确指定具体的返回类型。  | ActionResult 可以封装多种结果类型，提供一种更为方便的方式来构造响应。使用 ActionResult 使得返回不同类型的结果变得更加直观，因为它已经提供了多种常用的响应方法。 |
|通常用于方法签名中，以允许返回多种不同的结果类型。  |适用于需要返回多种类型响应的情况，尤其是在需要返回数据或视图的控制器方法中。  |

## 优化增删改查代码

* **GET方法优化：**
```C#
[HttpGet]
public ActionResult<IEnumerable<NewsDto>> Get([FromQuery] NewsParameter value)
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
    if (result != null && result.Count() >= 0)
    {
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
        return Ok(temp);
    }
    else
    {
        return NotFound();
    }
}
```
* **POST方法优化：**
```C#
[HttpPost]
public ActionResult<NewsDto> Post([FromBody] NewsPostDto value)
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
    return CreatedAtAction(nameof(Get), new { NewsId = insert.NewsId }, insert);
}
```
>`CreatedAtAction` 用于在控制器中返回一个` HTTP 201 Created` 响应。它通常用于创建资源后，向客户端返回新创建资源的位置信息。这个方法不仅会返回资源的表示，还会生成包含一个 Location 头的响应，指向新创建资源的 `URI`，以便客户端可以使用该 `URI` 进行后续的操作。

* `nameof(Get)`：指定生成的 `URI` 应指向 `Get` 方法。
* `new { NewsId = insert.NewsId }`：传递新的 `NewsId`，以便生成完整的 `URI`。
* `insert`：将新创建的News对象作为响应内容返回。

>测试返回：

![8345580d27dddf85a8a0cf7d74e98fef.png](en-resource://database/720:1)
![00fd4b66e5b151499e2a7cd22761857e.png](en-resource://database/722:1)

* **PUT方法优化：**
```C#
[HttpPut("{id}")]
public ActionResult Put(Guid id, [FromBody] NewsPutDto value)
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
        return NoContent();
    }
    else
    {
        return NoContent();
    }
}
```
* **DELETE方法优化：**
```C#
[HttpDelete("{id}")]
public ActionResult Delete(Guid id)
{
    //删除子表数据
    var deletefile = (from a in _webContext.NewsFiles
                      where a.NewsId == id
                      select a).SingleOrDefault();
    if (deletefile != null)
    {
        _webContext.NewsFiles.RemoveRange(deletefile);
    }
    //删除主表数据
    var delete = (from a in _webContext.News
                  where a.NewsId == id
                  select a).SingleOrDefault();
    if (delete != null)
    {
        _webContext.News.Remove(delete);
    }
    else
    {
        return NotFound();
    }
    _webContext.SaveChanges();
    return NoContent();
}
```