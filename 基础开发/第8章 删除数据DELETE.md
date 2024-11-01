# 删除数据DELETE

## 目录
- [外键级联删除](#外键级联删除)
- [同时删除父子数据](#同时删除父子数据)
- [删除多条指定数据](#删除多条指定数据)

## 外键级联删除
>在删除操作中，使用 `[HttpDelete("{id}")]` 路由，接收的参数为 `Guid id`：
```C#
[HttpDelete("{id}")]
public void Delete(Guid id)
{
}
```
>查找要删除的记录:
```C#
var delete = (from a in _webContext.News
              where a.NewsId == id
              select a)
              .Include(b=>b.NewsFiles)
              .SingleOrDefault();
```
>在这里，`Include(b=>b.NewsFiles) `用于加载相关的子数据。如果表中没有关联的子数据，则可以省略此部分。由于只期望获取一条记录，因此在查询末尾使用 `SingleOrDefault()`。
>找到记录后，可以执行删除操作。完整代码如下：
```C#
[HttpDelete("{id}")]
public void Delete(Guid id)
{
    var delete = (from a in _webContext.News
                  where a.NewsId == id
                  select a)
                  .Include(b=>b.NewsFiles)
                  .SingleOrDefault();
    if (delete != null)
    {
        _webContext.News.Remove(delete);
        _webContext.SaveChanges();
    }
}
```
>删除测试：

![8bfe54280ff72edd5a717c4fdad09d4a.png](en-resource://database/702:1)
![c9e814bc14f55f523a7f8ae9dcd55b91.png](en-resource://database/704:1)

## 同时删除父子数据
>如果无外键的情况下，需要删除父子数据，则先删除子表数据，再删除父表数据。
>完整代码如下：
```C#
[HttpDelete("{id}")]
public void Delete(Guid id)
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
    _webContext.SaveChanges();
}
```
>删除测试：

![d03d67ac48bfa724102ac14093713479.png](en-resource://database/706:1)
![34b72f972c50da5f938086d71572e83e.png](en-resource://database/708:1)

## 删除多条指定数据
>假设我们需要同时删除多条指定数据，此时传入的参数不再是单个 ID，而是一串 ID。我们可以使用 `JsonSerializer` 将该字符串转换为字符串列表。
>完整代码如下：
```C#
[HttpDelete("list/{ids}")]
public void DeleteList(string ids)
{
    var deleteList = JsonSerializer.Deserialize<List<Guid>>(ids);
    var delete = (from a in _webContext.News
                  where deleteList.Contains(a.NewsId)
                  select a).Include(a => a.NewsFiles).ToList();
    _webContext.News.RemoveRange(delete);
    _webContext.SaveChanges();
}
```
>删除测试：

![a9067dba61fedf0c48db4b835ec0a187.png](en-resource://database/710:1)
![01a146140f5d5b8c88f224ede4720783.png](en-resource://database/712:1)
![b70629f5024af7bf9773ca4f25e757ce.png](en-resource://database/714:1)

>**需要注意的是**，使用 `Contains` 方法通过LINQ生成的SQL语句中使用了`OPENJSON`方法，但该方法在SQL Server旧版本（2014 或更早版本）中不被支持。

![ab73654d10ef105db5fdfe72c12e64e4.png](en-resource://database/716:1)

>可以考虑以下替代方案进行处理：
```C#
[HttpDelete("list/{ids}")]
public void DeleteList(string ids)
{
    var deleteList = JsonSerializer.Deserialize<List<Guid>>(ids);
    foreach (Guid id in deleteList)
    {
        var delete = (from a in _webContext.News
                      where a.NewsId == id
                      select a).Include(a => a.NewsFiles);
        _webContext.News.RemoveRange(delete);
    }
    
    _webContext.SaveChanges();
}
```