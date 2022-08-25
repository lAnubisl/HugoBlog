---
title: "Dapper queries synchronized with MSSQL database schema"
date: 2022-08-25T12:43:45+02:00
summary: Dapper is a MicroORM that allows you to control SQL queries you are executing and removes the pain of mapping the dataset results back to your domain model. The thing is that when you specify SQL queries you have to make sure they are valid against the current Database schema. One solution is to use Stored Procedures... other one is this...
draft: false
---

[Dapper](https://github.com/DapperLib/Dapper) is a Micro ORM that allows you to control SQL queries you are executing and removes the pain of mapping the dataset results back to your domain model. The thing is that when you specify SQL queries you have to make sure they are valid against the current Database schema. One solution is to use [Stored Procedures](https://docs.microsoft.com/en-us/sql/relational-databases/stored-procedures/create-a-stored-procedure?view=sql-server-ver16)... other one is this:

### Database project

[Database project](https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-data-tools/hh272702(v=vs.103)?redirectedfrom=MSDN) is a special type of projects in Visual Studio which allows you to have MSSQL database schema in a source control.
![11](https://visualstudio.microsoft.com/wp-content/uploads/2021/10/Swimlane1.png)
It allows you to build the schema, compare it with the target database, generate change scripts and publish to the environment. It is possible to use the database project programmatically to build the C# definition of the database schema. In order to achieve this, we are going to use [T4 Text Templates](https://docs.microsoft.com/en-us/visualstudio/modeling/code-generation-and-t4-text-templates?view=vs-2022). Here is the exact template you can use:

#### **`Template.tt`**
``` c#
<#@ template debug="false" hostspecific="false" language="C#" #>
<#@ output extension=".cs" #>
<#@ SqlModelDirective processor="SqlModelDirectiveProcessor" #>
<#@ import namespace="System.Text" #>
<#@ import namespace="Microsoft.SqlServer.Dac" #>
<#@ import namespace="System.Collections.Generic" #>
<#@ import namespace="Microsoft.SqlServer.Dac.Model" #>
<# List<TSqlObject> allTables = GetAllTablesWithViews();
    WriteLine(@"#pragma warning disable SA1402 // File may only contain a single type");
    WriteLine(@"#pragma warning disable SA1629 // Documentation text should end with a period");
    WriteLine(@"#pragma warning disable SA1649 // File name should match first type name");
    WriteLine(@"namespace MyLibrary.Bll");
    WriteLine(@"{");
    foreach (var table in allTables)
    {
        var tableName = table.Name.Parts[1];
        var isTable = table.ObjectType.Name == "Table";
        WriteLine("");
        WriteLine($"    /// <summary>");
        WriteLine($"    /// {table.Name}");
        WriteLine($"    /// </summary>");
        WriteLine($"    internal static class {tableName}");
        WriteLine($"    {{");
        WriteLine($"        /// <summary>");
        WriteLine($"        /// {table.Name}");
        WriteLine($"        /// </summary>");
        WriteLine($"        internal static readonly string Table = \"[{tableName}]\";");	
        var columns = table.GetReferenced(isTable ? Table.Columns : View.Columns); 
        foreach(var column in columns)
        {
            WriteLine("");
            WriteLine($"        /// <summary>");
            WriteLine($"        /// {column.Name}");
            WriteLine($"        /// </summary>");
            WriteLine($"        internal static readonly string {column.Name.Parts[2]} = \"[{column.Name.Parts[2]}]\";");
        }
        WriteLine(@$"    }}");
    }
    WriteLine(@"}");
    WriteLine(@"#pragma warning restore SA1649 // File name should match first type name");
    WriteLine(@"#pragma warning restore SA1402 // File may only contain a single type");
    WriteLine(@"#pragma warning restore SA1629 // Documentation text should end with a period");
#>
<#+
    public List<TSqlObject> GetAllTablesWithViews()
    {
        List<TSqlObject> allObjects = new List<TSqlObject>();
        var model = Model;
        if (model != null)
        {
            var tables = model.GetObjects(DacQueryScopes.All, ModelSchema.Table);
            var views = model.GetObjects(DacQueryScopes.All, ModelSchema.View);
            if (tables != null)
            {
                allObjects.AddRange(tables);
            }
            if (views != null)
            {
                allObjects.AddRange(views);
            }
        }
        return allObjects;
    }
#>
```

This template reads all tables and views from the database project and generates C# class with the definition of each table and view as static read-only strings.

``` c#
namespace MyLibrary.Bll
{

    /// <summary>
    /// [dbo].[TblPerson]
    /// </summary>
    internal static class TblPerson
    {
        /// <summary>
        /// [dbo].[TblPerson]
        /// </summary>
        internal static readonly string Table = "[TblPerson]";

        /// <summary>
        /// [dbo].[TblPerson].[UserId]
        /// </summary>
        internal static readonly string UserId = "[UserId]";

        /// <summary>
        /// [dbo].[TblPerson].[FirstName]
        /// </summary>
        internal static readonly string FirstName = "[FirstName]";

        /// <summary>
        /// [dbo].[TblPerson].[LastName]
        /// </summary>
        internal static readonly string LastName = "[LastName]";

        /// <summary>
        /// [dbo].[TblPerson].[VNumber]
        /// </summary>
        internal static readonly string VNumber = "[VNumber]";
    }
}
```

The next part of the trick is to link the generated 'Template.cs' file to the project library that uses Dapper. In order to do this, add 'Template.cs' as linked file (Add -> Existing Item -> Add as Link).

Now you have 'Template.cs' file that reflects all database schema tables and views. Just don't forget to re-generate it from the template every time you modify the database schema.

Using this file, you now are able to use it as part of your SQL queries. For example:

``` c#
internal static partial class Queries
{
    internal static string GetUsers =
        $@"
        SELECT 
            u.{TblUser.Id} AS {nameof(User.Id)},
            u.{TblUser.Email} AS {nameof(User.Email)},
            u.{TblUser.StartDate} AS {nameof(User.StartDate)},
            u.{TblUser.EndDate} AS {nameof(User.EndDate)},
            u.{TblUser.Participants} AS {nameof(User.Participants)},
            p.{TblPerson.FirstName} AS {nameof(Person.FirstName)},
            p.{TblPerson.LastName} AS {nameof(Person.LastName)},
            p.{TblPerson.VNumber} AS {nameof(Person.VNumber)}
            FROM
            (
                SELECT TOP (@{nameof(GetUsersParam.Take)}) *
                FROM {TblUser.Table}
                WHERE 
                    {TblUser.Location} = @{nameof(GetUsersParam.Location)}
                AND {TblUser.Participants} = @{nameof(GetUsersParam.Participants)}
                AND {TblUser.RegisteredDate} > @{nameof(GetUsersParam.FromRegistrationDate)}
            ) u INNER JOIN {TblPerson.Table} p ON p.{TblPerson.UserId} = u.{TblUser.Id}";

    internal class GetUsersParam
    {
        public GetUsersParam(Location location, int participants, int take, DateTime fromRegistrationDate)
        {
            Location = (int)location;
            Participants = participants;
            Take = take;
            FromRegistrationDate = fromRegistrationDate;
        }

        public DateTime FromRegistrationDate { get; }

        public int Location { get; }

        public int Take { get; }

        public int Participants { get; }
    }
}
```

Can you see the '**{TblUser.Email}**' and parts like that in the query definition? If you rename or delete the 'Email' column from 'TblUser' database table you get the compilation error pointing you to the SQL query that uses 'TblUser.Email' which does not exist anymore.

### Query to domain model synchronization

You could also see the other part of the query: '**u.{TblUser.Email} AS {nameof(User.Email)}**'. As described above the '**{TblUser.Email}**' part sticks the SQL query to the database schema. From the other side '**{nameof(User.Email)}**' sticks the SQL query to the domain model property.

``` c#
public class User
{
    public Guid Id { get; set; }

    public int Participants { get; set; }

    public string Email { get; set; }

    public string Phone { get; set; }

    public DateTime StartDate { get; set; }

    public DateTime EndDate { get; set; }

    public Location Location { get; set; }

    public DateTime RegisteredDate { get; set; }

    public Person[] People { get; set; }
}
```

If you define your SQL queries this way you will never miss the change in database schema or in domain model definition. Fixing defects of this kind is a pain. Believe me I had it.

At last, here is an example of how I execute query from example above and map the result to the domain model:

``` c#
var users = new Dictionary<Guid, User>();
var people = new Dictionary<Guid, Collection<Person>>();
await connection.QueryAsync<User, Person, User>(
    Queries.GetUsers,
    (u, p) =>
    {
        if (!users.ContainsKey(u.Id))
        {
            users.Add(u.Id, u);
        }

        if (!people.ContainsKey(u.Id))
        {
            people.Add(u.Id, new Collection<Person>());
        }

        people[u.Id].Add(p);
        return null!;
    },
    new Queries.GetUsersParam(location, participants, take, fromRegisteredDate ?? new DateTime(2022, 1, 1, 0, 0, 0, DateTimeKind.Utc)),
    splitOn: nameof(Person.FirstName));
foreach (var kvp in users)
{
    kvp.Value.People = people[kvp.Key].ToArray();
}

return users.Values.ToArray();
```