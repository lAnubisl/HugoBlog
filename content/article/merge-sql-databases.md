---
title: "Merge SQL databases"
date: 2016-05-27T11:55:25+03:00
draft: false
Summary: "Imagine you have 2 databases with identical schema. These databases were working for different application instances for some time. And now you need to merge them together for some reason. You can use different tools to achieve this goal. For example [dbForge Data Compare](https://www.devart.com/ru/dbforge/sql/datacompare/) or [SQL Data Compare](http://www.red-gate.com/products/sql-development/sql-data-compare/). But these tools cost money and if you don't merge databases every day this is probably not an option for you. Also these tools does not know full specific of your database structure including unique indexes, check constraints and triggers. Another big deal is identity columns that are using as primary keys. For two databases these keys can be same but represent different entities.  In my practice I face database merge task second time and here is how I handle it."
---

Imagine you have 2 databases with identical schema. These databases were working for different application instances for some time. And now you need to merge them together for some reason.

You can use different tools to achieve this goal. For example [dbForge Data Compare](https://www.devart.com/ru/dbforge/sql/datacompare/) or [SQL Data Compare](http://www.red-gate.com/products/sql-development/sql-data-compare/). But these tools cost money and if you don't merge databases every day this is probably not an option for you. Also these tools does not know full specific of your database structure including unique indexes, check constraints and triggers.

Another big deal is identity columns that are using as primary keys. For two databases these keys can be same but represent different entities. 

In my practice I face database merge task second time and here is how I handle it.

## Step 1\. Identify tables that must be merged.

If you need to merge only several tables of hundred you probably don't want to have a deal with full database schema. You also definitely want to skip lookup tables such as "AddressType" it it holds only records like "1 | Home" and "2 | Business".

``` sql
 use TargetDB;
 go

 declare @TablesToImport table 
 (
   Name nvarchar(128)
 )

 insert into @TablesToImport (Name) values 
   ('Users'), ('Orders'), ('Products'), ('Categories')
```

## Step 2\. Get information about identity columns.

Identity columns are important. Usually they are used as Primary Keys for table rows. But if two databases were working for different application instances you can think that even if identity primary keys are the same for User table for example they represent different entities and there should be no duplicate records. In this case you need to modify all identity primary keys in source database to make data graphs non-overlapped. But first, you need to identify these identity columns.

``` sql
 declare @TableInformation table 
 (
   [Schema] nvarchar(128),
   Name nvarchar(128),
   IdentityColumn nvarchar(128),
   MaxIdentity bigint
 )

 -- SCAN SYSTEM TABLES
 insert into @TableInformation 
 (
   [Schema], 
   [Name], 
   [IdentityColumn]
 ) select 
     TABLE_SCHEMA, 
     TABLE_NAME, 
     COLUMN_NAME
   from INFORMATION_SCHEMA.COLUMNS
   where COLUMNPROPERTY(
     OBJECT_ID(TABLE_NAME), 
     COLUMN_NAME, 
     'IsIdentity') = 1
   and OBJECTPROPERTY(
     OBJECT_ID(TABLE_SCHEMA + '.' + TABLE_NAME), 
     'IsTable') = 1
   and TABLE_NAME in
   (
     select [Name] 
     from @TablesToImport
   )
 ```

## Step 3\. Calculate max identity value.

Now you have Table that contain all identity column information. Next you beed to find the greatedt identity value for all tables and calculate max value among all of your tables

``` sql
declare @Schema nvarchar(128),
   @Name NVARCHAR(128),
   @IdentityColumn NVARCHAR(128);

 declare TablesCursor cursor for 
   select [Schema], [Name], [IdentityColumn] 
   from @TableInformation

 open TablesCursor

 fetch next from TablesCursor 
   into @Schema, @Name, @IdentityColumn;

 while @@FETCH_STATUS = 0
 begin
   declare @Select nvarchar (max),
     @ParmDefinition nvarchar(500),
     @Result bigint;

   -- GENERATE DYNAMIC SQL STATEMENT	
   set @Select = N'select @Result = 
     max(' + @IdentityColumn + ') from ' + @Schema + '.' + @Name
   set @ParmDefinition = N'@Result bigint output';

   -- EXECUTE DYNAMIC SQL STATEMENT
   exec sp_executesql @Select, @ParmDefinition, @Result = @Result output;

   -- UPDATE MaxIdentity for Identity Column
     update @TableInformation 
       set MaxIdentity = @Result 
       where [Schema] = @Schema 
       and [Name] = @Name 
       and [IdentityColumn] = @IdentityColumn

   fetch next from TablesCursor 
     into @Schema, @Name, @IdentityColumn;
 end

 close TablesCursor
 deallocate TablesCursor

 -- DISPLAY TableInformation
 select * from @TableInformation

 -- DISPLAY MaxIdentity
 select MAX(MaxIdentity) from @TableInformation

```

## Step 4\. Identify Foreign keys that needs to be incremented along with Primary keys.

In step 3 you calculated the biggest identity value among all tables that must be merged. Now we are ready to increase all identities in source database. But you have to keep in mind that all references must not be broken. Because of that we have to identify all Foreign Keys for identity columns and increase them too.

``` sql
use SourceDB;
 go

 declare @Increment bigint
 set @Increment = 6000 -- value from step 3

 declare @TablesToImport table 
 (
   [Name] nvarchar(128)
 )

 insert into @TablesToImport (Name) values 
   ('Users'), ('Orders'), ('Products'), ('Categories')

 declare @FkToIncrement table 
 (
   [Schema] nvarchar(128),
   [Table] nvarchar(128),
   [Column] nvarchar(128)
 )

 -- SCAN SYSTEM TABLES
 insert into @FkToIncrement ([Schema], [Table], [Column])
 select
   sc.TABLE_SCHEMA,
   t.name [table], 
   c.name ForeignKeyColumn 
 from sys.foreign_key_columns as fk
   inner join sys.tables as t 
     on fk.parent_object_id = t.object_id
   inner join sys.columns as c 
     on fk.parent_object_id = c.object_id 
     and fk.parent_column_id = c.column_id
   inner join INFORMATION_SCHEMA.COLUMNS sc
     on sc.TABLE_NAME = t.name
     and sc.COLUMN_NAME = c.name
 where fk.referenced_object_id in 
   (
     select object_id 
     from sys.tables 
     where name in 
     (
       select [Name] 
       from @TablesToImport
     )
   )
   and t.name in 
   (
     select [Name] 
       from @TablesToImport
   )
```

## Step 5\. Remove all FullText indexed, Foreign Keys and Primary keys.

Now we need to remove everything that can stop us to change PK and FK column values. I recommend to delete FullText Indexes, Foreign Keys and Primary Keys. And that's why we identify FK columns step before (because after this step we will not be able to recognize them).

``` sql
 -- REMOVE ALL FULLTEXT INDEXES
 declare @dropFtSql nvarchar(max)
 set @dropFtSql = ''

 select @dropFtSql = @dropFtSql 
   + 'drop fulltext index on ' + t.name + ';'
   from sys.tables t 
   inner join sys.fulltext_indexes fi 
     on t.[object_id] = fi.[object_id] 

 if (@dropFtSql <> '')
 begin
   execute (@dropFtSql)
 end   
 print 'all fulltext indexes are removed'

 -- REMOVE ALL FOREIGN KEYS
 declare @removeFkSql nvarchar(MAX) 
 set @removeFkSql = ''

 select @removeFkSql = @removeFkSql + 'alter table ' 
   + QUOTENAME(CONSTRAINT_SCHEMA) + '.' 
   + QUOTENAME(TABLE_NAME) + ' drop constraint ' 
   + QUOTENAME(CONSTRAINT_NAME) + '; ' 
 from INFORMATION_SCHEMA.TABLE_CONSTRAINTS
   where CONSTRAINT_TYPE = 'FOREIGN KEY'

 if (@removeFkSql <> '')
 begin
   execute(@removeFkSql)
 end
 print 'all foreign keys are removed'

 -- REMOVE ALL PRIMARY KEYS
 declare @removePkSql nvarchar(MAX) 
 set @removePkSql = '' 

 select @removePkSql = @removePkSql + 'alter table ' 
   + QUOTENAME(CONSTRAINT_SCHEMA) + '.' 
   + QUOTENAME(TABLE_NAME) + ' drop constraint ' 
   + QUOTENAME(CONSTRAINT_NAME) + '; ' 
 from INFORMATION_SCHEMA.TABLE_CONSTRAINTS
   where CONSTRAINT_TYPE = 'PRIMARY KEY'

 if (@removePkSql <> '')
 begin
   execute(@removePkSql)
 end
 print 'all primary keys are removed'
 ```

## Step 6\. Increment all FK column values.

``` sql
declare @Schema nvarchar(128),
   @Table NVARCHAR(128),
   @Column NVARCHAR(128),
   @DataType NVARCHAR(128);

 declare FkCursor cursor for 
   select * from @FkToIncrement

 open FkCursor

 fetch next from FkCursor 
   into @Schema, @Table, @Column
 while @@FETCH_STATUS = 0
 begin
   execute('update ['+@Schema+'].['+@Table+'] 
     set ['+@Column+'] = ['+@Column+'] + ' + @Increment)
   fetch next from FkCursor into @Schema, @Table, @Column
 end

 close FkCursor
 deallocate FkCursor

 print 'all foreign keys are incremented'
```

## Step 7\. Increment all identity columns.

Identity columns are tricky. You cannot just remove "Identity" property from them and you also cannot update values in these columns. The easiest way to do this is to create a copy of identity column, drop the original one and rename the copy.

``` sql
declare @IdentityToIncrement table 
 (
   [Schema] nvarchar(128),
   [Table] nvarchar(128),
   [Column] nvarchar(128),
   [DataType] nvarchar(128)
 )

 -- SCAN SYSTEM TABLES
 insert into @IdentityToIncrement 
   (
     [Schema], 
     [Table], 
     [Column], 
     [DataType]
   )
   select 
     TABLE_SCHEMA, 
     TABLE_NAME, 
     COLUMN_NAME, 
     DATA_TYPE
   from INFORMATION_SCHEMA.COLUMNS
     where COLUMNPROPERTY(
       OBJECT_ID(TABLE_NAME), 
       COLUMN_NAME, 
       'IsIdentity') = 1
     and OBJECTPROPERTY(
       OBJECT_ID(TABLE_SCHEMA + '.' + TABLE_NAME), 
       'IsTable') = 1
     and TABLE_NAME IN (
       select Name 
       from @TablesToImport
     )

 -- INCREMENT IDENTITY COLUMNS		
 declare IdentityCursor cursor for 
   select * from @IdentityToIncrement

 open IdentityCursor

 fetch next from IdentityCursor 
   into @Schema, @Table, @Column, @DataType

 while @@FETCH_STATUS = 0
 begin
   execute('alter table ['+@Schema+'].['+@Table+'] add '+@Column+'_Copy ' + @DataType + '')
   execute('update ['+@Schema+'].['+@Table+'] set '+@Column+'_Copy = '+@Column+'')
   execute('alter table ['+@Schema+'].['+@Table+'] drop column '+@Column+'')
   execute('exec sp_rename ''['+@Schema+'].['+@Table+'].'+@Column+'_Copy'', '''+@Column+''', ''COLUMN''')
   execute('update ['+@Schema+'].['+@Table+'] set ['+@Column+'] = ['+@Column+'] + ' + @Increment)
   execute('alter table ['+@Schema+'].['+@Table+'] alter column ['+@Column+'] ' + @DataType + ' not null')
   fetch next from IdentityCursor into @Schema, @Table, @Column, @DataType
 end

 close IdentityCursor
 deallocate IdentityCursor

 print 'all identity columns are incremented'
```

## Step 8\. Handle constraint violations.

At this point of time you have 2 data graphs that potentially can be merged. But you probably have UNIQUE INDEXes or other CHECK CONSTRAINTs that help you to have data consistent. For example you cannot have 2 users with same login. But 2 databases can have same login for different user and here you must make a decision about how to handle this. It's up to you if you want to remove this user or change it's login. 

## Step 9\. Merge.

Finally we can move data from source database to target one. You can use any tool to automate this process, but you also can use pure SQL that looks like this:

``` sql
 -- MERGE Categories
 set identity_insert TargetDB.dbo.Categories on
 insert into TargetDB.dbo.Categories ([Id], [Name])
 select [Id],[Name]
   from SourceDB.dbo.Categories
 set identity_insert TargetDB.dbo.Categories off

 -- MERGE Products
 set identity_insert TargetDB.dbo.Products on
 insert into TargetDB.dbo.Products ([Id], [Name], [CategoryId])
 select [Id],[Name],[CategoryId]
   from SourceDB.dbo.Products
 set identity_insert TargetDB.dbo.Products off
```

That's it. I hope this was not too complicated. And [Here is the Gist with all steps together](https://gist.github.com/lAnubisl/c7656fecd4db571e5ec57d1a7ce8bac1).