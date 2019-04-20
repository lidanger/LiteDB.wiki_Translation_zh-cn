`LiteRepository` 是一个访问数据库的新类。它通过 `LiteDatabase` 实现，实际上只是一个不用 `LiteCollection` 类和 fluent query 而快速访问数据的封装层。

```C#
using(var db = new LiteRepository(connectionString))
{
    // 简单访问，Insert/Update/Upsert/Delete
    db.Insert(new Product { ProductName = "Table", Price = 100 });

    db.Delete<Product>(x => x.Price == 100);

    // 使用 fluent query 查询
    var result = db.Query<Order>()
        .Include(x => x.Customer) // add dbref 1x1
        .Include(x => x.Products) // add dbref 1xN
        .Where(x => x.Date == DateTime.Today) // 使用索引的查询
        .Where(x => x.Active) // used indexes query
        .ToList();

    var p = db.Query<Product>()
        .Where(x => x.ProductName.StartsWith("Table"))
        .Where(x => x.Price < 200)
        .Limit(10)
        .ToEnumerable();

    var c = db.Query<Customer>()
        .Where(txtName.Text != null, x => x.Name == txtName.Text) // conditional filter
        .ToList();

}
```

集合名称可以省略，通过新的 `BsonMapper.ResolveCollectionName` 函数解析获得 (默认：`typeof(T).Name`)。

这个 API 受启发于这个牛X项目[NPoco Micro-ORM](https://github.com/schotime/NPoco)。