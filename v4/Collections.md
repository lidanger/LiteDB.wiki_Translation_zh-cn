文档在集合中存储和组织。在 LiteDB 中，`LiteCollection` 是管理集合的泛型类。每个集合必须有唯一的名称：

- 只能包含字母、数字和 `_`
- **不分大小写**
- 以 `_` 开头的集合名称系统内部保留使用

一个数据库中所有集合名称的总大小上限为 3000 字节。如果想在数据库中创建很多集合，请为你的集合使用短名称。比如，假设集合名称有大约 10 个字节长，那么你可以在数据库中创建多达 300 个集合。

集合在第一个 `Insert` 或 `EnsureIndex` 操作时自动创建，在一个不存在的集合上执行查询、删除或更新文档等操作，不会创建集合。

`LiteCollection<T>` 是一个泛型类，对于无模式文档来说，它可以与作为 `BsonDocument` 的 `<T>` 一起使用。在系统内部，LiteDB 会转换 `T` 为 `BsonDocument`，并且，所有操作都可以使用这个泛型文档。

在这个例子中，两个代码片段产生同样的结果。

```C#
// 强类型类
using(var db = new LiteDatabase("mydb.db"))
{
    // 获取集合实例
    var col = db.GetCollection<Customer>("customer");
    
    // 在集合中插入文档 - 如果集合不存在，现在创建
    col.Insert(new Customer { Id = 1, Name = "John Doe" });
    
    // 如果 Name 字段上不存在索引，在 Name 字段上创建索引
    col.EnsureIndex(x => x.Name);
    
    // 现在，搜索你的文档
    var customer = col.FindOne(x => x.Name == "john doe");
}

// 通用 BsonDocument
using(var db = new LiteDatabase("mydb.db"))
{
    // 获得集合实例
    var col = db.GetCollection("customer");
    
    // 在集合中插入文档 - 如果集合不存在，创建
    col.Insert(new BsonDocument().Add("_id", 1).Add("Name", "John Doe"));
    
    // 如果 Name 字段上不存在索引，在 Name 字段上创建索引
    col.EnsureIndex("Name");
    
    // 现在，搜索你的文档
    var customer = col.FindOne(Query.EQ("Name", "john doe"));
}
```
# LiteDatabase API 实例方法

- **`GetCollection<T>`** - 此方法返回 `LiteCollection` 的新实例。如果忽略 `<T>`，`<T>` 就是 `BsonDocument`。这是获取集合实例的唯一方法。
- **`RenameCollection`** - 只是重命名集合名称 - 不改变任何文档
- **`CollectionExists`** - 检查集合是否已存在于数据库中
- **`GetCollectionNames`** - 获取数据库中所有的集合名称
- **`DropCollection`** - 删除集合中所有的文档、索引以及集合引用

### LiteCollection API 实例方法

- **`Insert`** - 插入一个新文档或者文档 `IEnumberable` 列表。如果你的文档无 `_id` 字段，Insert 将使用 `ObjectId` 创建一个。如果你映射了一个对象，可以使用 `AutoId`。参见[对象映射](Object-Mapping)
- **`InsertBulk`** - 用于插入大量文档。可以将文档分成几批，然后在每批中控制事务。此方法在每批插入后会清理缓存，以保证较低的内存占用。
- **`Update`** - 更新一个用 `_id` 字段标识的文档。如果未发现，返回 false
- **`Delete`** - 按 `_id` 或 `Query` 查找结果删除文档。如果未发现，返回 false
- **`Find`** - 使用 LiteDB 查询查找文档。参见[查询](Queries)
- **`EnsureIndex`** - 在字段上创建一个新的索引。参见[索引](Indexes)
- **`DropIndex`** - 删除一个已存在的索引
