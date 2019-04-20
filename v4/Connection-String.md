LiteDatabase 可以通过一个连接字符串进行初始化，语法为 `key1=value1; key2=value2; ...`。如果在你的连接字符串中没有 `;`，LiteDB 假定你的连接字符串是 Filename 键。键不分大小写。

### 选项

- **`Filename`** (string): 完整路径或从DLL文件夹开始的相对路径。
- **`Journal`** (bool): 启用或禁用双写检查来确保持久性 (默认: true)
- **`Password`** (string): 使用密码加密 (使用 AES) 你的数据文件 (默认: null - 无加密)
- **`Cache Size`** (int): 缓存中的最大页面数据。达到这个大小后，刷新数据到磁盘以防止占有太多内存 (默认: 5000)
- **`Timeout`** (TimeSpan): 等待解锁操作的超时时间 (线程锁和文件锁)
- **`Mode`** (Exclusive|ReadOnly|Shared): 数据文件的打开方式 (默认: NET35中为 `Shared`，NetStandard中为 `Exclusive`)
- **`Initial Size`** (string|long): 如果数据库是新建的，使用分配的空间初始化 - 支持 KB, MB, GB (默认: null)
- **`Limit Size`** (string|long): 数据文件的最大限制 - 支持 KB, MB, GB (默认: null)
- **`Upgrade`** (bool): 如果是 true，尝试从老版本 (v2) 升级数据文件 (默认: null)
- **`Log`** (byte): 数据库的调试消息 - 在 `LiteDatabase.Log` (默认: Logger.NONE)
- **`Async`** (bool): 支持 "sync over async" 文件流创建，以便于 UWP 中访问任何磁盘文件夹 (只用于NetStandard，默认: false)
- **`Flush`** (bool): 直接将数据写入到磁盘，以阻止操作系统缓存 (在 NET35 下不可用，默认: false) (v4.1.2)


### 示例

_App.config_
```XML
    <connectionStrings>
        <add name="LiteDB" connectionString="Filename=C:\database.db" />
    </connectionStrings>
```

_C#_
```C#
    System.Configuration.ConfigurationManager.ConnectionStrings["LiteDB"].ConnectionString
```