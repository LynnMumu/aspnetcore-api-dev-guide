# DTO基本教学

## 目录
- [DTO是什么？](#DTO是什么？)
- [DTO的作用](#DTO的作用)
- [DTO的使用场景](#DTO的使用场景)
- [DTO示例](#DTO示例)

## DTO是什么？

DTO（**Data Transfer Object，数据传输对象**）是一种设计模式，用在不同系统、层之间传递数据。DTO的目的是仅携带必要的数据，避免传递过程中暴露不必要的内部细节。DTO常用于分层架构，如控制器与服务层之间的交互。

## DTO的作用

* **数据封装**：避免暴露数据库模型或业务模型的内部实现，提升安全性。

* **简化数据传输**：减少网络传输的数据量，只传递必要的数据。

* **解耦**：将不同层的模型解耦，不直接暴露数据库实体模型，提升代码的可维护性和可扩展性。

* **格式转换**：用于数据的转换或格式化，如将复杂对象中的部分数据提取出来供前端使用。

## DTO的使用场景

* **Web API**：控制器将服务层的数据转换为DTO返回给客户端，确保客户端只看到需要的数据。

* **微服务之间的通信**：不同服务之间传递数据时，使用DTO避免传递内部模型。

* **持久层与业务逻辑层之间的数据传递**：在复杂的分层应用中，DTO充当数据载体。

## DTO示例

假设我们有一个User实体，但我们不希望直接将数据库模型暴露给前端（如隐藏用户密码）。我们可以创建一个UserDto用于数据传输。

* **实体类（Entity）**
```C#
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public string Password { get; set; }  // 不想暴露给客户端
}
```

* **DTO类**
```C#
public class UserDto
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
}
```

* **在服务层中转换实体为DTO**
```C#
public class UserService
{
    public UserDto GetUserDto(User user)
    {
        return new UserDto
        {
            Id = user.Id,
            Name = user.Name,
            Email = user.Email
        };
    }
}
```

* **在控制器中使用DTO**
```C#
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly UserService _userService;

    public UsersController(UserService userService)
    {
        _userService = userService;
    }

    [HttpGet("{id}")]
    public IActionResult GetUser(int id)
    {
        var user = new User
        {
            Id = id,
            Name = "John Doe",
            Email = "john@example.com",
            Password = "123456"
        };

        var userDto = _userService.GetUserDto(user);
        return Ok(userDto);  // 返回给客户端的DTO对象
    }
}
```
