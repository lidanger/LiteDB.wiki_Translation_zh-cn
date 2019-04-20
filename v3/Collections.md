文档被存储和组织在集合中。`LiteCollection`在LiteDB中是一个管理集合的通用类。每一个集合必须有一个唯一名称：

- 只包含字母，数字和`_`
- 集合名称是 **区分大小写的**
- 以`_`开头的集合名称被保留，内部使用

全部集合名称加起来被限制在3000个字节以内。如果你有很多集合的话，请使用短名称。如果每一个集合名称使用大约10个字节，在一个单数据库中，你可以拥有~300个集合。

集合在第一个`Insert` 或 `EnsureIndex`操作时自动创建。在一个不存在的的集合中运行查询，删除或更新并不会创建集合。

对无模式文档来说，`LiteCollection<T>`是一个通用类，它作为`BsonDocument`可以与`<T>`一起使用。LiteDB在内部会转换`T`为`BsonDocument`，并且所有操作都使用这个通用文档。

在这个例子中，两个代码片段产生同样的结果。

```C#
// 强类型类
using(var db = new LiteDatabase("mydb.db"))
{
    // 获取集合实例
    var col = db.GetCollection<Customer>("customer");
    
    // 向集合插入一个文档 - 如果集合不存在，现在创建
    col.Insert(new Customer { Id = 1, Name = "John Doe" });
    
    // 如果Name字段不存在索引，在Name字段上创建新索引
    col.EnsureIndex(x => x.Name);
    
    // 现在，搜索你想要的文档
    var customer = col.FindOne(x => x.Name == "john doe");
}

// 通用 BsonDocument
using(var db = new LiteDatabase("mydb.db"))
{
    // 获取集合实例
    var col = db.GetCollection("customer");
    
    // 向集合插入一个文件 - 如果集合不存在，现在创建
    col.Insert(new BsonDocument().Add("_id", 1).Add("Name", "John Doe"));
    
    // 如果Name字段不存在索引，在Name字段上创建新索引
    col.EnsureIndex("Name");
    
    // 现在，搜索你想要的文档
    var customer = col.FindOne(Query.EQ("Name", "john doe"));
}
```
### LiteDatabase API 实例方法

- **`GetCollection<T>`** - 这个方法返回`LiteCollection`的一个实例。如果省略`<T>`，`<T>`为`BsonDocument`。这是仅有的获取一个集合实例的方式。
- **`RenameCollection`** - 仅重命名一个集合名称 - 不改变任何文档
- **`CollectionExists`** - 检查一个集合是否已经存在于数据库中
- **`GetCollectionNames`** - 获取数据库中的所有集合名称
- **`DropCollection`** - 删除数据库上的所有文档，所有索引和集合引用

### LiteCollection API 实例方法

- **`Insert`** - 插入一个新文档或一个文档`IEnumberable`。如果你的文档没有`_id`字段，Insert操作会使用`ObjectId`创建一个新的。如果你有一个映射过的对象，可以使用 `AutoId`。参考 [Object Mapping](Object-Mapping)
- **`InsertBulk`** - 用于插入大量文档。将文档分批次，每一批控制一次事务。这个方法每批插入后会清理缓存以保持内存使用率。
- **`Update`** - 更新按`_id`字段标识的文档。如果没找到，返回false
- **`Delete`** - 按`_id`或一个`Query`删除文档。如果没找到，返回false
- **`Find`** - 使用LiteDB查询查找文档。参考 [Query](Query)
- **`EnsureIndex`** - 在一个字段上创建新索引。参考 [Indexes](Indexes)
- **`DropIndex`** - 删除一个存在的索引
