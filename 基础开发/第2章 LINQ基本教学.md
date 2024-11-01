# LINQ基本教学
## 目录

- [LINQ是什么](#LINQ是什么)
- [LINQ的优势](#LINQ的优势)
- [LINQ的两种查询语法](#LINQ的两种查询语法)
- [LINQ的常用操作符](#LINQ的常用操作符)
- [LINQ延迟执行](#LINQ延迟执行)
- [LINQ的扩展性](#LINQ的扩展性)
 
## LINQ是什么？

* **语言集成查询（全名：Language Integrated Query）**

* LINQ的核心思想是将数据查询与处理的逻辑从数据的存储结构中分离出来，使得无论你是操作对象、集合还是数据库，查询的方式都一致，从而提高了代码的可读性和可维护性。

* 用不同数据库举个例子，只要学习了LINQ语法，就可以通过同一个语法查询不同数据库中的数据，在代码中由框架底层去判断连接的数据库类型并解析成该数据库语法，而不需要我们去学习不同的数据库语法，降低了学习成本，并且降低了后续同项目使用不同数据库所带来的迁移成本。

## LINQ的优势

* **统一的查询语法：** 无论是查询数据库（`LINQ to SQL`）、XML文件（`LINQ to XML`）、还是集合（`LINQ to Object`），LINQ都提供了统一的查询语法。
* **类型安全：** LINQ查询在编译时就会进行类型检查，减少运行时错误。
* **可读性和简洁性：** LINQ的语法接近SQL，减少了复杂的循环操作，使代码更易读。
* **延迟执行：** LNQ查询默认时延迟执行的，查询在真正访问数据时才会执行，这可以提高性能。
* **扩展性：** 可以通过自定义的扩展方法来扩展LINQ的功能。

## LINQ的两种查询语法

#### 查询表达式语法
``` C#
// 使用查询表达式语法
var query = from num in numbers
            where num > 2
            orderby num descending
            select num;

foreach (var n in query)
{
    Console.WriteLine(n);
}
```

#### 方法语法（lambda表达式）
``` C#
// 使用方法语法
var query = numbers.Where(num => num > 2).OrderByDescending(num => num);

foreach (var n in query)
{
Console.WriteLine(n);
}
```

## LINQ的常用操作符

#### 筛选操作符

* **Where**：用于筛选符合条件的元素。
``` C#
var result = numbers.Where(num => num > 2);
```
#### 投影操作符

* **Select**：用于将每个元素映射为新的形式。

``` C#
var result = numbers.Select(num => num * 2);
```

* **SelectMany**：用于将集合中的每个子集合展平成单一集合。

``` C#
var result = customers.SelectMany(c => c.Orders);
```

#### 排序操作符

* **OrderBy**：按升序排序。

* **OrderByDescending**：按降序排序。

``` C#
var result = numbers.OrderBy(num => num);
```

#### 集合操作符

* **Distinct**：去除重复的元素。

* **Union**：返回两个集合的并集。

* **Intersect**：返回两个集合的交集。

* **Except**：返回一个集合中存在，但另一个集合中不存在的元素。

``` C#
var result = numbers.Distinct();
```

#### 分组操作符

* **GroupBy**：将集合中的元素按照指定的条件分组。

```C#
var result = numbers.GroupBy(num => num % 2 == 0 ? "Even" : "Odd");
```

#### 聚合操作符

* **Count**：计算元素的个数。

* **Sum**：计算数值集合的总和。

* **Average**：计算数值集合的平均值。

* **Min、Max**：返回集合中的最小值和最大值。 

```C#
var count = numbers.Count();
var sum = numbers.Sum();
```

#### 连接操作符

* **Join**：将两个集合中的元素根据某个条件进行连接。

```C#
var result = customers.Join(orders,
                            customer => customer.Id,
                            order => order.CustomerId,
                            (customer, order) => new { customer.Name, order.OrderId });
```

## LINQ延迟执行

* LINQ查询通常时延迟执行的，这意味着查询本身不会立即执行，而是在枚举（如使用`foreach`遍历）或调用诸如`ToList（）、Count（）`等方法时才会执行。这种延迟的机制可以提高性能，避免不必要的计算。

```C#
var query = from num in numbers
            where num > 5
            select num;

// 查询尚未执行

foreach (var n in query)
{
    Console.WriteLine(n);  // 此时查询才真正执行
}
```

* 如果希望立即执行查询，可以使用`ToList()、ToArray()`等方法来触发查询执行。

```C#
var result = query.ToList();  // 查询立即执行，并返回结果	
```

## LINQ的扩展性

* LINQ操作符本质上是扩展方法，可以根据自己的需求定义自定义的LINQ扩展方法。

* 自定义LINQ扩展方法示例：

```C#
public static class LinqExtensions
{
    public static IEnumerable<T> MyCustomFilter<T>(this IEnumerable<T> source, Func<T, bool> predicate)
    {
        foreach (var item in source)
        {
            if (predicate(item))
            {
                yield return item;
            }
        }
    }
}

// 使用自定义的LINQ方法
var filteredNumbers = numbers.MyCustomFilter(n => n > 5);

foreach (var n in filteredNumbers)
{
    Console.WriteLine(n);  // 输出 6, 7, 8, 9, 10
}
```
