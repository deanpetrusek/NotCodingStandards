### Get

1) The client calls the API and the request is routed to the controller method
2) The controller method takes the request and send a Mediator request to a Core handler
3) The core handler does all the work and returns an acceptable response
4) the controller method returns the result of the mediator handler back to the client


Example:

```
1)
[HttpGet("{id}")]
public ActionResult<object> Get(int id)
{
    // 2)
    return _mediator
        .Request(new GetValue(id));
    // 4)
}
```


```
3)
using Common.Mediator;

namespace Example.Core
{
    public class GetValueHandler: IQueryHandler<GetValue, object>
    {
        public object Handle(GetValue query)
        {
            // GET Value by ID and return it;
            return new { };
        }
    }
}
```

The goal here is "thin" controllers. Keep the feature work in the `Core` application as much as we can.


---

### Post, Put, Delete

1) The client calls the API and the request is routed to the controller method
2) The controller method takes the request and send an NServiceBus Notify Command to the Server endpoint
3) The NServiceBus Notify Command Handler receives the request and sends a Mediator command 
4) The core handler does all the work

Example:

```
1)
[HttpPost]
public async Task Post([FromBody] string value)
{
    //  2)
    await _bus
        .Send(new NotifyAddValue())
        .ConfigureAwait(false);
}
```

```
3)
public class NotifyAddValueHandler: IHandleMessages<NotifyAddValue>
{
    private readonly IMediator _mediator;

    public NotifyAddValueHandler(
        IMediator mediator)
    {
        _mediator = mediator;
    }

    public async Task Handle(NotifyAddValue message, IMessageHandlerContext context)
    {
        // 3)
        _mediator.Send(new AddValueCommand());
    }
}
```

4)
```
public class AddValueHandler: ICommandHandler<AddValueCommand>
{
    public void Handle(AddValueCommand message)
    {
        // HERE I DO MY WORK FOR THE FEATURE
    }
}
```

The goal here is to do our transactional work on the `Server` thread.
In this flow, the Client calls the API controller and all the API controller does is send an NServiceBus message and then return.
Once the API Controller returns the client can continue, and in the meantime the work is queued up on the `Server` endpoint.

The NServiceBus Server will wrap the work in a transaction, which gives us the ability to rollback if one part of the process fails. 

