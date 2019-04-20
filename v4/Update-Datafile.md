> 从 v3 到 v4 LiteDB **数据文件并没有变化**。你可以使用相同的数据文件!

要从 LiteDB v2 升级你的数据文件，你可以在连接字符串中使用 "upgrade=true"。像这样：

```
var db = new LiteDatabase("filename=old.db;upgrade=true");
```

LiteDB 将更新 (如果有必要) 你的数据文件到新的数据文件格式。也可以使用 shell：

```
> upgrade my-old-file.db

如果你的旧数据文件有密码保护，使用：

> upgrade filename=my-old-file.db;password=mypass
```

> 只支持 v2 版本的数据文件 (文件格式 6)。
