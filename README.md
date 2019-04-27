> branch: vf - Virtual File

I will use this branch to implement new virtual file and new storage processes looking for performance with better memory managment.

The main idea here is:
- Avoid, at all, re-create byte[] in memory for each use - always re-use same heap memory allocation
- Create large memory store (extensions) and use slices to read data page
- Change how BasePage read/write data on page - do not read all page structure (Nodes/DataBlocks) in memory for each page-read
    - Use same structure as SQL Server (index offset on page footer)
- All this will speed up read data and changes will be not re-write all page    
- Back to async write

---

# LiteDB v5

> What's new in v5?

- New Storage Engine
    - New WAL (Write-Ahead Logging) for fast durability
    - Database lock per collection
    - MultiVersion Concurrency Control (Snapshots & Checkpoint)
    - Multi concurrent `Stream` readers - Single async writer
    - No lock for reader
    - Up to 32 indexes per collection
    - Atomic multi-document transactions
    - PageSize: 8KB

- New BsonExpression
    - New super-fast tokenizer parser
    - Clean syntax with optional use of `$`
    - Input/Output parameter support: `@name`
    - Simplified document notation `{ _id, name, year }`
    - Support partial BSON document: read/deserialize only used data in query
    
- System Collections
    - Support query over internal collection 
    - `$transactions`, `$database`, `$dump`

- New QueryBuilder
    - Fluent API for write queries
    - Simple syntax using BsonExpressions
    - Support OrderBy/GroupBy expressions
    - Query optimization with Explain Plan
    - Aggregate functions
    - LINQ to `BsonExpression` query support
    
- New SQL-Like syntax
    - Simple SQL syntax for any command
    - Syntax near to SQL ANSI 
    - Support INSERT/UPDATE/DELETE/...

```SQL
 SELECT { _id, fullname: name.first + ' ' + name.last, age: 2018 - YEAR(birthday) }
   FROM customers
INCLUDE orders
  WHERE name LIKE 'John%'
    AND _id BETWEEN 1 AND 2000
  ORDER BY name
  LIMIT 10

SELECT { city, count: COUNT($), high_pop: MAX(pop) }
  FROM zip
 GROUP BY city
```    
    
- New Native UI - LiteDB.Studio
    - WinForms app to manipulate database
    - Based on SQL commands
    - Show results in grid or as text
    - Multi tabs, multi threads, multi transactions

> What was droped?

- Single process only - optimazed for multi thread (open file as exclusive mode)
- Datafile Encryption (will be external plugin)
- Drop .NET 3.5/4.0 - works only in .NET 4.5+ and .NETStandard 2.0
- Shell commands (use SQL commands)
    
.. but still...   
 
- Embedded support
- Single database file 
- Single DLL, no dependency and 100% C#
- 100% free open source
    
> Roadmap: late in 2018 :smile:
    
---

# LiteDB - A .NET NoSQL Document Store in a single data file

[![Join the chat at https://gitter.im/mbdavid/LiteDB](https://badges.gitter.im/mbdavid/LiteDB.svg)](https://gitter.im/mbdavid/LiteDB?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge) [![Build status](https://ci.appveyor.com/api/projects/status/sfe8he0vik18m033?svg=true)](https://ci.appveyor.com/project/mbdavid/litedb) [![Build Status](https://travis-ci.org/mbdavid/LiteDB.svg?branch=master)](https://travis-ci.org/mbdavid/LiteDB)

LiteDB is a small, fast and lightweight NoSQL embedded database. 

- Serverless NoSQL Document Store
- Simple API similar to MongoDB
- 100% C# code for .NET 4.5 / NETStandard 2.0 in a single DLL (less than 300kb)
- Thread safe and process safe
- ACID in document/operation level
- Data recovery after write failure (journal mode)
- Datafile encryption using DES (AES) cryptography
- Map your POCO classes to `BsonDocument` using attributes or fluent mapper API
- Store files and stream data (like GridFS in MongoDB)
- Single data file storage (like SQLite)
- Index document fields for fast search (up to 16 indexes per collection)
- LINQ support for queries
- Shell command line - [try this online version](http://www.litedb.org/#shell)
- Pretty fast - [compare results with SQLite here](https://github.com/mbdavid/LiteDB-Perf)
- Open source and free for everyone - including commercial use
- Install from NuGet: `Install-Package LiteDB`

## New in 4.0
- New `Expressions/Path` index/query support. See [Expressions](https://github.com/mbdavid/LiteDB/wiki/Expressions)
- Nested `Include` support
- Optimized query execution (with explain plain debug)
- Fix concurrency problems
- Remove transaction and auto index creation
- Support for full scan search and LINQ search
- New shell commands: update fields based on expressions and select/transform documents
- See full [changelog](https://github.com/mbdavid/LiteDB/wiki/Changelog)

## Try online

[Try LiteDB Web Shell](http://www.litedb.org/#shell). For security reasons, in the online version not all commands are available. Try the offline version for full feature tests.

## Documentation

Visit [the Wiki](https://github.com/mbdavid/LiteDB/wiki) for full documentation. For simplified chinese version, [check here](https://github.com/lidanger/LiteDB.wiki_Translation_zh-cn).

## Download

Download the source code or binary only in [LiteDB Releases](https://github.com/mbdavid/LiteDB/releases)

## How to use LiteDB

A quick example for storing and searching documents:

```C#
// Create your POCO class
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
    public string[] Phones { get; set; }
    public bool IsActive { get; set; }
}

// Open database (or create if doesn't exist)
using(var db = new LiteDatabase(@"MyData.db"))
{
    // Get customer collection
    var col = db.GetCollection<Customer>("customers");

    // Create your new customer instance
	var customer = new Customer
    { 
        Name = "John Doe", 
        Phones = new string[] { "8000-0000", "9000-0000" }, 
        Age = 39,
        IsActive = true
    };
    
    // Create unique index in Name field
    col.EnsureIndex(x => x.Name, true);
	
    // Insert new customer document (Id will be auto-incremented)
    col.Insert(customer);
	
    // Update a document inside a collection
    customer.Name = "Joana Doe";
	
    col.Update(customer);
	
    // Use LINQ to query documents (with no index)
    var results = col.Find(x => x.Age > 20);
}
```

Using fluent mapper and cross document reference for more complex data models

```C#
// DbRef to cross references
public class Order
{
    public ObjectId Id { get; set; }
    public DateTime OrderDate { get; set; }
    public Address ShippingAddress { get; set; }
    public Customer Customer { get; set; }
    public List<Product> Products { get; set; }
}        

// Re-use mapper from global instance
var mapper = BsonMapper.Global;

// "Produts" and "Customer" are from other collections (not embedded document)
mapper.Entity<Order>()
    .DbRef(x => x.Customer, "customers")   // 1 to 1/0 reference
    .DbRef(x => x.Products, "products")    // 1 to Many reference
    .Field(x => x.ShippingAddress, "addr"); // Embedded sub document
            
using(var db = new LiteDatabase("MyOrderDatafile.db"))
{
    var orders = db.GetCollection<Order>("orders");
        
    // When query Order, includes references
    var query = orders
        .Include(x => x.Customer)
        .Include(x => x.Products) // 1 to many reference
        .Find(x => x.OrderDate <= DateTime.Now);

    // Each instance of Order will load Customer/Products references
    foreach(var order in query)
    {
        var name = order.Customer.Name;
        ...
    }
}

```

## Where to use?

- Desktop/local small applications
- Application file format
- Small web applications
- One database **per account/user** data store
- Few concurrent write operations

## Plugins

- A GUI viewer tool: https://github.com/falahati/LiteDBViewer
- A GUI editor tool: https://github.com/JosefNemec/LiteDbExplorer 
- Lucene.NET directory: https://github.com/sheryever/LiteDBDirectory
- LINQPad support: https://github.com/adospace/litedbpad
- F# Support: https://github.com/Zaid-Ajaj/LiteDB.FSharp

## Changelog

Change details for each release are documented in the [release notes](https://github.com/mbdavid/LiteDB/releases).

## License

[MIT](http://opensource.org/licenses/MIT)

Copyright (c) 2017 - Maurício David

## Thanks

A special thanks to @negue and @szurgot helping with portable version and @lidanger for simplified chinese translation. 
