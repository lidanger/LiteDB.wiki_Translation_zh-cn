为了使内存占用不那么高，LiteDB 将文档大小上限限制为 1 Mb。对于文本文档来说，这已经超大了。但对于很多二进制文件来说，1 Mb 还是太小了。因此，LiteDB 实现了 `FileStorage`，用于存储文件和流的定制集合。

LiteDB 使用两个特殊的集合将文件内容分裂成块：

- `_files` 集合只存储文件引用和元数据

```JS
{
    _id: "my-photo",
    filename: "my-photo.jpg",
    mimeType: "image/jpg",
    length: { $numberLong: "2340000" },
    uploadDate: { $date: "2015-01-01T00:00:00Z" },
    metadata: { "key1": "value1", "key2": "value2" }
}
```

- `_chunks` 集合按 1 MB 每块存储二进制数据。

```JS
{
    _id: "my-photo\00001",
    data: { $binary: "VHlwZSAob3Igc ... GUpIGhlcmUuLi4" }
}
{
    _id: "my-photo\00002",
    data: { $binary: "pGaGhlcmUuLi4 ... VHlwZSAob3Igc" }
}
{
   ...
}
```

文件用一个 `_id` 字符串值标识，规则如下：

- 以字母、数字、`_`, `-`, `$`, `@`, `!`, `+`, `%`, `;` 或 `.` 开头
- 如果包含 `/`，必须跟在以上字符后面

要想更好地组织很多文件，你可以将 `_id` 设定为 `directory/file_id`。要想使用 `Find` 方法快速查找同一个文件夹的所有文件，这是个相当不错的方案。

例如: `$/photos/2014/picture-01.jpg`

`FileStorage` 集合包含了一些简单的方法，如下：

- **`Upload`**: 发送文件或流到数据库。可以用于文件或 `Stream`。如果文件已存在，那么文件内容将被覆盖。
- **`Download`**: 从数据库获取你的文件，并复制到 `Stream` 参数
- **`Delete`**: 删除一个文件引用以及它的所有数据块
- **`Find`**: 在 `_files` 集合中查找一个或多个文件。返回的 `LiteFileInfo` 对象，可以用于稍后下载数据。
- **`SetMetadata`**: 更新已存储文件的元数据。此方法不改变已存储文件的内容，它只更新 `_files.metadata` 的值。
- **`OpenRead`**: 通过 `_id` 查找文件，并返回一个 `LiteFileStream` 流来读取文件内容

```C#
// 从文件系统上传一个文件
db.FileStorage.Upload("$/photos/2014/picture-01.jpg", @"C:\Temp\picture-01.jpg");

// 从流上传一个文件
db.FileStorage.Upload("$/photos/2014/picture-01.jpg", stream);

// 查找文件引用 - 如果未发现返回 null
LiteFileInfo file = db.FileStorage.FindById("$/photos/2014/picture-01.jpg");

// 现在，加载二进制数据并保存到文件系统
file.SaveAs(@"C:\Temp\new-picture.jpg");

// 或者将二进制数据获取为流并复制到另一个流中
file.CopyTo(Response.OutputStream);

// 查找同一个 "文件夹" 中的所有文件 
var files = db.FileStorage.Find("$/photos/2014/");
```

`FileStorage` 不支持事务，以避免，在将整个文件存储到磁盘前，文件的所有部分都被加载到内存中。块*是*使用事务的，每个上传的块在单独的事务中提交。

当上传每个块时，`FileStorage` 将 `_files.length` 保存在 `0` 的位置。当所有的块上传完毕后，`FileStorage` 会对每个块的大小进行合计，并更新 `_files.length`。如果你下载的文件长度为零字节，那么上传的文件已经损坏。但一个损坏的文件并不一定就意味着一个损坏的数据库。 
