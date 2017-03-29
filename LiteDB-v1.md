# LiteDB - A .NET NoSQL Document Store

欢迎来到LiteDB文档Wiki页面。在这里你会找到所有你需要的信息，以更好地了解LiteDB以及你可以怎样使用。

> Documentation for **v.1.0.x**

## 准备开始

LiteDB是一个简单，快速和轻量级的嵌入式.NET文档数据库。LiteDB受到MongoDB启发，它的API非常类似于MongoDB的官方.NET API。

### 如何安装

LiteDB是一个无服务器数据库，因此不需要安装。只需要复制 [LiteDB.dll](https://github.com/mbdavid/LiteDB/releases) 到你的 Bin 文件夹并添加为引用即可。或者如果你喜欢，你可以通过 NuGet: `Install-Package LiteDB`安装。如果你要运行在一个web环境下，确认你的IIS用户对数据文件夹有写权限。

### First 例子

A quick example to store and search for documents:

```C#
// Create your POCO class entity
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string[] Phones { get; set; }
    public bool IsActive { get; set; }
}

// Open database (or create if doesn't exist)
using(var db = new LiteDatabase(@"C:\Temp\MyData.db"))
{
	// Get a collection (or create, if doesn't exist)
	var col = db.GetCollection<Customer>("customers");

    // Create your new customer instance
	var customer = new Customer
    { 
        Name = "John Doe", 
        Phones = new string[] { "8000-0000", "9000-0000" }, 
        IsActive = true
    };
	
	// Insert new customer document (Id will be auto-incremented)
	col.Insert(customer);
	
	// Update a document inside a collection
	customer.Name = "Joana Doe";
	
	col.Update(customer);
	
	// Index document using document Name property
	col.EnsureIndex(x => x.Name);
	
	// Use LINQ to query documents
	var results = col.Find(x => x.Name.StartsWith("Jo"));
}
```

### Working with files

Need to store files? No problem: use FileStorage.

```C#
// Upload a file from file system to database
db.FileStorage.Upload("my-photo-id", @"C:\Temp\picture-01.jpg");

// And download later
db.FileStorage.Download("my-photo-id", @"C:\Temp\copy-of-picture-01.jpg");
```

## 数据结构

LiteDB stores data as documents, which are JSON-like field and value pairs. Documents are a schema-less data structure. Each document has your data and structure together.

```JS
{
    _id: 1,
    name: { first: "John", last: "Doe" },
    age: 37,
    salary: 3456.0,
    createdDate: { $date: "2014-10-30T00:00:00.00Z" },
    phones: ["8000-0000", "9000-0000"]
}
```

- `_id` contains document primary key - an unique value in collection
- `name` containts an embedded document with `first` and `last` fields
- `age` contains a `Int32` value
- `salary` contains a `Double` value
- `createDate` contains a `DateTime` value
- `phones` contains an array of `String`

LiteDB stores documents in collections. A collection is a group of related documents that have a set of shared indices. Collections are analogous to a table in relational databases.

### BSON

LiteDB stores documents in the BSON (Binary JSON) data format. BSON is a binary representation of JSON with additional type information. In the documents, the value of a field can be any of the BSON data types, including other documents, arrays, and arrays of documents. BSON is a fast and simple way to serialize documents in binary format.

LiteDB uses only a subset of [BSON data types](http://bsonspec.org/spec.html). See all supported LiteDB BSON data types and .NET equivalents.

|BSON Type |.NET type                                                   |
|----------|------------------------------------------------------------|
|MinValue  |-                                                           |
|Null      |Any .NET object with `null` value                           |
|Int32     |`System.Int32`                                              |
|Int64     |`System.Int64`                                              |
|Double    |`System.Double`                                             |
|String    |`System.String`                                             |
|Document  |`System.Collection.Generic.Dictionary<string, BsonValue>`   |
|Array     |`System.Collection.Generic.List<BsonValue>`                 |
|Binary    |`System.Byte[]`                                             |
|ObjectId  |`LiteDB.ObjectId`                                           |
|Guid      |`System.Guid`                                               |
|Boolean   |`System.Boolean`                                            |
|DateTime  |`System.DateTime`                                           |
|MaxValue  |-                                                           |

### JSON 扩展

To serialize a document to JSON, LiteDB uses an extended version of JSON so as not to lose any BSON type information that does not exist in JSON. Extended data type is represented as an embedded document, using an initial `$` key and value as string.

|BSON data type|JSON representation                                   |Description                        |
|--------------|------------------------------------------------------|-----------------------------------|
|ObjectId      |`{ "$oid": "507f1f55bcf96cd799438110" }`              |12 bytes in hex format             |
|Date          |`{ "$date": "2015-01-01T00:00:00Z" }`                 |UTC and ISO-8601 format            |
|Guid          |`{ "$guid": "ebe8f677-9f27-4303-8699-5081651beb11" }` |                                   |
|Binary        |`{ "$binary": "VHlwZSgaFc3sdcGFzUpcmUuLi4=" }`        |Byte array in base64 string format |
|Int64         |`{ "$numberLong": "12200000" }`                       |                                   |
|MinValue      |`{ "$minValue": "1" }`                                |                                   |
|MaxValue      |`{ "$maxValue": "1" }`                                |                                   |

LiteDB implements JSON in its `JsonSerializer` static class. Serialize and deserialize only accepts `BsonValue`s as input/output. If you want convert your object type to a BsonValue, you need use a `BsonMapper`.

```C#
var customer = new Customer { Id = 1, Name = "John Doe" };

var doc = BsonMapper.Global.ToDocument(customer);

var jsonString = JsonSerialize.Serialize(doc, pretty, includeBinaryData);
```

`JsonSerialize` also supports `TextReader` and `TextWriter` to read/write directly from a file or `Stream`.

### ObjectId

`ObjectId` is a 12 bytes BSON type:

- `Timestamp`: Value representing the seconds since the Unix epoch (4 bytes)
- `Machine`: Machine identifier (3 bytes)  
- `Pid`: Process id (2 bytes)
- `Increment`: A counter, starting with a random value (3 bytes)

In LiteDB, documents are stored in a collection that requires a unique `_id` field that acts as a primary key. Because `ObjectIds` are small, most likely unique, and fast to generate, LiteDB uses `ObjectIds` as the default value for the `_id` field if the `_id` field is not specified.

Unlike the Guid data type, ObjectIds are sequential, so it's a better solution for indexing. ObjectIds use hexadecimal numbers represented as strings.

```C#

var id = ObjectId.NewObjectId();

// You can get creation datetime from an ObjectId
var date = id.CreationTime;

// ObjectId is represented in hex value
Debug.Print(id);
"507h096e210a18719ea877a2"

// Create an instance based on hex representation
var nid = new ObjectId("507h096e210a18719ea877a2");
```

## BsonDocument

The `BsonDocument` class is LiteDB's implementation of documents. Internally, a `BsonDocument` stores key-value pairs in a `Dictionary<string, BsonValue>`.

```C#
var customer = new BsonDocument();
customer["_id"] = ObjectId.NewObjectId();
customer["Name"] = "John Doe";
customer["CreateDate"] = DateTime.Now;
customer["Phones"] = new BsonArray { "8000-0000", "9000-000" };
customer["IsActive"] = true;
customer["IsAdmin"] = new BsonValue(true);
customer.Set("Address.Street", "Av. Protasio Alves, 1331");
```

About document field **keys**:

- Keys must contains only letters, numbers or `_` and `-`
- Keys are case-sensitive
- Duplicate keys are not allowed
- LiteDB keeps the original key order, including mapped classes. The only exception is for `_id` field that will always be the first field. 

About document field **values**:

- Values can be any BSON value data type: Null, Int32, Int64, Double, String, Embedded Document, Array, Binary, ObjectId, Guid, Boolean, DateTime, MinValue, MaxValue
- When a field is indexed, the value must be less then 512 bytes after BSON serialization.
- Non-indexed values themselves have no size limit, but the whole document is limited to 1Mb after BSON serialization. This size includes all extra bytes that are used by BSON.
- `_id` field cannot be: `Null`, `MinValue` or `MaxValue`
- `_id` is unique indexed field, so value must be less then 512 bytes

About .NET classes

- `BsonValue` 
    - This class can hold any BSON data type, including null, array or document.
    - Has implicit constructor to all supported .NET data types
    - Value never changes
    - `RawValue` property that returns internal .NET object instance
- `BsonArray` 
    - Supports `IEnumerable<BsonValue>`
    - Each array item can have different BSON type objects
- `BsonDocument`
    - Missing fields always return `BsonValue.Null` value
    - `Set` and `Get` methods can be used with dotted notation to access/create inner documents

```C#
// Testing BSON value data type
if(customer["Name"].IsString) { ... }

// Helper to get .NET type
string str = customer["Name"].AsString;
```

To use other .NET data types you need a custom `BsonMapper` class.

## 对象映射

LiteDB supports POCO classes to strongly type documents. When you get a `LiteCollection` instance from `LiteDatabase.GetCollection<T>`, `<T>` will be your document type. If `<T>` is not a `BsonDocument`, LiteDB internally maps your class to `BsonDocument`. To do this, LiteDB uses the `BsonMapper` class:

```C#
// Simple strongly-typed document
public class Customer
{
    public ObjectId CustomerId { get; set; }
    public string Name { get; set; }
    public DateTime CreateDate { get; set; }
    public List<Phone> Phones { get; set; }
    public bool IsActive { get; set; }
}

var typedCustomerCollection = db.GetCollection<Customer>("customer");

var schemelessCollection = db.GetCollection("customer"); // <T> is BsonDocument
```

### 映射器约定

`BsonMapper.ToDocument()` auto converts each property of a class to a document field following these conventions:

- Classes must be public with a public parameterless constructor
- Properties must be public
- Properties can be read-only or read/write
- The class must have an `Id` property, `<ClassName>Id` property or any property with `[BsonId]` attribute
- A property can be decorated with `[BsonIgnore]` to not be mapped to a document field
- A property can be decorated with `[BsonField]` to customize the name of the document field
- No circular references are allowed
- Max depth of 20 inner classes
- Class fields are not converted to document
- `BsonMapper` uses a global instance that caches mapping information for better performance. This instance is available at `LiteDatabase.Mapper` as well.

In addition to basic BSON types, `BsonMapper` maps others .NET types to BSON data type:

|.NET type                 |BSON type     |
|--------------------------|--------------|
|`Int16`, `UInt16`, `Byte` |Int32         |
|`UInt32` , `UInt64`       |Int64         |
|`Single`, `Decimal`       |Double        |
|`Char`, `Enum`            |String        |
|`IList<T>`                |Array         |
|`T[]`                     |Array         |
|`NameValueCollection`     |Document      |
|`IDictionary<K,T>`        |Document      |
|Any other .NET type       |Document      |

- `Nullable<T>` are accepted. If value is `null` the BSON type is Null, otherwise the mapper will use `T?`.
- For `IDictionary<K, T>`, `K` key must be `String` or simple type (convertible using `Convert.ToString(..)`). 

#### 注册一个自定义类型(Needs updating as BsonMapper.Global is not available in LiteDB 2.0)

You can register your own map function, using the `RegisterType<T>` instance method. To register, you need to provide both serialize and deserialize functions.

```C#
BsonMapper.Global.RegisterType<Uri>
(
    serialize: (uri) => uri.AbsoluteUri,
    deserialize: (bson) => new Uri(bson.AsString)
);
```

- `serialize` functions pass a `<T>` object instance as the input parameter and expect return a `BsonValue`
- `deserialize` function pass a `BsonValue` object as the input parameter and except return a `<T>` value
- `RegisterType` supports complex objects via `BsonDocument` or `BsonArray` 

#### 映射选项

`BsonMapper` class settings:

|Name                   |Default |Description                                                |
|-----------------------|--------|-----------------------------------------------------------|
|`SerializeNullValues`  |false   |Serialize field if value is `null`                         |
|`TrimWhitespace`       |true    |Trim strings properties before mapping to document         |
|`EmptyStringToNull`    |true    |Empty strings convert to `null`                             |
|`ResolvePropertyName`  |(s) => s|A function to map property name to document field name     |

`BsonMapper` offers 2 predefined functions to resolve property name: `UseCamelCase()` and `UseLowerCaseDelimiter('_')`.

```C#
BsonMapper.Global.UseLowerCaseDelimiter('_');

public class Customer
{
    public int CustomerId { get; set; }

    public string FirstName { get; set; }

    [BsonField("customerLastName")]
    public string LastName { get; set; }
}

var doc = BsonMapper.Global.ToDocument(new Customer { FirstName = "John", LastName = "Doe" });

var id = doc["_id"].AsInt;
var john = doc["first_name"].AsString;
var doe = doc["customerLastName"].AsString;
```    

### AutoId

By default, typed documents will receive an auto **Id** value on insert. LiteDB support auto-id for this data types: 

|.NET data type  |New Value                                     |
|----------------|----------------------------------------------|
|`ObjectId`      |`ObjectId.NewObjectId()`                      |
|`Guid`          |`Guid.NewGuid()`                              |
|`Int32`         |Auto-increment, per collection, starting in 1 |

But if you don't want to use any of the built-in methods, or want to use another data type for **Id**, you can set it yourself:

```C#
var customer = new Customer { Id = "john-doe", Name = "John Doe" };
```

You also can create your own auto-id function, using `RegisterAutoId<T>`:

```C#
BsonMapper.Global.RegisterAutoId<long>
(
    isEmpty: (value) => value == 0,
    newId: (collection) => DateTime.UtcNow.Ticks
);

public class Calendar
{
    public long CalendarId { get; set; }
}
```

- `isEmpty` returns true to indicate when this type is considered empty. In this example, zero will be an empty value.
- `newId` returns a new id value. It takes a `LiteCollection<BsonDocument>` instance as an input parameter that you may use to determine the new Id. In this example, the collection is ignored and the current Ticks value is returned.

### 索引定义
    
`BsonMapper` supports index definition directly on a property using the `[BsonIndex]` attribute. You can define index options like `Unique` or `IgnoreCase`. This allows you to avoid always needing to call `col.EnsureIndex("field")` before running a query.

```C#
public class Customer
{
    public int Id { get; set; }
    
    [BsonIndex]
    public string Name { get; set; }
    
    [BsonIndex(new IndexOptions { Unique = true, EmptyStringToNull = false, RemoveAccents = false })]
    public string Email { get; set; }
}
```

`IndexOptions` class settings:

|Name                 |Default|Description                                       |
|---------------------|-------|--------------------------------------------------|
|`Unique`             |false  |Do not allow duplicates values in index           |
|`IgnoreCase`         |true   |Store string on index in lowercase                |
|`RemoveAccents`      |true   |Store string on index removing accents            |
|`TrimWhitespace`     |true   |Store string on index removing trimming spaces    |
|`EmptyStringToNull`  |true   |If string is empty, convert to `Null`             |

To get better performance, `[BsonIndex]` checks only if the index exists, but does not check if you are changing options. To change an index option in a existing index you must run `EnsureIndex` with the new index options. This method drops the current index and creates a new one with the new options.

Do not use `[BsonIndex]` attribute on your **Id** primary key property. This property already has an unique index with the default options.

## 集合

Documents are stored and organized in collections. `LiteCollection` is a generic class to manage collections in LiteDB. Each collection must have a unique name:

- Contains only letters, numbers and `_`
- Collection names are **case insensitive**
- Collection names starting with `_` are reserved for internal use

LiteDB supports up to 256 collections per database. 

Collections are auto created on first `Insert` or `EnsureIndex` operation. Running a query, delete or update on a document in a non existing collection does not create one.

`LiteCollection<T>` is a generic class that can be used with `<T>` as `BsonDocument` for schema-less documents.  Internally LiteDB converts `T` to `BsonDocument` and all operations use the this generic document.

In this example, both code snippets produce the same results.

```C#
// Strong Type Class
using(var db = new LiteDatabase("mydb.db"))
{
    // Get collection instance
    var col = db.GetCollection<Customer>("customer");
    
    // Insert document to collection - if collection do not exits, create now
    col.Insert(new Customer { Id = 1, Name = "John Doe" });
    
    // Create, if not exists, new index on Name field
    col.EnsureIndex(x => x.Name);
    
    // Now, search for document your document
    var customer = col.FindOne(x => x.Name == "john doe");
}

// Generic BsonDocument
using(var db = new LiteDatabase("mydb.db"))
{
    // Get collection instance
    var col = db.GetCollection("customer");
    
    // Insert document to collection - if collection do not exits, create now
    col.Insert(new BsonDocument().Add("_id", 1).Add("Name", "John Doe"));
    
    // Create, if not exists, new index on Name field
    col.EnsureIndex("Name");
    
    // Now, search for document your document
    var customer = col.FindOne(Query.EQ("Name", "john doe"));
}
```
### LiteDatabase API Instance Methods

- **`GetCollection<T>`** - This method returns a new instance of `LiteCollection`. If `<T>` if ommited, `<T>` is `BsonDocument`. This is the only way to get a collection instance.
- **`RenameCollection`** - Rename a collection name only - do not change any document
- **`CollectionExists`** - Check if a collection already exits in database
- **`GetCollectionNames`** - Get all collections names in database
- **`DropCollection`** - Delete all documents, all indexes and the collection reference on database

### LiteCollection API Instance Methods

- **`Insert`** - Inserts a new document or an `IEnumberable` of documents. If your document has no `_id` field, Insert will create a new one using `ObjectId`. If you have a mapped object, `AutoId` can be used. See [Object Mapping](Object-Mapping)
- **`InsertBulk`** - Used for inserting a high volume of documents. Breaks documents into batches and controls transaction per batch. This method keeps memory usage low by cleaning a cache after each batch inserted.
- **`Update`** - Update one document identified by `_id` field. If not found, returns false
- **`Delete`** - Delete document by `_id` or by a `Query` result. If not found, returns false
- **`Find`** - Find documents using LiteDB queries. See [Query](Query)
- **`EnsureIndex`** - Create a new index in a field. See [Indexes](Indexes)
- **`DropIndex`** - Drop an existing index

### 使用 `DbRef<T>`

LiteDB is a document database, so there is no JOIN between collections. If you need reference a document in another document you can use `DbRef<T>`. This document reference can be loaded when the database is initialized or when a query is run, or after a query is finished.


### 在数据库初始化时映射一个引用
```C#
public class Customer
{
    public int CustomerId { get; set; }
    public string Name { get; set; }
}

public class Order
{
    [BsonId]
    public int OrderNumber { get; set; }
    public DateTime OrderDate { get; set; }
    public Customer[] Customers { get; set; }
}

public class DatabaseSubclass : LiteDatabase {
	protected override void OnModelCreating(BsonMapper mapper) {
		mapper.Entity<Order>()
			.DbRef(x => x.Customers, "CustomersCollection" );
	}
}
```

### 在查询或之后映射一个引用
```C#
public class Customer
{
    public int CustomerId { get; set; }
    public string Name { get; set; }
}

public class Order
{
    [BsonId]
    public int OrderNumber { get; set; }
    public DateTime OrderDate { get; set; }
    public DbRef<Customer> Customer { get; set; }
}

// Getting customer and order collections
var customers = db.GetCollection<Customer>("customers");
var orders = db.GetCollection<Order>("orders");

// Creating a new Customer instance
var customer = new Customer { CustomerId = 5, Name = "John Doe" };

customers.Insert(customer);

// Create a instance of Order and reference to Customer John
var order = new Order
{
    OrderNumber = 1,
    OrderDate = DateTime.Now,
    Customer = new DbRef<Customer>(customers, customer.CustomerId)
};

orders.Insert(order);
```

At this point you have a **Order** document like this:

```JS
{
    _id: 1,
    OrderDate: { $date: "2015-01-01T00:00:00Z" },
    Customer: { $ref: "customers", $id: 5 }
}
```

To query **Order** and returns Customer information use `Include` method

```C#
// Include Customer document during query process
var order = orders
    .Include((c) => c.Customer.Fetch(db))
    .Find(o => o.OrderDate == DateTime.Now);
    
var john = order.Customer.Item.CustomerName;

// Or include later...

var john = order.Customer.Fetch(db).CustomerName
```

## 文件存储

To keep its memory profile slim, LiteDB has a limited document size of 1Mb. For text documents, this is a huge size. But for many binary files, 1Mb is too small. LiteDB therefore implements `FileStorage`, a custom collection to store files and streams.

LiteDB uses two special collections to split file content in chunks:

- `_files` collection stores file reference and metadata only

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

- `_chunks` collection stores binary data in 1MB chunks.

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

Files are identified by an `_id` string value, with following rules:

- Starts with a letter, number, `_`, `-`, `$`, `@`, `!`, `+`, `%`, `;` or `.`
- If contains a `/`, must be sequence with chars above 

To better organize many files, you can use `_id` as a `directory/file_id`. This will be a great solution to quickly find all files in a directory using the `Find` method.

Example: `$/photos/2014/picture-01.jpg`

The `FileStorage` collection contains simple methods like:

- **`Upload`**: Send file or stream to database. Can be used with file or `Stream`. If file already exits, file content is overwritten.
- **`Download`**: Get your file from database and copy to `Stream` parameter
- **`Delete`**: Delete a file reference and all data chunks
- **`Find`**: Find one or many files in `_files` collection. Returns `LiteFileInfo` class, that can be download data after.
- **`SetMetadata`**: Update stored file metadata. This method doesn't change the value of the stored file.  It updates the value of `_files.metadata`.
- **`OpenRead`**: Find file by `_id` and returns a `LiteFileStream` to read file content as stream

```C#
// Upload a file from file system
db.FileStorage.Upload("$/photos/2014/picture-01.jpg", @"C:\Temp\picture-01.jpg");

// Upload a file from a Stream
db.FileStorage.Upload("$/photos/2014/picture-01.jpg", strem);

// Find file reference only - returns null if not found
LiteFileInfo file = db.FileStoage.FindById("$/photos/2014/picture-01.jpg");

// Now, load binary data and save to file system
file.SaveAs(@"C:\Temp\new-picture.jpg");

// Or get binary data as Stream and copy to another Stream
file.CopyTo(Response.OutputStream);

// Find all files references in a "directory"
var files = db.FileStoage.Find("$/photos/2014/");
```

`FileStorage` does not support transactions to avoid putting all of the file in memory before storing it on disk. Transactions *are* used per chunk. Each uploaded chunk is committed in a single transaction.

`FileStorage` keeps `_files.length` at `0` when uploading each chunk. When all chunks finish uploading `FileStorage` updates `_files.length` by aggregating the size of each chunk. If you try to download a zero bytes length file, the uploaded file is corrupted. A corrupted file doesn't necessarily mean a corrupt database. 


## 使用索引

LiteDB improves search performance by using indices on document fields. Each index stores the value of a specific field ordered by the value (and type) of the field. Without an index, LiteDB must execute a query using a full document scan. Full document scans are inefficient because LiteDB must deserialize all documents to test each one by one.

### 索引实现

LiteDB uses the a simple index solution: **Skip Lists**. Skip lists are double linked sorted list with up to 32 levels. Skip lists are super easy to implement (only 15 lines of code) and statistically balanced. The results are great: insert and find results has an average of O(ln n) = 1 milion of documents = 13 steps. If you want to know more about skip lists, see [this great video](https://www.youtube.com/watch?v=kBwUoWpeH_Q). 

Documents are schema-less, even if they are in the same collection. So, you can create an index on a field that can be one type in one document and another type in another document. When you have a field with different types, LiteDB compares only the types. Each type has an order:

|BSON Type            |Order|
|---------------------|-----|
|MinValue             |1    |
|Null                 |2    |
|Int32, Int64, Double |3    |
|String               |4    |
|Document             |5    |
|Array                |6    |
|Binary               |7    |
|ObjectId             |8    |
|Guid                 |9    |
|Boolean              |10   |
|DateTime             |11   |
|MaxValue             |12   |

- Numbers (Int32, Int64 or Double) have the same order. If you mix these number types in the same document field, LiteDB will convert them to Double when comparing.

### 索引选项

LiteDB has some options that you can use when you are creating a new index. These options can't be modified after the index is created. If an index needs to be altered, LiteDB drops the current index and creates a new one.

- **`Unique`** - Defines an index that has only unique values.
- **`IgnoreCase`** - Convert field value to lower case. (String only)
- **`RemoveAccents`** - Remove all accents on field value (áéíóú => aeiou) (String only)
- **`TrimWhitespace`** - Apply String.Trim() on field value. (String only)
- **`EmptyStringToNull`** - If field value are empty convert to `BsonValue.Null`. (String only)

Note: a change to an index value **does not** effect the indexed document's field value. These rules are applied only to the value stored in the index.

### EnsureIndex()

Indices are created via `LiteCollection.EnsureIndex`. This instance method ensures an index: create the index if it does not exist, re-creates if the index options are different from the existing index, or does nothing if the index is unchanged.

Another way to ensure the creation of an index is to use the `[BsonIndex]` class attribute. This attribute will be read and will run `EnsureIndex` the first time you query. For performance reasons, this method checks only if the index exists and no index options have changed. 

Indices are identified by document field name. LiteDB only supports 1 field per index, but this field can be any BSON type, even an embedded document.

```JS
{
    _id: 1,
    Address:
    {
        Street: "Av. Protasio Alves, 1331",
        City: "Porto Alegre",
        Country: "Brazil"
    }
}
```

- You can use `EnsureIndex("Address")` to create an index to all `Address` embedded document
- Or `EnsureIndex("Address.Street")` to create an index on `Street` using dotted notation
- Indices are executed as `BsonDocument` fields. If you are using a custom `ResolvePropertyName` or `[BsonField]` attribute, you must use your document field name and not the property name. See [Object Mapping](Object-Mapping).
- You can use a lambda expression to define an index field in a strongly typed collection: `EnsureIndex(x => x.Name)`

### 限制

- Index values must have less than 512 bytes (after BSON serialization)
- Max of 16 indexes per collections - including the `_id` primary key
- `[BsonIndex]` is not supported in embedded documents

## 查询

In LiteDB, queries must have an index on the search field to find documents. If there is no index on the field, LiteDB automatically creates a new index with default options. To make queries, you can use:

- `Query` static helper class
- Using LINQ expressions

### 查询实现

`Query` is an static class that creates query criteria. Each method a different criteria operation that can be used to query documents.

- **`Query.All`** - Returns all documents. Can be specified an index field to read in ascending or descending index order.
- **`Query.EQ`** - Find document are equals (==) to value.
- **`Query.LT/LTE`** - Find documents less then (<) or less then equals (<=) to value.
- **`Query.GT/GTE`** - Find documents greater then (>) or greater then equals (>=) to value.
- **`Query.Between`** - Find documents between start/end value.
- **`Query.In`** - Find documents that are equals of listed values.
- **`Query.Not`** - Find documents that are NOT equals (!=) to value.
- **`Query.StartsWith`** - Find documents that strings starts with value. Valid only for string data type.
- **`Query.Contains`** - Find documents that strings contains value. Valid only for string data type. This query do index search, only index scan (slow for many documents).
- **`Query.And`** - Apply intersection between two queries results. 
- **`Query.Or`** - Apply union between two queries results. 

```C#
var results = collection.Find(Query.EQ("Name", "John Doe"));

var results = collection.Find(Query.GTE("Age", 25));

var results = collection.Find(Query.And(
    Query.EQ("FirstName", "John"), Query.EQ("LastName", "Doe")
));

var results = collection.Find(Query.StartsWith("Name", "Jo"));
```

In all queries:

- **Field** must have an index (by using `EnsureIndex` or `[BsonIndex]` attribute. See [[Indexes]]). If there is no index, LiteDB will create a new index on the fly.
- **Field** name on left side, **Value** (or values) on right side
- Queries are executed in `BsonDocument` class before mapping to your object. You need to use the `BsonDocument` field name and BSON types values. If you are using a custom `ResolvePropertyName` or `[BsonField]` attribute, you must use your document field name and not the property name on your type. See [Object Mapping](Object-Mapping).
- LiteDB does not support values as expressions, like `CreationDate == DueDate`.

### Find(), FindById(), FindOne() 和 FindAll()

Collections are 4 ways to return documents:

- **`FindAll`**: Returns all documents on collection
- **`FindOne`**: Returns `FirstOrDefault` result of `Find()`
- **`FindById`**: Returns `SingleOrDefault` result of `Find()` by using primary key `_id` index.
- **`Find`**: Return documents using `Query` builder or LINQ expression on collection.

`Find()` supports `Skip` and `Limit` parameters. These operations are used at the index level, so it's more efficient than in LINQ to Objects.

`Find()` method returns an `IEnumerable` of documents. If you want do more complex filters, value as expressions, sorting or transforms results you can use LINQ to Objects.

```C#
collection.EnsureIndex(x => x.Name);

var result = collection
    .Find(Query.EQ("Name", "John Doe")) // This filter is executed in LiteDB using index
    .Where(x => x.CreationDate >= x.DueDate.AddDays(-5)) // This filter is executed by LINQ to Object
    .OrderBy(x => x.Age)
    .Select(x => new 
    { 
        FullName = x.FirstName + " " + x.LastName, 
        DueDays = x.DueDate - x.CreationDate 
    }); // Transform
```

### Count() 和 Exists()

These two methods are useful because you can count documents (or check if a document exists) without deserializing the document. These methods use `In`

```C#
// This way is more efficient
var count = collection.Count(Query.EQ("Name", "John Doe"));

// Than use Find + Count
var count = collection.Find(Query.EQ("Name", "John Doe")).Count();
```

- In the first count, LiteDB uses the index to search and count the number of index occurrences of "Name = John" without deserializing and mapping the document.
- If the `Name` field does not have an index, LiteDB will deserialize the document but will not run the mapper. Still faster than `Find().Count()`
- The same idea applies when using `Exists()`, which is again better than using `Count() >= 1`. Count needs to visit all matched results and `Exists()` stops on first match (similar to LINQ's `Any` extension method).

### Min() 和 Max()

LiteDB uses a skip list implementation for indices (See [Indexes](Indexes)). Collections offer `Min` and `Max` index values. The implementation is:

- **`Min`** - Read head index node (MinValue BSON data type) and move to next node. This node is the lowest value in index. If index are empty, returns MinValue. Lowest value is not the first value!
- **`Max`** - Read tail index node (MaxValue BSON data type) and move to previous node. This node is the highest value on index. If index are empty, returns MaxValue. Highest value is not the last value!

This `Max` operation are used on `AutoId` for `Int32` types. It's fast to get this value because it needs to read only two index nodes (tail + previous).

### LINQ 表达式

Some LiteDB methods support predicates to allow you to easily query strongly typed documents.  If you are working with `BsonDocument`, you need to use classic `Query` class methods. 

```C#
var collection = db.GetCollection<Customer>("customer");

var results = collection.Find(x => x.Name == "John Doe");

var results = collection.Find(x => x.Age > 30);

var results = collection.Find(x => x.Name.StartsWith("John") && x.Age > 30);
```

- LINQ implementations are: `==, !=, >, >=, <, <=, StartsWith, Contains, Equals, &&, ||`
- Property name support inner document field: `x => x.Name.Last == "Doe"`
- Behind the scene, LINQ expressions are converted to `Query` implementations using `QueryVisitor` class.

## 数据库如何工作

### 文件格式

LiteDB is a single file database. But databases has many different types of information, like indexes, collections, documents. To manage this, LiteDB implements database pages concepts. Page is a block of same information type and has 4096 bytes. Page is the minimum read/write operation on disk file. There are 6 page types:

- **Header Page**: Contains database information, like file version, data file size and pointer to free list pages. Is the first page on database (`PageID` = 0).
- **Collection Page**: Each collection use one page and hold all collection information, like name, indexes, pointers and options. All collections are connected each others by a double linked list.
- **Index Page**: Used to hold index nodes. LiteDB implement skip list indexes, so each node has one key and levels link pointers to others index nodes.
- **Data Page**: Data page contains data blocks. Each data block represent an document serialized in BSON format. If a document is bigger than one page, data block use a link pointer to an extended page.
- **Extend Page**: Big documents that need more than one page, are serialized in multiples extended pages. Extended pages are double linked to create a single data segment that to store documents. Each extend page contains only one document or a chunk of a document.
- **Empty Page**: When a page is excluded becomes a empty page. Empty pages will be use on next page request (for any king of page).

Each page has a own header and content. Header is used to manage common data structure like PageID, PageType, Free Space. Content are implement different on each page type.

#### 页面未使用空间

Index pages and data pages contains a collection of elements (index nodes and data blocks). This pages can store data and keep with available space to more. To hold this free space on each page type, LiteDB implements free list pages.

Free list are a double linked list, sorted by available space. When database need a page to store data use this list to search first available page. After this, `PagerService` fix page order or remove from free list if there is no more space on page.

To create near data related, each collection contains an data free list. So, in a data page, all data blocks are of same collection. The same occurs in indexes. Each index (on each collection) contains your own free list. This solution consume more disk space but are much faster to read/write operations because data are related and near one witch other. If you get all documents in a collection, database needs read less pages on disk.

You can use `dump` command on shell to see all pages and basic information of each page. I used this to debug linked lists.

### 连接字符串

When you open a database file, you can just use file name as connection string or use more connection parameters like:

- Filename: Database to open or create (if no exits). Required, `String`
- Journal: If this database connection will be use journal mode (See [Journaling and Recovery](Journaling-and-Recovery)). Optional, `Boolean`, default: true
- Timeout: Set timeout to wait a locked transaction. Optional, `Timespan`, default: 00:01:00
- Version: Define database scheme version (See [Updating Database Version](Updating-Database-Version)). Optional, `Int`, default: 1 

```C#
// Get MyData.db from current path
var db = new LiteDatabase("MyData.db");

// Get string connection from <connectionStrings>
var db = new LiteDatabase("userdb");

// Hard code connection string
var db = new LiteDatabase("filename=C:\Path\mydb.db; journal=false; version=5");
```

### 限制

- Collection Name:
    - Pattern: `A-Z`, `_-`, `0-9`
    - Maxlength of 30 chars
    - Case insensitive
- Index Name: 
    - Pattern: `A-Z`, `_-`, `0-9` (`.` for nested document)
    - Maxlength of 30 chars
    - Case sensitive
- BsonDocument Key Name: 
    - `A-Z`, `_-`, `0-9`
    - Case sensitive
- FileStorage FileId:
    - Pattern: same used in file system path/file.
    - Case sensitive
- Collections: 256 collections per database
- Documents per collections: `UInt.MaxValue`
- FileStorage Max File Length: 2Gb per file
- Index key size: 512 bytes after BSON serialization
- BSON document size: 1.044.224 bytes (~1Mb = 256 pages)
- Nested Depth for BSON Documents: 20 
- Page Size: 4096 bytes
- DataPage Reserved Bytes: 2039 bytes (like PCT FREE)
- IndexPage Reserved Bytes: 100 bytes (like PCT FREE)
- Database Max Length: In theory, `UInt.MaxValue` * PageSize (4096) = ~ Too big!

## 更新数据库版本

LiteDB is has a scheme less data structure. But what happens if you need initialize your database with some data in first time use? And if your database must be update (after deploy your app) and you don't know which version user datafile is?

LiteDB implements a user version database. User version is an **integer** that starts with 1 and can be updated in string connection when you need update your database. User version do not allow double value or letters, so use only positive integers. LiteDB doesn't use this number in any other place.

This resource exists in many embedded databases, like [SQLLite](http://www.sqlite.org/pragma.html#pragma_schema_version) and web [IndexedDb](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Using_IndexedDB) or Entity Framework (Migrations).

To use the user version you must create a class that implements `LiteDatabase` and override `OnVersionUpdate` virtual method.

```C#
public class AppDB : LiteDatabase
{
    public AppDB(string connectionString) : base(connectionString) { }

    protected override void OnVersionUpdate(int newVersion)
    {
        if (newVersion == 1)
        {
            // Do init stuff
        }
        else if (newVersion == 2)
        {
            // Do update version 1 -> 2 stuff
        }
        else if (newVersion == 3)
        {
            // Do update version 2 -> 3 stuff
        }
    }
}
```

When you need to open your database, just add a `version` command in the connection string.

```C#
using(var db = new AppDB("filename=user.db;version=1"))
{
    // database are initialized if file not exists
}
```

And, when you need to update your database structure, just change increase the version number:

```C#
using(var db = new AppDB("filename=user.db;version=2"))
{
    // database are updated to version 2
}
```

Your database can be updated many versions: 

```C#
// If database user version is 1
using(var db = new AppDB("filename=user.db;version=4"))
{
    // LiteDB run:
    // OnVersionUpdate(2);
    // OnVersionUpdate(3);
    // OnVersionUpdate(4);
}
```

## 事务和并发

LiteDB is has a scheme less data structure. But what happens if you need initialize your database with some data in first time use? And if your database must be update (after deploy your app) and you don't know which version user datafile is?

LiteDB implements a user version database. User version is an **integer** that starts with 1 and can be updated in string connection when you need update your database. User version do not allow double value or letters, so use only positive integers. LiteDB doesn't use this number in any other place.

This resource exists in many embedded databases, like [SQLLite](http://www.sqlite.org/pragma.html#pragma_schema_version) and web [IndexedDb](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Using_IndexedDB) or Entity Framework (Migrations).

To use the user version you must create a class that implements `LiteDatabase` and override `OnVersionUpdate` virtual method.

```C#
public class AppDB : LiteDatabase
{
    public AppDB(string connectionString) : base(connectionString) { }

    protected override void OnVersionUpdate(int newVersion)
    {
        if (newVersion == 1)
        {
            // Do init stuff
        }
        else if (newVersion == 2)
        {
            // Do update version 1 -> 2 stuff
        }
        else if (newVersion == 3)
        {
            // Do update version 2 -> 3 stuff
        }
    }
}
```

When you need to open your database, just add a `version` command in the connection string.

```C#
using(var db = new AppDB("filename=user.db;version=1"))
{
    // database are initialized if file not exists
}
```

And, when you need to update your database structure, just change increase the version number:

```C#
using(var db = new AppDB("filename=user.db;version=2"))
{
    // database are updated to version 2
}
```

Your database can be updated many versions: 

```C#
// If database user version is 1
using(var db = new AppDB("filename=user.db;version=4"))
{
    // LiteDB run:
    // OnVersionUpdate(2);
    // OnVersionUpdate(3);
    // OnVersionUpdate(4);
}
```

## 日志 / 恢复

LiteDB is an ACID (Atomicity, Consistency, Isolation, Durability) database, so your data transactions are always consistent across concurrency access.

### 日志

To guarantee data integrity and fail tolerance, LiteDB use a temporary file to write all changes before write on data file. Before write direct to disk, LiteDB create a temp file (called committed journal file) to store all dirty pages. The journal is always located in the same directory as the database file and has the same name as the database file except with the `-journal` appended.

- If there is any error during write journal file, the transaction are aborted - and no changes are made on original data file.
- After write all committed pages on journal file, LiteDB starts writing this pages on your data file. If occur an error at this moment, your data file is corrupted because not all pages was write. LiteDB needs re-write all pages again, using journal file. This recovery operation occurs when you try open database next time.
- When all operation end journal file is deleted.

Journaling is enabled by default. You can disable on string connection to get a fast write operations with some risk!

```C#
var db = new LiteDatabase("filename=mydata.db; journal=false");
``` 

## Shell

LiteDB project has a simple console application that can be used to work with your databases with an application. It's very useful to see, update and test your data. Shell application support all database commands in a similar MongoDB shell syntax.

Shell commands are available in a console application and `LiteDatabase`:

```C#
using(var db = new LiteDatabase("MyDatabase.db"))
{
    db.Run("db.customer.insert { _id:1, Name: \"John Doe\" }");

    var col = db.GetCollection("customer");
    var john = col.FindById(1);
}
```

### 参考

#### Shell控制台命令

The commands here works only in console application `LiteDB.Shell.exe`:

- **`help [full]`** - Display basic or full commands help
- **`open <filename>`** - Open new database. If not exists, create a new one
- **`close`** - Close current database
- **`pretty`** - Show JSON in pretty format
- **`ed`** - Open notepad.exe with last command to edit
- **`run <filename>`** - Read filename and add each line as a new command
- **`spool [on|off]`** - Turn on/off spool of all commands
- **`timer`** - Show a timer on command prompt
- **`--`** - Comments (ignore rest of line)
- **`/<command>/`** - Supports multiline command
- **`quit`** - Exit shell application

#### 集合命令

Syntax: `db.<collection>.<command>`

- **`insert <jsonDoc>`** - Find a document using query filter syntax
    - `db.customer.insert { _id:1, Name: "John Doe", Age: 38 }`
    - `db.customer.insert { Name: "John Doe" }` - Auto create Id as `ObjectId`

- **`bulk <filename.json>`** - Insert all documents inside a JSON file. JSON file must be an JSON array of documents
    - `db.customer.bulk my-documents.json`

- **`update <jsonDoc>`** - Update a existing document using `_id`
    - `db.customer.update { _id:1, Name: "Joana Doe", Age: 29 }`

- **`delete <filter>`** - Delete documents using filter syntax
    - `db.customer.delete _id = 1`
    - `db.customer.delete Name like "Jo"`

- **`ensureIndex <field> [true|<jsonOptions>]`** - Create a new index on collection in field. Support json options [See Indexes](Indexes)
    - `db.customers.ensureIndex Name true` - Create a unique index
    - `db.customers.ensureIndex Name { unique: true, removeAccents: false }`

- **`indexes`** - List all indexes on collections
    - `db.customers.indexes`

- **`dropIndex <field>`** - Drop an index
    - `db.customers.dropIndex Name`

- **`drop`** - Drop a collection and all documents inside. Return false in not exists
    - `db.customer.drop`

- **`rename`** - Rename a collection. Return true if success or false in an error occurs
    - `db.customer.rename my_customers`

- **`count <filter>`** - Count documents with defined filter. See `<filter>` syntax below
    - `db.customer.count Name = "John Doe"`
    - `db.customer.count Age between [20, 40]

- **`min <field>`** - Get lowest value in field index
    - `db.customer.min _id`
    - `db.customer.min Age`

- **`max <field>`** -
    - `db.customer.max _id`
    - `db.customer.max Age`

- **`find [filter][skip N][limit M]`** - Find documents using query filter syntax.  See `<filter>` syntax below
    - `db.customer.find`
    - `db.customer.find limit 10`
    - `db.customer.find Name = "John Doe"`
    - `db.customer.find Age between [20, 40]`
    - `db.customer.find Name LIKE "John" and Age > 30`

- `<filter>` = `<field> [=|>|>=|<|<=|!=|like|contains|in|between] <jsonValue>` 
- `<filter>` = `<filter> [and|or] <filter>`
- `<jsonDoc>` and `<jsonValue>` are JSON extended format. See [Data Structure](Data-Structure).

#### 文件存储

Syntax: `fs.<command>`

- **`upload <fileId> <filename>`** - Upload a new local file to database. If fileId exists, override data content
    - `fs.upload my-file picture.jpg`
    - `fs.upload $/photos/pic-001 C:\Temp\mypictures\picture-1.jpg`

- **`download <fileId> <filename>`** - Download existing database file to a local file.
    - `fs.download my-file copy-picture.jpg`
    - `fs.download $/photos/pic-001 C:\Temp\mypictures\copy-picture-1.jpg`

- **`update <fileId> <jsonDoc>`** - Update file metadata
    - `fs.update my-file { user: "John", machine: "srv001" }`

- **`delete <fileId>`** - Delete a file
    - `fs.delete my-file`

- **`find [fileId]`** - List all files inside database or starting with fileId parameter
    - `fs.find`
    - `fs.find $/photos/`

#### 数据库命令

- **`begin`** - Begin a new transaction
- **`commit`** - Commit current transaction
- **`rollback`** - Rollback current transaction
- **`db.info`** - Show information about database file
