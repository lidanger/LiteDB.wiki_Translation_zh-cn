# LiteDB v4 发布了

## 新特性
- 增加对 NETStandard 2.0 的支持 (支持 `Shared` 模式)
- 新的文档 `表达式` parser/executor - 参考[表达式](https://github.com/mbdavid/LiteDB/wiki/Expressions)
- 支持表达式索引创建
```C#
col.EnsureIndex(x => x.Name, "LOWER($.Name)");
col.EnsureIndex("GrandTotal", "SUM($.Items[*].Qtd * $.Items[*].Price)");
```
- 在引擎级别上支持使用 `Include` 查询，支持任意嵌套的 includes
```C#
col.Include(x => x.Users)
   .Include(x => x.Users[0].Address)
   .Include(x => x.Users[0].Address.City)
   .Find(...)
```
- 支持复杂的 Linq 查询，使用 `LinqQuery` 编译器 (像 linq to object 一样工作)
  - `col.Find(x => x.Name == "John" && x.Items.Length.ToString().EndsWith == "0")`
- 在多个查询语句中有更好的执行计划 (包含调试信息) 
- 不再有外部日志文件 - 使用相同的数据文件来存储临时数据
- 修复并发问题 (保证线程/进程安全)
- 有可能的话转换 `Query.And` 为 `Query.Between`
- 添加对 `Query.Between` 打开/关闭间隔的支持
- **与 LiteDB `v3` 一样的数据文件 (不需要升级)**

## Shell
- 新的 UPDATE/SELECT 语句
- Shell 命令 parser/executor 重新回到 LiteDB.dll
- 更好的 shell 解析错误信息，在错误中包含位置
- 在调试模式下打印查询执行计划
`(Seek([Age] > 10) and Filter([Name] startsWith "John"))`
(准备开发新的可视化 LiteDB 数据库管理工具)

## 突破性变化
- 移除事务
- 移除自定义类型的 auto-id 注册函数
- 移除映射时的索引定义 (fluent/特性)
- 移除查询执行过程中自动创建索引。如果未发现索引，执行全扫描搜索 (在初始化数据库时使用 `EnsureIndex`)
