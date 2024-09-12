---
title: "Azure SQL Database Export to Localhost using SqlPackage"
date: 2024-09-12T06:53:27Z
draft: false
keywords: "azure, sql, database, backup, sqlpackage"
description: "How to create local backup of azure sql database."
Summary: "
![](/images/azure-sql-database-export-to-local/terminal.png)
Exporting an Azure SQL Server database locally is a common task for developers and administrators who need to create backups, test changes, or migrate databases to other environments. If your SQL Server instance is configured with Entra ID (formerly Azure AD) authentication, you might face an issue connecting to the database with username and password from the command line tool like [SqlPackage](https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage?view=sql-server-ver16).

In this guide, I’ll walk you through the process of exporting an Azure SQL Server database to your local windows machine using Powershell, SqlPackage tool and Azure SQL Server with Entra ID authentication."
---

![](/images/azure-sql-database-export-to-local/terminal.png)

Exporting an Azure SQL Server database locally is a common task for developers and administrators who need to create backups, test changes, or migrate databases to other environments. If your SQL Server instance is configured with Entra ID (formerly Azure AD) authentication, you might face an issue connecting to the database with username and password from the command line tool like [SqlPackage](https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage?view=sql-server-ver16).

In this guide, I’ll walk you through the process of exporting an Azure SQL Server database to your local windows machine using Powershell, SqlPackage tool and Azure SQL Server with Entra ID authentication.

## Prerequisites
Before we begin, ensure the following:

- SqlPackage [Installed](https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-download?view=sql-server-ver16).
- Entra ID Access: You must have the necessary permissions (like DBOwner or DBReader) to access the database using your Entra ID.
- Azure SQL Database: Ensure the database you want to export is accessible.

## Export

To authenticate with your credentials including MFA use the syntax as shown in the last example of the [documentation page](https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-export?view=sql-server-ver16#examples).

``` console
sqlpackage.exe `
	/Action:Export `
	/TargetFile:"C:\Users\User\{BACKUP_FILE_NAME}.bacpac" `
    /UniversalAuthentication:True `
	/SourceConnectionString:"Server=tcp:{SQL_SERVER_NAME}.database.windows.net,1433;Initial Catalog={DATABASE_NAME};MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
```

The important part here is `/UniversalAuthentication:True`. When set to True, the interactive authentication protocol is activated supporting Multi-Factor Authentication.

The commang will trigger browser window to open and ask you to authenticate:

![](/images/azure-sql-database-export-to-local/signin.png)

After the authentication the command will proceed:

``` console
Connecting to database 'db-mydatabase' on server 'tcp:sql-myserver.database.windows.net,1433'.
Extracting schema
Extracting schema from database
Resolving references in schema model
Validating schema model
Validating schema model for data package
Validating schema
Exporting data from database
Exporting data
Processing Export.
Processing Table '[gde].[task]'.
Processing Table '[gde].[comment]'.
Processing Table '[gde].[tag]'.
Processing Table '[gde].[user]'.
Successfully exported database and saved it to file 'C:\Users\User\my_database_backup.bacpac'.
Changes to connection setting default values were incorporated in a recent release.  More information is available at https://aka.ms/dacfx-connection
Time elapsed 0:04:31.81
```

## Conclusion

Using SqlPackage with Entra ID authentication, you can easily export an Azure SQL Server database to your local environment. This method avoids the need for SQL Server authentication and leverages the security of Entra ID. With the right permissions and tools in place, exporting a database is straightforward and can be done with just a few command-line instructions.