要从LiteDB v2升级你的数据文件，你可以在连接字符串中使用"upgrade=true"。像这样：

```
var db = new LiteDatabase("filename=old.db;upgrade=true");
```

LiteDB将更新（如果必要）到新的数据文件格式。你也可以使用Shell：

```
> upgrade my-old-file.db

如果你的旧数据文件是密码保护的，使用：

> upgrade filename=my-old-file.db;password=mypass
```

> 只支持v2数据文件(文件格式 6)。从v1(文件格式4)升级仍然在开发中