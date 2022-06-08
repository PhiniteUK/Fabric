# Commands
Commands in Fabric represent a command from the CQRS pattern. We have created a set of classes and interfaces that can be used to help streamline creation of commands within your applications.

## Class Structure
We would recommend using a class structure similar to the below to aid with speed of development.

```csharp
public sealed class ExampleCommand
{
    public sealed record Request(Guid Id) : ICommand;
    
    internal sealed class RequestValidator : AbstractValidator<Request>
    {
        public RequestValidator()
        {
            RuleFor(m => m.Id).NotEmpty();
        }
    }
    
    internal sealed class Handler : ICommandHandler<Request>
    {
        public async Task<CommandResult> Handle(Request request, CancellationToken cancellationToken)
        {
            ... Command Code ...
        }
    }
}
```

### Nested Classes
We chose to recommend nested classes as the default pattern since it helps to reduce the amount of renaming required when creating new commands in your project. You can easily copy a command and only need to rename the outer class, leaving the inner classes with the same names (`Request` and `Handler`).

We also found that having all of the classes relating to a single command in one `.cs` file helped reduce the time spent searching for files when working on a single command.

- [ ] 📝 Possibly add note about disabling roslyn analyser

### Validators
We recommend adding validators for a `Request` using `Fluent Validation` as this provides a great separation of concerns and can be automatically validated using a pipeline behaviour before the `Handler` fires.

## ICommand Interface
Each command you create should inherit from the `ICommand` interface. 

The `ICommand` interface reduces the complexity of the code you need to write when creating a command. It provides a way to say that a specific request is a command and removes the need for you to specify the `CommandResult` leading to improved consistency and preventing you from accidentally returning data from a command.

```csharp
public interface ICommand : IRequest<CommandResult>
{ }
```

_The `IRequest` interface comes from the MediatR library_

## ICommandHandler interface
Each command handler you create should inherit from the `ICommandHandler` interface.

The `ICommandHandler` interface reduces the complexity of the code you need to write when creating a command handler. It provides a handler that always returns a `CommandResult` and only works in conjunction with an `ICommand`. 

```csharp
public interface ICommandHandler<in TCommand> : IRequestHandler<TCommand, CommandResult>
    where TCommand : ICommand
{ }
```

_The `IRequestHandler` interface comes from the MediatR library_

## Command Results
Commands have been set up to always return the `CommandResult` object. This forces consistency across all commands as well as making it clear that there can be no data returned from a command.

```csharp
public sealed class CommandResult
{
    public string ErrorMessage => Exception?.Message ?? "An unknown error occurred.";

    public Exception? Exception { get; private set; }

    public Guid? ObjectId { get; private set; }

    public bool Successful { get; private set; }
}
```

### Error Message
We chose to add a helper to prevent the need for writing common code when an error message is always needed, eg. for showing to a user. If no exception with a specific message is available in the `CommandResult` then a default error message is returned (An unknown error occurred.).

### Exception
The exception can be provided to send an error back to the caller to determine the result of the command.

### Object Id
We chose to allow a command to return a single piece of data, the ID of the object that was created. This decision was made to make querying data simpler after initial object creation. Given the common flow of execute a command then load the created object it makes retrieving the ID of the newly created object much simpler.

### Successful
This provides the status of the command (did it succeed or fail).

## Creating Results
To allow creation of results more easily there are a number of static factory methods.

### Error Result
An error result should be returned when something goes wrong within the `ICommandHandler`. The `Successful` property will be set to `false` and the `Exception` is required.

```csharp
CommandResult.Error(Exception exception);
```

### Not Found Result
A not found result should be returned when an entity could not be found within the `ICommandHandler`. The `Successful` property will be set to `false` and an `EntityNotFoundException` will be automatically created using the `objectId` and `entityName`.

```csharp
CommandResult.NotFound(string entityName, Guid objectId);
```

### Success Result
A success result should be returned when an entity could not be found within the `ICommandHandler`. The `Successful` property will be set to `true`.

```csharp
CommandResult.Success();
```

You can also set the `objectId` of the result.

```csharp
CommandResult.Success(Guid objectId);
```
