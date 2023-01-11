# Command Bar Help #

## Feature Description ##
We are creating a command bar on our Blazor web UI to allow customers to quickly find customers/items/orders and view them or edit them as well as other commands.  This is a feature like the command palette in VS code, but much simpler.

I am storing the various options that will display to a user as they are typing in MongoDb and it will mostly be populated by a `MongoProject<>` class from Eventuous.

This way, when a customer is created, say with the name Chris' Cookies, I can update the `CommandBarDocument` in Mongo with the appropriate commands.

So when the `CustomerEvents.V1.CustomerAdded` event fires, the Projection class handles adding a new instance of the document with the following new options:
* View Customer Chris' Cookies (this would navigate to the view customer page for this customer when selected.)
* Edit Customer Chris' Cookies (this would navigate to the edit customer page for this customer when selected.)
* Find Orders for Chris' Cookies (this would navigate to the order list page filtering to this customer when selected.)


## Code ##
### CommandBarDocument ###
```csharp
public sealed record CommandBarDocument : ProjectedDocument
{
    public CommandBarDocument(string id) : base(id)
    {
    }

    public string TenantId { get; init; } = string.Empty;
    public string AggregateId { get; init; } = string.Empty;
    public string AggregateType { get; init; } = string.Empty;
    public CommandBarOption[] Options { get; init; } = Array.Empty<CommandBarOption>();
}

public sealed record CommandBarOption
{
    public CommandBarOption()
    {
    }

    public CommandBarOption(string name, string value) => (Name, Value) = (name, value);

    public string Name { get; init; } = string.Empty;
    public string Value { get; init; } = string.Empty;
}
```

### CommandBarProjection ###
```csharp
public sealed class CommandBarProjection : MongoProjection<CommandBarDocument>
{
    /// <inheritdoc />
    public CommandBarProjection(IMongoDatabase database, TypeMapper? typeMap = null) : base(database, typeMap)
    {
        On<CustomerEvents.V1.CustomerAdded>(builder =>
            builder.UpdateOne
                .DefaultId()
                .Update((evt, update) => update
                    .Set(d => d.TenantId, evt.TenantId)
                    .Set(d => d.AggregateType, "Customer")
                    .Set(d => d.AggregateId, evt.CustomerId)
                    .PushEach(d => d.Options, new[]
                    {
                        new CommandBarOption($"View Customer - {evt.Name}",
                            $"{Nav.Page.CustomerView}/{evt.CustomerId}"),
                        new CommandBarOption($"Edit Customer - {evt.Name}",
                            $"{Nav.Page.CustomerEdit}/{evt.CustomerId}"),
                        new CommandBarOption($"Find Orders for {evt.Name}",
                            $"{Nav.Page.OrderList}/?cust={evt.CustomerId}")
                    })
                ));
    }
}
```

## Challenge ##

How do I create idempotent updates that handle when a customer is created and then updated?

* On Customer Added as show above, works fine, but it's not idempotent.
* On Customer Edit, I don't know how to update each `CommandBarOption` in an idempotent fashion as 
 I don't think I can do multiple update calls during the handling of one event.
* What would be easiest, is if I had the option of `.ReplaceOne()` in Eventous, but I don't.