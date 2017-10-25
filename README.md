偶发兴趣，对LiteDB 文档的翻译，详见[wiki页面](https://github.com/lidanger/LiteDB.wiki_Translation_zh-cn/wiki)

原项目地址：https://github.com/mbdavid/LiteDB
======

# LiteDB - 一个单数据文件 .NET NoSQL 文档存储

[![Join the chat at https://gitter.im/mbdavid/LiteDB](https://badges.gitter.im/mbdavid/LiteDB.svg)](https://gitter.im/mbdavid/LiteDB?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge) [![Build status](https://ci.appveyor.com/api/projects/status/sfe8he0vik18m033?svg=true)](https://ci.appveyor.com/project/mbdavid/litedb) [![Build Status](https://travis-ci.org/mbdavid/LiteDB.svg?branch=master)](https://travis-ci.org/mbdavid/LiteDB)

LiteDB 一个小巧、快速、轻量级的 NoSQL 嵌入式数据库。 

- Serverless NoSQL 文档存储
- 类似于 MongoDB 的简单 API
- 100% C# 代码，支持.NET 3.5 / .NET 4.0 / NETStandard 1.3 / NETStandard 2.0，单 DLL (小于 300kb)
- 支持线程和进程安全
- 支持文档/操作级别的 ACID
- 支持写失败后的数据还原 (日志模式)
- 可使用 DES (AES) 加密算法进行数据文件加密
- 可使用特性或 fluent 映射 API 将你的 POCO类映射为 `BsonDocument`
- 可存储文件与流数据 (类似 MongoDB 的 GridFS)
- 单数据文件存储 (类似 SQLite)
- 支持基于文档字段索引的快速搜索 (每个集合支持多达16个索引)
- 支持 LINQ 查询
- Shell 命令行 - [试试这个在线版本](http://www.litedb.org/#shell)
- 相当快 - [这里是与 SQLite 的对比结果](https://github.com/mbdavid/LiteDB-Perf)
- 开源，对所有人免费 - 包括商业应用
- 可以从 NuGet 安装：`Install-Package LiteDB`

## 4.0 新特性
- 新的 `表达式/路径` 索引/查询支持。参考[表达式](https://github.com/mbdavid/LiteDB/wiki/Expressions)
- 支持 `Include` 嵌套
- 优化的查询执行过程 (包含简单的调试说明)
- 修复并发问题
- 移除事务和索引自动创建
- 支持全扫描搜索和 LINQ 搜索
- 新的 shell 命令：基于表达式选择/转换文档以及更新字段
- 浏览全部[更新日志](https://github.com/mbdavid/LiteDB/wiki/Changelog)

## 在线试用

[试试 LiteDB Web Shell](http://www.litedb.org/#shell). 由于安全原因，在线版本中不是所有命令都是可用的。请使用离线版本进行全特性测试。

## 文档

访问 [Wiki](https://github.com/mbdavid/LiteDB/wiki) 即可获取全部文档。简单中文版本，[请看这里](https://github.com/lidanger/LiteDB.wiki_Translation_zh-cn)。

## 下载

可以在 [LiteDB Releases](https://github.com/mbdavid/LiteDB/releases) 下载源代码或二进制文件。

## LiteDB 使用入门

一个存储、搜索文档的简单示例：

```C#
// 创建你的 POCO 类
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
    public string[] Phones { get; set; }
    public bool IsActive { get; set; }
}

// 打开数据库 (如果不存在则创建)
using(var db = new LiteDatabase(@"MyData.db"))
{
    // 获得 customer 集合
    var col = db.GetCollection<Customer>("customers");

    // 创建你的新 customer 实例
	  var customer = new Customer
    { 
        Name = "John Doe", 
        Phones = new string[] { "8000-0000", "9000-0000" }, 
        Age = 39,
        IsActive = true
    };
    
    // 在 Name 字段上创建唯一索引
    col.EnsureIndex(x => x.Name, true);
	
    // 插入新的 customer 文档 (Id 是自增的)
    col.Insert(customer);
	
    // 更新集合中的一个文档
    customer.Name = "Joana Doe";
	
    col.Update(customer);
	
    // 使用 LINQ 查询文档 (未使用索引)
    var results = col.Find(x => x.Age > 20);
}
```

使用 fluent 映射器和跨文档引用处理更复杂的数据模型

```C#
// DbRef 交叉引用
public class Order
{
    public ObjectId Id { get; set; }
    public DateTime OrderDate { get; set; }
    public Address ShippingAddress { get; set; }
    public Customer Customer { get; set; }
    public List<Product> Products { get; set; }
}        

// 重用全局实例的映射器
var mapper = BsonMapper.Global;

// "Produts" 和 "Customer" 来自其他集合 (而不是嵌入的文档)
mapper.Entity<Order>()
    .DbRef(x => x.Customer, "customers")   // 1对1/0引用
    .DbRef(x => x.Products, "products")    // 1对多引用
    .Field(x => x.ShippingAddress, "addr"); // 嵌入的子文档
            
using(var db = new LiteDatabase("MyOrderDatafile.db"))
{
    var orders = db.GetCollection<Order>("orders");
        
    // 当查询 Order 时，包含引用
    var query = orders
        .Include(x => x.Customer)
        .Include(x => x.Products) // 1对多引用
        .Find(x => x.OrderDate <= DateTime.Now);

    // 每个 Order 实例都会加载 Customer/Products 引用
    foreach(var order in query)
    {
        var name = order.Customer.Name;
        ...
    }
}

```

## 应用场景

- 桌面/本地的小应用程序
- 应用程序文件格式（Application file format）
- 小型 web 应用程序
- **每个账户/用户**一个数据库的数据存储
- 少量并发写操作

## 插件

- 一个 GUI 查看器工具: https://github.com/falahati/LiteDBViewer
- 一个 GUI 编辑器工具: https://github.com/JosefNemec/LiteDbExplorer 
- Lucene.NET 目录: https://github.com/sheryever/LiteDBDirectory
- LINQPad 支持: https://github.com/adospace/litedbpad

## 更新日志

每个发布版本的变化详情都记录在[发行说明](https://github.com/mbdavid/LiteDB/releases)中。

## 许可证

[MIT](http://opensource.org/licenses/MIT)

Copyright (c) 2017 - Maurício David

## 致谢

特别感谢 @negue 和 @szurgot 对可移植版本提供的帮助，以及 @lidanger 的简体中文翻译。 
