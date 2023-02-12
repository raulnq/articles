# Accessing an Informix database in .NET

[Informix](https://www.ibm.com/products/informix) was a database (part of the IBM family) popular during the 90s and today still in the market. Due to this, it is always good to know how we can use it from our .NET applications (who knows, maybe someday we will have to work with it). First, let's start an instance of the database with the following command:

```bash
docker run --name informix-sandbox -i -d -e SIZE=medium --privileged -p 9088:9088 -p 9089:9089 -p 27017:27017 -p 27018:27018 -p 27883:27883 -e LICENSE=accept ibmcom/informix-developer-database:12.10.FC9W1DE
```

These are different ports/protocols through which we can access the database:

* `-p`, expose port `9088` to allow remote connections from TCP clients
    
* `-p`, expose port `9089` to allow remote connections from DRDA clients
    
* `-p`, expose port `27017` to allow remote connections from Mongo clients
    
* `-p`, expose port `27018` to allow remote connections from REST clients
    
* `-p`, expose port `27883` to allow remote connections from MQTT clients
    

The default user is `informix` with `in4mix` as password. The DRDA protocol is used by the .NET provider that we will see. If you already have a database available, verify that the protocol is enabled. If not, you can follow [this](https://wiki.genexus.com/commwiki/servlet/wiki?49253,Enabling+DRDA+on+Informix) article to enable it.

**IBM Data Server .NET Provider** is the current client for the Microsoft Ecosystem. This package is self-contained and no other installation is needed to connect to IBM Database servers. Every supported platform has its Nuget Package:

* [Windows](https://www.nuget.org/packages/Net.IBM.Data.Db2/)
    
* [Linux](https://www.nuget.org/packages/Net.IBM.Data.Db2-lnx)
    
* [macOS](https://www.nuget.org/packages/Net.IBM.Data.Db2-osx)
    

And support multiples IBM database servers:

* zOS
    
* LUW including IBM dashDB
    
* Db2
    
* Informix
    

But, there are some limitations:

* Only 64-bit applications are supported.
    
* For this package to work, there should not be any other DB2Connect installations present in the system.
    
* Developers need to modify the package since there are different packages for each platform.
    

[Here](https://community.ibm.com/community/user/datamanagement/blogs/vishwa-hs1/2020/07/12/db2-net-packages-download-and-configure) you can find information about all the supported .NET versions. Plus, there is Entity Framework support using **IBM.EntityFrameworkCore** Nuget Package (remember, there is a different package per each platform). Let's build an API to store and list our favorite animes. Run the following commands:

```bash
dotnet new webapi -n AnimeAPI
dotnet add AnimeAPI package Dapper
dotnet add AnimeAPI package Net.IBM.Data.Db2
dotnet new sln -n InformixSandbox
dotnet sln add --in-root AnimeAPI
```

Open the solution and create the `Anime.cs` file with the following content:

```csharp
public class Anime
{
    public string Id { get; set; } = null!;
    public string Name { get; set; } = null!;
}
```

The `Repository.cs` file with the content as follows:

```csharp
public class Repository
{
    private readonly string _connectionString;

    public Repository(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task Save(Anime anime)
    {
        using (var db = new DB2Connection(_connectionString))
        {
            string sqlQuery = "INSERT INTO animes (id, name) VALUES(@Id, @Name)";

            await db.ExecuteAsync(sqlQuery, anime);
        }
    }

    public async Task<IEnumerable<Anime>> List()
    {
        using (var db = new DB2Connection(_connectionString))
        {
            string sqlQuery = "SELECT * FROM animes";

            return await db.QueryAsync<Anime>(sqlQuery);
        }
    }
}
```

Replace the default controller class with:

```csharp
[ApiController]
[Route("animes")]
public class AnimesController : ControllerBase
{
    private readonly Repository _repository;

    public AnimesController(Repository repository)
    {
        _repository = repository;
    }

    [HttpPost()]
    public async Task<Guid> Post([FromBody] RegisterAnime command)
    {
        var id = Guid.NewGuid();

        var anime = new Anime() { Id = id.ToString(), Name = command.Name };

        await _repository.Save(anime);

        return id;
    }

    [HttpGet()]
    public Task<IEnumerable<Anime>> Get()
    {
        return _repository.List();
    }

    public record RegisterAnime(string Name);
}
```

Finally, update the `Program.cs` file to create the corresponding table to store our data:

```csharp
using AnimeAPI.Controllers;
using Dapper;
using IBM.Data.Db2;

var builder = WebApplication.CreateBuilder(args);
var connectionString = "Database=sysmaster;Server=localhost:9089;UID=informix;PWD=in4mix;";
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddSingleton(new Repository(connectionString));
var app = builder.Build();

using (var conn = new DB2Connection(connectionString))
{
    var exists = false;
    conn.Open();
    var reader = await conn.ExecuteReaderAsync("select 1 from systables where tabname = 'animes'");
    if (reader.HasRows)
    {
        exists = true;
    }
    if (!exists)
    {
        var sql = @"CREATE TABLE animes
        (
        id varchar(36) PRIMARY KEY,
        name varchar(255) NOT NULL
        );";
        await conn.ExecuteAsync(sql);
    }
}
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

Run the application and test the API. If we see an error like this:

> IBM.Data.DB2.Core.DB2Exception (0x80004005): ERROR \[08001\] \[IBM\] SQL30081N A communication error has been detected. Communication protocol being used: "TCP/IP". Communication API being used: "SOCKETS". Location where the error was detected: "10.2.4.25". Communication function detecting the error: "recv". Protocol specific error ode(s): "\*", "\*", "0". SQLSTATE=08001

That means we are not using the right DRDA port. The code is available [here](https://github.com/raulnq/informix-sandbox). Thank you, and happy coding.