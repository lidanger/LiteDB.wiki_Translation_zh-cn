LiteDB像RDBMS数据库一样实现多文档ACID事务。

```C#
using(var db = new LiteDatabase("mydata.db"))
{
    var col = db.GetCollection<Customer>("customer");

    using(var trans = db.BeginTrans())
    {
        col.Insert(new Customer { Name = "John Doe" });
        col.Insert(new Customer { Name = "Joanna Doe" });
        col.Insert(new Customer { Name = "Foo Bar" });

        trans.Commit();
    } // 所有或一个也没有!
}
```

### 事务原理

LiteDB使用`CacheService`来跟踪从磁盘上读取的页面。当一个页面发生变化时，缓存将这个变化保持在内存中，直到一个`Commit()`被调用。提交之前，数据文件不会发生变化，一切只在内存中发生。如果一个`Rollback()`被调用，缓存服务将丢弃所有变化的页面。

LiteDB运行时需要事务。如果是省略事务的写操作，像`Insert()`, `Update()` 和 `Delete()`，LiteDB将为每一个操作创建一个自动事务。相对于在开始使用`BeginTrans()`结束时使用`Commit()`来说，这是一个慢速方案，因为自动事务在每次文档变化后都会写入硬盘。

### 并发

LiteDB v3支持线程安全，因此你可以在所有线程间共享你的数据库实例对象。对LiteDB来说，这是最好的用法。因为所有缓存的页面由此单实例维护。当你调用`Dispose`时，数据文件会被关闭，所有内存缓存会被清除。LiteDB v3只支持单文件连接-而不是多文件并发。

### 嵌套事务

LiteDB支持嵌套事务。你可以在主事务中含有嵌套的`BeginTrans()` 和 `Commit()`。所有嵌套事务只是被当作一个简单的栈计数器。只有主事务`Commit()`会将所有更改写到磁盘上。

但是，如果在任何事务（主事务或嵌套）出现一个`Rollback()`，就会执行回滚，并清空事务栈。

在写操作过程中，如果出现任何错误，LiteDB会抛出异常并回滚事务，比如插入重复键。

```C#
using(var db = new LiteDatabase("mydata.db"))
{
    var col = db.GetCollection("customer");

    using(var trans1 = db.BeginTrans())
    {
        col.Insert(new Customer { Id = 1, Name = "John" });

        using(var trans2 = db.BeginTrans()) // Stack +1
        {
            col.Insert(new Customer { Id = 2, Name = "Joana" });

            trans2.Commit(); // Stack -1
        }

        col.Insert(new Customer { Id = 1, Name = "Foo Bar" }); // 造成索引重复键

        // 一个异常会被抛出，一个回滚会反转所有数据，没有数据会被保存！
        trans1.Commit();
    }
}
```
