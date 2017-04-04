LiteDB是一个简单、快速、轻量级的嵌入式.NET文档数据库。LiteDB受到MongoDB启发，因此它的API非常类似于MongoDB的官方.NET API。

### 如何安装

LiteDB是一个无服务器数据库，因此不需要安装。只需要复制[LiteDB.dll](https://github.com/mbdavid/LiteDB/releases)到你的Bin文件夹并添加引用即可。当然，如果你喜欢的话，也可以通过NuGet安装：`Install-Package LiteDB`。如果你要在一个Web环境下运行，请确保你的IIS用户对数据文件夹有写权限。

### 第一个例子

一个快捷示例，存储和搜索文档：

```C#
// 创建你的POCO类实体
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string[] Phones { get; set; }
    public bool IsActive { get; set; }
}

// 打开数据库（或不存在创建它）
using(var db = new LiteDatabase(@"C:\Temp\MyData.db"))
{
	// 获取一个集合（或不存在而创建它）
	var col = db.GetCollection<Customer>("customers");

        // 创建你的新自定义实例
	var customer = new Customer
        { 
            Name = "John Doe", 
            Phones = new string[] { "8000-0000", "9000-0000" }, 
            IsActive = true
        };
	
	// 插入一个新的自定义文档(Id是自增的)
	col.Insert(customer);
	
	// 更新一个集合中的一个文档
	customer.Name = "Joana Doe";
	
	col.Update(customer);
	
	// 使用文档的Name属性为文档创建索引
	col.EnsureIndex(x => x.Name);
	
	// 使用LINQ查询文档
	var results = col.Find(x => x.Name.StartsWith("Jo"));
}
```

### 使用文件

需要存储文件？没问题：使用FileStorage。

```C#
// 从文件系统上传一个文件到数据库
db.FileStorage.Upload("my-photo-id", @"C:\Temp\picture-01.jpg");

// 过一会儿下载
db.FileStorage.Download("my-photo-id", @"C:\Temp\copy-of-picture-01.jpg");
```
