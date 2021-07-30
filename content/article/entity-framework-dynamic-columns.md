---
title: "Reading user defined columns from database with Entity Framework"
date: 2015-03-12T20:43:25+03:00
draft: false
Summary: "Reading user defined columns from database with Entity Framework"
---

Data tables are common part of a user interface. Recall how many times you have implemented table based data representation in your project. In some cases when you need something more than just displaying ID and Name, and this 'something' needs additional runtime calculations (for example amount of money spend in last year, top 10 favorite songs, etc.) you can decide to use a database view that does all calculations for you. In more advanced scenarios you have 100 columns in a database view that represent different UI elements and user has an ability to pick some of them that are important right now. If you implement selective database query, you will have positive performance impact based on reduced amount of data that web server needs to fetch into the user's browser. Moreover, an SQL server can optimize the query execution plan, as you are not asking all the columns from database view. In order to implement this your application should be able to recognize, which columns user would like to see. Most likely it would be a string representation of these columns. If you need to use the Entity Framework, here is how it's done.

Jumping ahead you will be able to use this syntax:

``` csharp
using (var ctx = new MyContext())
{
    return ctx.Customers.SelectProperties(new [] { "LastName", "Phone1" });
}
```

![entity-framework-dynamic-columns-fig1](/images/entity-framework-dynamic-columns/entity-framework-dynamic-columns-fig1.jpg)

<span style="line-height: 20.7999992370605px;">Let's see how to achieve a similar result in a more straightforward way for some particular case. Imagine we have the following Customer table:</span>

<ins>Customer:</ins>

*   First Name
*   Middle Name
*   Last Name
*   Birth Date
*   Phone 1
*   Phone 2
*   Address Line
*   State
*   Post Code

And user would like to see only the Last Name and Phone1\. The C# code will look like this:

``` csharp
using (var ctx = new MyContext())
 {
     return ctx.Customers
         .Select(c => new
             {
                 LastName = c.LastName,
                 Phone1 = c.Phone1
              })
         .ToList();
 }
 ```

Ok. Now we need to solve two problems:

*   Generate "Select" statements for any column combination
*   Return result as a list of Customer type

First two thing google suggests for this are:

*   [Linq Dynamic Query Library](http://weblogs.asp.net/scottgu/dynamic-linq-part-1-using-the-linq-dynamic-query-library)
*   [This answer on StackOverflow](http://stackoverflow.com/questions/606104/how-to-create-linq-expression-tree-to-select-an-anonymous-type)

However, the main problem with these solutions is that they are solving only the first problem stated above. The second problem is a bit tricky because there is no straightforward way to return a mapped entity from the EF context using the "select new" statement. If you try to do something like this:

``` csharp
using (var ctx = new MyContext())
 {
     return ctx.Customers
         .Select(c => new Customer()
             {
                 LastName = c.LastName,
                 Phone1 = c.Phone1
             })
         .ToList();
 }
  ```

you will get the following exception:

``` 
The entity or complex type 'Customer' cannot be constructed in a LINQ to Entities query.
```

and you cannot fix this issue as it's the Entity Framework's feature (EF has to track changes in each instance, but it's hard to imagine for the cases with partly-initialized entities). Even if you create a class that is inherited from the mapped class and try to use it in a similar way, you will fail. But this should not stop you if you know what you are doing and for what purpose. The only way out is to create a projection to a type whicht is not mapped (for example anonymous type), then select a number of such instances from a database and [use them to create the new instances of the required type in memory using LINQ](https://social.msdn.microsoft.com/Forums/en-US/cdd1e6b3-11e3-4341-ae90-9f6093010c0c/the-entity-or-complex-type-categories-cannot-be-constructed-in-a-linq-to-entities-query?forum=adodotnetentityframework).

Taking approach from [answer on StackOverflow](http://stackoverflow.com/questions/606104/how-to-create-linq-expression-tree-to-select-an-anonymous-type) as a basis and expanding it to fit our particular goal, we can do the following:

First let's agree on a method signature. It should take IQueryable<T> to allow you to pass any pre-condition for the query. It should also request the properties that needs to be selected from the database.

``` csharp
public static ICollection<T> SelectProperties<T>(
     this IQueryable<T> source, 
     IEnumerable<string> selectedProperties) where T: class
```

Then you need to do the following:

1.  Make sure all properties passed as arguments actually exist in T type.
2.  Create a Runtime type that will contain set of selected fields (these fields' types must be equal to the corresponding properties from the T type)
3.  Create mapping from T type properties to the Runtime type fields
4.  Create Lambda Expression with a projection from T type to the Runtime type
5.  Invoke the Select method using that Lambda Expression

Let's do this!

``` csharp
// Take properties from the mapped entitiy that match selected properties
 IDictionary<string, PropertyInfo> sourceProperties = 
 	GetTypeProperties<T>(selectedProperties);

 // Construct runtime type by given property configuration
 Type runtimeType = RuntimeTypeBuilder.GetRuntimeType(sourceProperties);
 Type sourceType = typeof (T);

 // Create instance of source parameter
 ParameterExpression sourceParameter = Expression.Parameter(sourceType, "t");

 // Take fields from generated runtime type
 FieldInfo[] runtimeTypeFields = runtimeType.GetFields();

 // Generate bindings from source type to runtime type
 IEnumerable<MemberBinding> bindingsToRuntimeType = runtimeTypeFields
 	.Select(field => Expression.Bind(
 		field, 
 		Expression.Property(
 			sourceParameter, 
 			sourceProperties[field.Name]
 		)
 	));

 // Generate projection trom T to runtimeType and cast as IQueryable<object>
 IQueryable<object> runtimeTypeSelectExpressionQuery 
 	= GetTypeSelectExpressionQuery<object>(
 		sourceType, 
 		runtimeType,
 		bindingsToRuntimeType, 
 		source, 
 		sourceParameter
 	);

 // Get result from database
 List<object> listOfObjects = runtimeTypeSelectExpressionQuery.ToList();
```

**GetTypeProperties** takes all properties from T type and returns only those, that match the selected properties.

``` csharp
 private static IDictionary<string, PropertyInfo> GetTypeProperties<T>(
             IEnumerable<string> SelectedProperties) where T:class
         {
             var existedProperties = typeof (T)
                 .GetProperties()
                 .ToDictionary(p => p.Name);

             return SelectedProperties
                 .Where(existedProperties.ContainsKey)
                 .ToDictionary(p => p, p => existedProperties[p]);
         }
```

**RuntimeTypeBuilder** is a singleton that holds each generated runtime type in a dictionary for caching purposes. The runtime type generation is done in the following way:

``` csharp
var assemblyName = new AssemblyName {Name = "DynamicLinqTypes"};

 moduleBuilder = Thread.GetDomain()
     .DefineDynamicAssembly(assemblyName, AssemblyBuilderAccess.Run)
     .DefineDynamicModule(assemblyName.Name);

 string typeName = fields.Aggregate(string.Empty, (current, field) => 
     current + (field.Key + ";" + field.Value.Name + ";"));

 var typeBuilder = moduleBuilder.DefineType(
     typeName, 
     TypeAttributes.Public | TypeAttributes.Class | TypeAttributes.Serializable
 );

 foreach (var property in properties)
 {
     typeBuilder.DefineField(
         property.Key, 
         property.Value.PropertyType, 
         FieldAttributes.Public
     );
 }

 return typeBuilder.CreateType();
 ```

**GetTypeSelectExpressionQuery** is a sequence of methods that generates Lambda Expression, Select statement over it and invokes it:

``` csharp
private static IQueryable<TT> GetTypeSelectExpressionQuery<TT>(
             Type sourceType, 
             Type targetType, 
             IEnumerable<MemberBinding> binding, 
             IQueryable source,
             ParameterExpression sourceParameter)
         {
             LambdaExpression typeSelector =
                 Expression.Lambda(
                     Expression.MemberInit(
                         Expression.New(
                             targetType.GetConstructor(Type.EmptyTypes)
                         ),
                         binding
                     ),
                     sourceParameter
                 );

             MethodCallExpression typeSelectExpression = 
                 Expression.Call(
                     typeof(Queryable),
                     "Select",
                     new[] { sourceType, targetType },
                     Expression.Constant(source),
                     typeSelector
                 );

             return Expression.Lambda(typeSelectExpression)
                 .Compile()
                 .DynamicInvoke() as IQueryable<TT>;
         }
```

Great! We are 50% finished! Now we need to do the following:

1.  Convert List<object> (that is actually List<runtimeType>) to IQueryable
2.  Do all steps above in a reverse order to get List<T>

Converting to IQueryable:

``` csharp
     MethodInfo castMethod = typeof(Queryable)
         .GetMethod("Cast", BindingFlags.Public | BindingFlags.Static)
         .MakeGenericMethod(runtimeType);

     // Cast list<objects> to IQueryable<runtimeType>
     IQueryable castedSource = castMethod.Invoke(
         null, 
         new Object[] { listOfObjects.AsQueryable() }
     ) as IQueryable;
```

and the rest of the job:

``` csharp
     // Create instance of runtime type parameter
     ParameterExpression runtimeParameter = Expression.Parameter(runtimeType, "p");

     IDictionary<string, FieldInfo> dynamicTypeFieldsDict = 
         runtimeTypeFields.ToDictionary(f => f.Name, f => f);

     // Generate bindings from runtime type to source type
     IEnumerable<MemberBinding> bindingsToTargetType = sourceProperties.Values
         .Select(property => Expression.Bind(
             property, 
             Expression.Field(
                 runtimeParameter, 
                 dynamicTypeFieldsDict[property.Name]
             )
         ));

     // Generate projection trom runtimeType to T and cast as IQueryable<object>
     IQueryable<T> targetTypeSelectExpressionQuery 
         = GetTypeSelectExpressionQuery<T>(
             runtimeType,
             sourceType, 
             bindingsToTargetType, 
             castedSource, 
             runtimeParameter
     );

     // Return list of T
     return targetTypeSelectExpressionQuery.ToList();
```

That's it!

And here is a [working solution in GIT repository](https://github.com/lAnubisl/entity-framework-dynamic-queries). Enjoy!