# ACE ADO Cache Engine model generation
T4 scripts to generate model classes compatibile with [ACE ADO Cache Engine](https://github.com/saklis/ado-cache-engine).

## Features
* Generate C# classes based on connected database.
* Declare tables for which you want classes generated.
* Table and column names singularization.
* Configurable culture for names manipulation.
* Automatically detect columns with keys and auto-increment.
* Support for read-only types.
* Partial classes for easy extension.
* Can be used both as model for ACE ADO Cache Engine and as self-standing tool (as long as you still have ACE library for definitions).
* Appends API required by ACE ADO Cache Engine as protected to not clutter API of objects.

## Installation
With Visual Studio NuGet Package Manager: `PM> Install-Package AdoCacheEngineModelGeneration`

or download library from https://www.nuget.org/packages/AdoCacheEngineModelGeneration/

## Getting started with model generation
The solution consists of three files:
* DbConfig.tt - Holds configuration variables. It's also an entry point for generation process.
* TableTemplate.ttinclude - Defines a template that is used to generate C# classes.
* FileManager.ttinclude - [Multiple outputs from T4 made easy – revisited](https://damieng.com/blog/2009/11/06/multiple-outputs-from-t4-made-easy-revisited) by [Damien Guard](https://github.com/damieng).

To start using the generator all you need to care about is the first one - DbConfig.tt.
Everything you need to modify is the content of SqlConnectionStringBuilder initializer and array of tables for which you want classes generated.

For database connection, you need to edit SqlConnectionStringBuilder
```c#
string connectionString = new SqlConnectionStringBuilder {
    DataSource = "server_address",
    InitialCatalog = "database_name",
    //IntegratedSecurity = true
    UserID = "sql_login",
    Password = "P@ssw00rd"
}.ToString();
```

Just edit Data source and initial catalog you want connect to and give the script some account to be used. Mind, that this account will only be used during classes generation process and any traces of it will not be noticable in output prepared model. Script will not create any connection definition, too.

Second configuration activity you need to do is to provide a list of tables that you want model generated for.
```c#
string[] tableNames = new[] { "tableName_1", "tableName_2", "tableName_3" };
```

If your table name is the same as reserved word of SQL language (which it really shouldn't be, but accidents happen...) you need to enclose the name in `[]` signs. Example of generating model for table named `User`:
```c#
string[] tableNames = new[] { "Orders", "[User]", "Account" };
```

And that's it. Just save your changes and Visual Studio should ask you to run the script. Do so and check the results.

### Additional configuration
There are few extra configuration options available in DbConfig.tt file.

The `usings` array allows you to add some additional usings directives you may need.
```c#
string[] usings = new string[]{};
```

This may become useful in case you want to include some additional base classes. You can define those by editing `baseClassName` variable.
```c#
string baseClassName= "AdoCacheEntity";
```

The content of this variable will be placed in class declaration just after `:` sign, so you could edit it to your liking by adding extra interfaces you want implemented. You can also remove `AdoCacheEngine` class but keep in mind that if you do the model no longer can be used with ACE ADO Cache Engine and you'll also need to do some additional modifications connected with `IsManagedByCacheEngine` variable (explained below).

Last variable you can edit is `cultureCode` that is used to singularize table names.
```c#
string cultureCode = "en-GB";
```

## Tour around the generated model class.
At first glance generated model class seems complex, but most of the logic hidden inside is actually connected with ACE ADO Cache Engine mechanisms. All of them are also protected, so your editor of choice should keep you protected from them (pun intended).

### Attributes
Model classes make use of many attributes to signal special properties of a column that property is based on. Belowe you'll find a description of those attributes.

* `TableName` - Holds name of the table that is a source for the model. This is attribut of the model class.
* `Key` - This property is (or is part of) Primary Key for the table. ACE ADO Cache Engine requires table to have Primary Key defined.
* `AutoIncrement` - Value in this column is automatically changed by SQL Server as an Identity.
* `ReadOnly` - This is information for ACE ADO Cache Engine that it should not attempt to modify value in this column. This property of some SQL column type, such as `timestamp` or Identity columns and outside of that it should be used very carefully, as Cache Engine can easily fall out of sync with columns marked with this attribute.
* `NewValueField` and `CurrentValueField` - Fields marked with those attributes are used by ACE ADO Cache Engine to handle unsaved changes. If value in those two fields is different the object is considered "dirty" and contains unsaved changes. Each property should have two fields - one marked with `NewValueField` and the other with `CurrentValueField`.

### IsManagedByCacheEngine
`IsManagedByCacheEngine` variable is a flag used by ACE ADO Cache Engine for checking if this particular instance of model class is part of Cache. From model perspective it have impact on the place where new values of each properties are stored. In general, for each property there's a field market with `CurrentValueField` attribute that stores this property current value. This field is also getting set when you use setter for connected property. However if `IsManagedByCacheEngine` is set to true, instead of changing current value, another field marked with `NewValueField` attribute will be set. In that case ACE ADO Cache Engine will update current value field (by usage of `CopyNewValues()` method) for you after those were save in underlying database.

This variable is actually defined in `AdoCacheEntity` class, which is a base class for all ACE ADO Cache Engine model classes. If you decide to not use `AdoCacheEntity` as a base class for your model, you need to changes setter for each property to remove references to `IsManagedByCacheEngine` variable.

### `CopyNewValues()` and `UndoPendingChanges()` methods
Finally, `CopyNewValues()` and `UndoPendingChanges()` methods are used by ACE ADO Cache Engine for updating current values of properties if those were correcty save to database or reverting new values back in case of saving error.

## Third party code
* Multiple outputs from T4 made easy – revisited: <https://damieng.com/blog/2009/11/06/multiple-outputs-from-t4-made-easy-revisited>
