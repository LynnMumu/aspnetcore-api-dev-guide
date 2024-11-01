# 理解异步Async与Await

## 目录
- [什么是异步编程？](#什么是异步编程？)
- [基本概念](#基本概念)
- [如何定义异步方法](#如何定义异步方法)
- [调用异步方法](#调用异步方法)
- [注意事项](#注意事项)
- [常见用法](#常见用法)
- [同步与异步的简单示例](#同步与异步的简单示例)


## 什么是异步编程？
通过 `async` 和 `await` 关键字来实现的。这种编程方式能够在执行耗时的操作（如网络请求或文件I/O）时，不阻塞主线程，从而提高应用的响应能力。

## 基本概念
- **异步方法**: 用 `async` 修饰的方法，返回类型通常是 `Task` 或 `Task<T>`，表示一个异步操作。
- **等待操作**: 使用 `await` 关键字来等待一个异步操作完成，该操作必须返回 `Task` 或 `Task<T>`。
- **机制**: 由线程池和执行上下文处理异步任务。异步方法不会直接创建新的线程来处理任务，而是依赖于现有的线程池资源

## 如何定义异步方法
```C#
public async Task<int> GetDataAsync()
{
    // 模拟异步操作
    await Task.Delay(1000); // 等待 1 秒
    return 42; // 返回结果
}
```

## 调用异步方法
在调用异步方法时，可以使用 `await` 来等待其完成：
```C#
public async Task ProcessDataAsync()
{
    int result = await GetDataAsync(); // 等待异步方法完成
    Console.WriteLine($"Result: {result}"); // 打印结果
}
```
## 注意事项
- **异常处理**: 异步操作可能会引发异常，使用 `try-catch` 块来捕获错误。
- **上下文**: 在某些情况下，`await` 会捕获当前上下文（如 UI 线程），在继续执行时可能会在原来的上下文中恢复。
- **避免死锁**: 在使用 `await` 时，如果在同步上下文（如 UI 线程）中调用异步方法，可能会导致死锁。通常情况下，推荐在异步方法中使用 `ConfigureAwait(false)` 来避免这种问题。

## 常见用法
- **文件 I/O**: 读取和写入文件时使用异步方法，例如 `File.ReadAllTextAsync()`。
- **数据库操作**: 使用异步数据库访问库，如 Entity Framework Core 提供的异步查询方法。
- **事件处理**: 在事件处理程序中使用异步方法，确保事件处理快速返回而不阻塞 UI。

## 同步与异步的简单示例

1. 同步示例：
```C#
namespace WebAPITest.Services
{
    public class AsyncService
    {
        private int test1()
        {
            Task.Delay(1000).Wait();
            return 1;
        }
        private int test2()
        {
            Task.Delay(2000).Wait();
            return 2;
        }
        private int test3()
        {
            Task.Delay(3000).Wait();
            return 3;
        }

        public int Test()
        {
            int result1 = test1();
            int result2 = test2();
            int result3 = test3();
            return result1 + result2 + result3;
        }
    }
}
```
```C#
using Microsoft.AspNetCore.Mvc;
using WebAPITest.Services;

namespace WebAPITest.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class AsyncController : ControllerBase
    {
        private readonly AsyncService _asyncService;
        public AsyncController(AsyncService asyncService)
        {
            _asyncService = asyncService;
        }
        [HttpGet]
        public int Get()
        {
            return _asyncService.Test();
        }
    }
}
```
从测试返回结果来看，整个执行过程大约耗时6秒：
![be9d7c9e9a508f718400a0bd9be00612.png](en-resource://database/778:1)

2. 将同步方法改写为异步方法后：

```C#
namespace WebAPITest.Services
{
    public class AsyncService
    {
        private async Task<int> test1Async()
        {
            await Task.Delay(1000);
            return 1;
        }
        private async Task<int> test2Async()
        {
            await Task.Delay(2000);
            return 2;
        }
        private async Task<int> test3Async()
        {
            await Task.Delay(3000);
            return 3;
        }

        public async Task<int> TestAsync()
        {
            var result1 = test1Async();
            var result2 = test2Async();
            var result3 = test3Async();
            return await result1 + await result2 + await result3;
        }
    }
}
```
```C#
using Microsoft.AspNetCore.Mvc;
using WebAPITest.Services;

namespace WebAPITest.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class AsyncController : ControllerBase
    {
        private readonly AsyncService _asyncService;
        public AsyncController(AsyncService asyncService)
        {
            _asyncService = asyncService;
        }
        [HttpGet]
        public async Task<int> Get()
        {
            return await _asyncService.TestAsync();
        }

    }
}
```
从测试返回结果来看，整个执行过程大约耗时3秒，这就是最长作业的执行时间：
![501ae4b991c5bda986dca0366b757ddf.png](en-resource://database/780:1)

* 需要注意的是，使用异步方法时，所有从上到下的调用都必须采用异步形式。另外，`await` 的位置非常重要。如果将 `await` 放置在以下位置，程序的行为将与同步方法相同，依然是耗时6秒。
```C#
public async Task<int> TestAsync()
{
    var result1 = await test1Async();
    var result2 = await test2Async();
    var result3 = await test3Async();
    return  result1 + result2 + result3;
}
```