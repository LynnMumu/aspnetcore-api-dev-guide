# 删除数据DELETE

## 目录
- [外键级联删除](#外键级联删除)
- [同时删除父子数据](#同时删除父子数据)
- [删除多条指定数据](#删除多条指定数据)

## 外键级联删除
在删除操作中，使用 `[HttpDelete("{id}")]` 路由，接收的参数为 `Guid id`：
```C#
[HttpDelete("{id}")]
public void Delete(Guid id)
{
}
```
查找要删除的记录:
```C#
var delete = (from a in _webContext.News
              where a.NewsId == id
              select a)
              .Include(b=>b.NewsFiles)
              .SingleOrDefault();
```
在这里，`Include(b=>b.NewsFiles) `用于加载相关的子数据。如果表中没有关联的子数据，则可以省略此部分。由于只期望获取一条记录，因此在查询末尾使用 `SingleOrDefault()`。

找到记录后，可以执行删除操作。完整代码如下：
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
删除测试：

![1730440761315](https://github.com/user-attachments/assets/f1d47ae2-0f55-469a-b053-2a1b734ac9b7)
![1730440768207](https://github.com/user-attachments/assets/97740f48-8fec-47dd-b8fd-ea9689702ded)

## 同时删除父子数据
如果无外键的情况下，需要删除父子数据，则先删除子表数据，再删除父表数据。

* 完整代码如下：
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
* 删除测试：

![1730440790871](https://github.com/user-attachments/assets/c6cbbc3a-b4d1-4dab-b61f-1cf8e1743d96)
![1730440798909](https://github.com/user-attachments/assets/2533cd42-9198-485a-a1c5-be2774e8c599)

## 删除多条指定数据
假设我们需要同时删除多条指定数据，此时传入的参数不再是单个 ID，而是一串 ID。我们可以使用 `JsonSerializer` 将该字符串转换为字符串列表。

* 完整代码如下：
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
* 删除测试：

![1730440818768](https://github.com/user-attachments/assets/07186e6f-dfda-45b0-a125-c02e912e90cd)
![1730440825059](https://github.com/user-attachments/assets/d87901f5-7bc2-4dab-acdb-70fa6e249ad1)
![1730440831486](https://github.com/user-attachments/assets/6808b8aa-c588-4142-b4c2-814f3ec521fa)

**需要注意的是**，使用 `Contains` 方法通过LINQ生成的SQL语句中使用了`OPENJSON`方法，但该方法在SQL Server旧版本（2014 或更早版本）中不被支持。

![1730440842566](https://github.com/user-attachments/assets/8e7685f7-3fea-49d7-b3b5-8e4431ccb80a)

可以考虑以下替代方案进行处理：
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
