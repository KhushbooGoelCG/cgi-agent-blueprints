---
description: 'Guidelines for sql database .net project'
applyTo: "src/*.SqlDb/**"
---

# **GitHub Copilot Instructions for `<ProjectName>.SqlDb.sqlproj`**

## **Purpose**

This document instructs GitHub Copilot on how to generate, maintain, and enhance SQL code within the **SQL Database Project** (`<ProjectName>.SqlDb.sqlproj`) in Visual Studio.  
Copilot should follow the standards, conventions, and architecture defined below when suggesting or completing code.

***

## **Project Overview**

`<ProjectName>.SqlDb.sqlproj` is a SQL Server Database Project containing schema objects that support the application platform, including tables, views, stored procedures, functions, security objects, and reference/lookup data.

Build output must remain compatible with **SQL Server Data Tools (SSDT)** and Visual Studio database publishing.

***

# **General Rules for Copilot**

### **1. Follow SSDT Folder + File Structure**

When generating new objects:

    dbo\Tables
      <EntityName>.sql
    dbo\StoredProcedures
      sp_<Action>_<Entity>.sql
    dbo\Views
      vw_<Entity>.sql
    dbo\Functions
      fn_<Description>.sql
    dbo\Security
      Roles.sql
      Permissions.sql

Use **one object per file**, named correctly.

***

### **2. Always Use Schema Qualification**

Default schema is:

    [dbo]

Always fully qualify:

*   Tables → `[dbo].[TableName]`
*   Procedures → `[dbo].[sp_Action_Entity]`
*   Functions → `[dbo].[fn_Description]`

***

### **3. SQL Coding Standards**

Copilot must follow these conventions:

#### **Table Naming**

*   Table names must be singular (e.g., Customer instead of Customers, Contact instead of Contacts).


#### **Formatting**

*   Uppercase keywords (`SELECT`, `JOIN`, `WHERE`, etc.)
*   Four‑space indentation
*   One statement per line

#### **Identifiers**

*   PascalCase for tables and columns
*   camelCase for variables
*   Avoid abbreviations unless domain‑approved (`Banking`, `ETL`, `ID`)


### **4. Data Modeling Rules**

#### **Primary Keys**

All tables must have:

```sql
<PrimaryKeyName> INT IDENTITY(1,1) NOT NULL PRIMARY KEY
```

#### **Foreign Keys**

Always include:

*   ON DELETE NO ACTION
*   ON UPDATE NO ACTION

Unless business rules specify otherwise.

#### **Naming**

| Object Type  | Rule                            |
| ------------ | ------------------------------- |
| PK           | PK\_<TableName>                 |
| FK           | FK\_<ChildTable>\_<ParentTable> |
| IX           | IX\_<TableName>\_<ColumnName>   |
| Unique Index | UX\_<TableName>\_<ColumnName>   |

***

### **5. Domain Rules**

For new tables, include:

```sql
CreatedDate DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
CreatedBy NVARCHAR(100) NOT NULL
ModifiedDate DATETIME2 NULL
ModifiedBy NVARCHAR(100) NULL
IsActive BIT NOT NULL DEFAULT 1
```

***

### **6. Project Integration (`<ProjectName>.SqlDb.sqlproj`)**

####   When adding a new database object, ensure it is registered in the `<ProjectName>.SqlDb.sqlproj` file.

*   File Registration: Every .sql file must have a corresponding <Build Include="..." /> entry within an <ItemGroup> in the project file.

*   Build Action: The BuildAction must always be set to Build.

*   Example .sqlproj Entry
    <ItemGroup>
      <Build Include="dbo\Tables\Customers.sql" />
    </ItemGroup>

### **7. Build & Deployment Requirements**

Copilot must ensure generated code:

*   Compiles under SSDT
*   Avoids unsupported T‑SQL features
*   Doesn’t include `USE <Database>`

***

# **How Copilot Should Behave in This Repo**

✔ Generate high‑quality SQL adhering to project conventions  
✔ Maintain schema consistency  
✔ Suggest performance‑optimized queries  
✔ Avoid anti‑patterns  
✔ Follow naming conventions automatically

❌ Do not create objects without schema  
❌ Do not modify existing database patterns  
❌ Do not generate unsafe or non‑transactional write logic  
❌ Do not use deprecated SQL Server syntax

***