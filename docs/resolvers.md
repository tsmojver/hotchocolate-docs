---
id: resolvers
title: Resolvers
---

In GraphQL every field has it's own resolver logic that can run independently from the other fields.

This allows us to build a powerfull schema that consists of multiple data sources in a very simple way.

## Schema-First

In Hot Chocolate we have multiple approaches to write resolvers depending on how you declare your schema.

With the schema-first approach the simplest approach of writing a resolver is binding a delegate that resolves the data to a field in your schema.

```CSharp
c.BindResolver(ctx =>
{
    // my resolver code goes here ...
}).To("Query", "foo");
```

Furthermore, I could bind a class as a resolver and bind specific methods of this call to fields in my schema

```CSharp
services.AddGraphQL(
    File.ReadAllText("schema.graphql"),
    sp => Schema.Create(c =>
{
    c.BindResolver<Query>()
        .To("Query")
        .Resolve("greetings")
        .With(t => t.GetGreetings());
}));
```

Since, the class `Query` is used as a resolver type we will create this instance as a singleton and keep the instance until for the lifetime of the schema unless the type `Query` is registered with your dependency injection provider, in which case the dependency injection container is responsible for the lifetime.

Sometimes, we do not want to explicitly declare resolvers. Sometimes, we have already modeled our entities very well and just want to map those to our schema. In this case we can just bind our type like the following:

```CSharp
services.AddGraphQL(
    File.ReadAllText("schema.graphql"),
    sp => Schema.Create(c =>
{
    c.BindType<Query>().To("Query");
}));
```

Hot Chocolate will generate an in-memory assembly with resolver code for the bound type. Moreover, we can combine our approach in order to provide specific resolver logic for our entity or in order to extend on our entity. In many cases our entity may just represent part of the data structure that we want to expose. In this case we can just provide resolver logic to fill the gaps.

```CSharp
services.AddGraphQL(
    File.ReadAllText("schema.graphql"),
    sp => Schema.Create(c =>
{
    c.BindType<Query>().To("Query");

    c.BindResolver<QueryResolver>()
        .To("Query")
        .Resolve("greetings")
        .With(t => t.GetGreetings(default));
}));
```

In the above case the `GetGreetings` function can have an argument `Query` which is our bound entity. Resolver methods can specify the original field arguments as specified by the schema and context arguments. In our case we could just specify that we want the object that represents the instance of the declaring type of our field that we are bout to resolve.

```CSharp
public string GetGreetings([Parent]Query query) => query.Greetings;
```

The `ParentAttribute` tells the query engine that this argument shall be the instance of the declaring type of our field.

We also could let the query engine inject us the resolver context which provides us with all the context data for our resolver. For example could we access all the previous resolved object in our path by accessing `IResolverContext.Source`. Or, we could access scoped context data that was passed down by one of the resolvers in my path.

In order to keep our resolver clean and easy to test we can also just let the query engine inject the parts of the resolver context that we need for our resolver like the `ObjectType` to which our current field belongs.

```CSharp
public string GetGreetings(ObjectType type) => type.Name;
```

## Code-First

Code-first is the second approach that can be used to describe a GraphQL schema. In code first field definition and resolver logic are more closely bound.

```CSharp
public class QueryType
    : ObjectType
{
    protected override void Configure(IObjectTypeDescriptor descriptor)
    {
        descriptor.Name("Query");
        descriptor.Field("greetings").Resolver(ctx => "foo");
    }
}
```

The above example declares a type named `Query` with one field called `greetings` of the type `String` that always returns `foo`. Like with the schema-first approach I can create types that are not explicitly bound to a specific .NET type like in the above example.

```CSharp
public class PersonType
    : ObjectType<Person>
{
    protected override void Configure(IObjectTypeDescriptor<Person> descriptor)
    {
    }
}
```

If we bind our type to a specific payload type then, we will by default infer the possible type structure and it`s resolvers from the .NET type. We can always overwrite the defaults or define everything explicitly.

```CSharp
public class PersonType
    : ObjectType<Person>
{
    protected override void Configure(IObjectTypeDescriptor<Person> descriptor)
    {
        descriptor.Name("Person123");
        descriptor.Field(t => t.Name).Type<NonNullType<StringType>>();
        descriptor.Field(t => t.FriendId)
            .Name("friend")
            .Resolver(ctx => ctx.Service<IRepository>().GetPerson(ctx.Parent<Person>().FriendId));
    }
}
```

## Resolver Types

Since, a lot of resolver logic like the one in the above example can the code difficult to test and difficult to read, resolvers can also be included from classes.

```CSharp
descriptor.Field<PersonResolvers>(t => t.GetFriend(defaults)).Type<PersonType>();
```

Furthermore, I can also include all fields of a resolver type implicitly like the following:

```CSharp
descriptor.Include<Query>();
```

We can also reverse the relationship between the type and it`s resolvers. We can include and declare resolvers explicitly in the type, but we can also declare a class that shall act as a resolver type for one or more types.

```CSharp
[GraphQLResolverOf(typeof(Person))]
[GraphQLResolverOf("Query")]
public class SomeResolvers
{
    public Person GetFriend([Parent]Person person)
    {
        // resolver code
    }

    [GraphQLDescription("This field does ...")]
    public string GetGreetings([Parent]Query person, string name)
    {
        // resolver code
    }
}
```

The above example class `SomeResolvers` provides resolvers for multiple types. The types can be declared with the `GraphQLResolverOfAttribute` either by providing the payload .NET type or by providing the schema type name.

The schema builder will associate the various resolver methods with the correct schema fields and types by analything the method parameters. We are providing a couple of attributes that can be used to give the resolver method more context like the return type or the description and so on.


## Dependency Injection



## Resolver Context

The resolver context represents the execution context for a specific field that is being resolved.

| Member        | Type | Description |
| ------------- | ----------- | ----------- |
| `Schema` | `ISchema` | The GraphQL schema. |
| `ObjectType` | `ObjectType` | The object type on which the field resolver is being executed. |
| `Field` | `ObjectField` | The field on which the field resolver is being executed. |
| `QueryDocument` | `DocumentNode` | The query that is being executed. |
| `Operation` | `OperationDefinitionNode` | The operation from the query that is being executed. |
| `FieldSelection` | `FieldNode` | The field selection for which a field resolver is being executed. |
| `Source` | `ImmutableStack<object>` | The source stack contains all previous resolver results of the current execution path |
| `Path` | `Path` | The current execution path. |
| `Parent<T>()` | `T` | Gets the previous (parent) resolver result. |
| `Argument<T>(string name)` | `T` | Gets a specific field argument. |
| `Service<T>()` | `T` | Gets as specific service from the dependency injection container. |
| `CustomContext<T>()` | `T` | Gets a specific custom context object that can be used to build up a state. |
| `DataLoader<T>(string key)` | `T` | Gets a specific DataLoader. |
