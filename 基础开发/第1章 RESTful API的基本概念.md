# RESTful API的基本概念

## 目录
- [简介](#简介)
- [基本概念](#基本概念)



## 简介

RESTful是一种设计和开发Web服务的架构风格，它基于`HTTP`协议，使用`HTTP`请求（如`GET、POST、PUT、DELETE`等）与服务器进行交互。REST代表**REpresentational State Transfer（表述性状态转移）**，它是一种轻量级的方式来创建可扩展、可维护的Web API。

RESTful服务使用标准的`HTTP`方法来执行常见的`CRUD`操作（创建、读取、更新和删除），并使用`URL`定位资源。REST是一种无状态的通信方式，即每个请求都包含了所有必要的信息，服务器不保存客户端的会话状态。

## 基本概念

#### 1.资源（Resource）

* 资源是REST架构的核心概念，它指的是网络上可操作的任何对象。资源可以是数据、文档、图像、视频、用户等。

* 每个资源通过唯一的URL进行标识。

* 可以通过不同的表示（Representation）形式来传递，如JSON、XML、HTML等。

* 资源的URL示例：

    
``` 
https://api.example.com/users/123  // 资源的唯一标识，代表ID为123的用户
```



#### 2.资源的表示（Representation）

资源可以通过多种格式来表述，最常用的是`JSON`和`XML`。

服务器根据客户端请求的内容类型（如`Content-Type: application/json`），将资源转换为相应的表示形式。

#### 3.URI（统一资源标识符）

* 每个资源通过URI（**Uniform Resource Identifier**）进行唯一标识。

* URI不应该包含动词，应该是名词，因为它表示的是资源，而不是操作。

* URI设计的良好示例：
``` 
GET https://api.example.com/users/    // 获取所有用户
GET https://api.example.com/users/123 // 获取ID为123的用户
POST https://api.example.com/users/   // 创建新用户
PUT https://api.example.com/users/123 // 更新ID为123的用户
DELETE https://api.example.com/users/123 // 删除ID为123的用户
```

#### 4.HTTP方法（HTTP Verbs）

* RESTful API通过使用标准的`HTTP`方法来执行对资源的操作。以下是常用的`HTTP`方法及其对应的操作：

|HTTP方法  |操作  |标签  |说明  |
| --- | --- | --- | --- |
|GET  |读取（Read）  |[HttpGet][HttpGet(“{id}”)]  |用于从服务器检索资源。  |
|POST  |创建（Create）  |[HttpPost]  |用于向服务器发送数据，并创建新资源。  |
|PUT  |更新（Update/Replace）  |[HttpPut]  |用于更新资源，通常是替换资源的全部内容。  |
|PATCH  |更新（Update/Modify）  |[HttpPatch]  |用于更新资源的部分内容。  |
|DELETE  |删除（Delete） |[HttpDelete]  |用于从服务器删除资源。  |

#### 5.无状态（Stateless）

* REST架构是无状态的，每个请求都必须包含所有必要的信息（如认证信息、状态信息等），服务器不保存任何客户端的状态

* 这使得RESTful服务易于扩展，因为服务器之间不需要共享会话数据。

#### 6.使用HTTP状态码

* RESTful API通常使用标准的HTTP状态码来表示请求的结果。

* 常见的状态码包括：

| 状态码 |  含义                           |
|--------|----------------------------------|
| 200    | 成功 (OK)                       |
| 201    | 已创建 (Created)                |
| 204    | 无内容 (No Content)             |
| 400    | 错误请求 (Bad Request)          |
| 401    | 未授权 (Unauthorized)           |
| 403    | 禁止访问 (Forbidden)            |
| 404    | 资源未找到 (Not Found)          |
| 500    | 服务器内部错误 (Internal Server Error) |

#### 7.内容协商（Content Negotiation）

* RESTful API支持多种资源的表示形式，如`JSON、XML`等。

* 客户端可以通过设置`Accept`头来指定希望接收的格式，服务器将根据这个头部返回合适的内容。

* 示例：

API请求：
``` C#
GET / users / 123
Accept: application / json // 请求返回JSON格式
```
服务器返回： 
``` C#
{
  "id": 123,

  "name": "John Doe",

  "email": "john@example.com"
}
```