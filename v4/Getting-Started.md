LiteDB 是一个简单、快速、轻量级的嵌入式 .NET 文档数据库。由于受到 MongoDB 数据库的启发，LiteDB API 与 MongoDB 官方 .NET API 差不多（这里指 1.x 版）。

### 如何安装

LiteDB 是一个 serverless 数据库, 不需要安装，只要复制[LiteDB.dll](https://github.com/mbdavid/LiteDB/releases) 到你的 Bin 文件夹并添加引用即可。当然，如果你喜欢的话，也可以通过 NuGet 安装：`Install-Package LiteDB`。如果需要在 Web 环境下运行，请确保 IIS 用户对数据文件所在文件夹有写权限。

### 第一个例子

一个简单的例子，用来展示如何存储和搜索文档：

```C#
// 创建你的 POCO 类实体
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string[] Phones { get; set; }
    public bool IsActive { get; set; }
}

// 打开数据库 (如果不存在自动创建)
using(var db = new LiteDatabase(@"C:\Temp\MyData.db"))
{
    // 获取一个集合 (如果不存在创建)
    var col = db.GetCollection<Customer>("customers");

    // 创建新顾客实例
    var customer = new Customer
    { 
        Name = "John Doe", 
        Phones = new string[] { "8000-0000", "9000-0000" }, 
        IsActive = true
    };
	
    // 插入新顾客文档 (Id 自增)
    col.Insert(customer);
	
    // 更新集合中的一个文档
    customer.Name = "Joana Doe";
	
    col.Update(customer);
	
    // 使用文档的 Name 属性为文档建立索引
    col.EnsureIndex(x => x.Name);
	
    // 使用 LINQ 查询文档
    var results = col.Find(x => x.Name.StartsWith("Jo"));

    // 让我们创建在电话号码字段上创建一个索引 (使用表达式). 它是一个多键值索引
    col.EnsureIndex(x => x.Phones, "$.Phones[*]"); 

    // 现在我们可以查询电话号码
    var r = col.FindOne(x => x.Phones.Contains("8888-5555"));
}
```

### 用于文件存储

需要存储文件？没问题：使用 FileStorage。

```C#
// 从文件系统上传一个文件到数据库
db.FileStorage.Upload("my-photo-id", @"C:\Temp\picture-01.jpg");

// 然后下载
db.FileStorage.Download("my-photo-id", @"C:\Temp\copy-of-picture-01.jpg");
```