# NotCodingStandards
This is meant to be a guide, a way to do things. There are no requirements here, just a possible solution.


# REST API

I recommend using C#

## Controller

Your controller should do two things 
1. route to the appropriate section of your core application
1. return an acceptable response

#### Route to the appropriate section of your core application
This involves matching the request from the client to a controller method and then sending a mediated query or command.

Try to think about your routes from the client's perspective, and remember to try and keep it REST. 

REST only has 2 rules for URIs that we're concerned with, 
1. your uri should be a hierarchy of resources 
1. your query parameters should not be resources

Lets start with an example:
```
[Route("api/Groceries")]
    [ApiController]
    public class Groceries : ControllerBase
    {
        [HttpGet]
        public ActionResult<IEnumerable<string>> Get()
        {
            // return all the groceries
        }

        [HttpGet("{id}")]
        public ActionResult<string> Get(int id)
        {
            // return the grocery item with this Id
        }

        [HttpPost]
        public void Post([FromBody] string value)
        {
            // add an item to all the groceries
        }

        [HttpPut("{id}")]
        public void Put(int id, [FromBody] string value)
        {
            // update one of the grocery items
        }

        [HttpDelete("{id}")]
        public void Delete(int id)
        {
            // remove a grocery item
        }
    }
```

This is pretty straightforward. CRUD operations are easy to fit into the idea of a REST API endpoint. But once you venture beyond CRUD operations, things get confusing pretty fast. 

For example, what if I want to give the user the ability to purchase groceries. Well, in that case, I'll have to make up a resource to accomplish that. We could create a new resource called a `purchase`

```
[Route("api/Purchases")]
    public class PurchasesController : Controller
    {
        [HttpGet]
        public IEnumerable<string> Get()
        {
            return new string[] { "Groceries I bought", "Groceries I bought" };
        }

        [HttpGet("{id}")]
        public string Get(int id)
        {
            return "One grocery item by ID";
        }

        [HttpPost]
        public void Post([FromBody]string value)
        {
            // HERE ARE ARE BUYING SOMETHING
        }

        [HttpDelete("{id}")]
        public void Delete(int id)
        {
            // HERE I'M CANCELLING A PURCHASE
        }
    }
```

Now I've taken a user action and turned it into a resource. The `purchase` resource would contain the data I need about the user's purchase. There may or may not be a `purchases` database table, it's possible that the idea of this resource only exists as far as this controller.

##### Another example
Let's suppose we have a store application, and the store has locations, and each location has groceries / aisles / employees / inventory / purchase orders / delivery schedules / etc. Using a RESTful approach, we could model our routes as:

* `api/stores` -> All the stores
* `api/stores/{storeid}/` -> One store
* `api/stores/{storeid}/groceries` -> All the groceries for one store
* `api/stores/{storeid}/aisles` -> All the aisles in the store

Our data here exists in a hierarchy, and we are drilling down that hierarchy to get to nested resources. Seems straightforward.

But maybe I need to know what store carries coffee and what aisle I'd find it in. 
We could query to get all the stores `api/stores` then for each store we could query all the groceries to find the ones that had coffee:
* `api/stores/1/groceries?category=coffee`
* `api/stores/2/groceries?category=coffee`
* `api/stores/3/groceries?category=coffee`

Then assuming the grocery items had a reference to the aisle they were in, we could get that data from there. 

That's kind of a pain. Why cant we just have one query instead? 

You may be tempted to do something like this:
`api/findgroceries/coffee` but that's not very RESTful because the uri is not a hierarchy. `findgroceries` is not a resource, it's an action.

Instead, we can create another hierarchy with the same resources:
* `api/groceries/coffee/stores/`
We have inverted the relationship here as `groceries/{groceryid}/stores` but we're still describing a hierarchy of resources so it's still REST.

###### 2) return an appropriate response

We should always return an appropriate http response.
`api/stores/0` if a store with id 0 doesn't exist, we should expect a `404` response
The response codes are fairly self explanatory: https://www.w3.org/Protocols/HTTP/HTRESP.html

What this means though is that you need to be sure you wrap all of your work in a try/catch block inside a controller method. Returning a `500` internal server error because a resource doesn't exist is not an acceptable response. Make sure you're anticipating how the request might fail and how the client should be notified of that failure.

###### 3) (optional, but preferred) call mediator

We definitely don't want business logic in our controllers. Some implementations avoid this by pushing all the business logic into `service` classes that are then called from the controller. This leads to controllers with large constructors and service classes with large constructors and complex dependency graphs.

Another approach is to use mediation. Mediator allows us to bring in one dependency, `Mediator` and then send out a decoupled command / event / query that can be handled somewhere else. 

The benefit there is that our handler can describe its own dependencies and we're much less likely to pull in dependencies that don't get used. 

Another benefit is that we can keep a 1-to-1 mapping of feature to implementation. We don't have to call bloated service classes with possible side effects. If we need another feature that is similar, instead of reusing mediator handlers we can create a new one. This allows us to implement each feature given that feature's specific needs. 

Here is a complete example:
```
[Route("api/Purchases")]
    public class PurchasesController : Controller
    {
        private readonly IMediator _mediator;

        public PurchasesController(IMediator mediator)
        {
            _mediator = mediator;
        }

        [HttpGet]
        public async Task<IActionResult> Get()
        {
            try
            {
                var allPurchases = await _mediator.Request(new GetPurchases());
                return Ok(allPurchases);
            }
            catch(Exception ex)
            {
                return new InternalServerError(ex);
            }
        }



        [HttpGet]
        [Route("{id}")]
        public async Task<IActionResult> Get(string id)
        {
            try
            {
                var purchase = await _mediator.Request(new GetPurchase(id));
                if(purchase == null){
                    return NotFound();
                }
                return Ok(purchase);
            }
            catch (Exception ex)
            {
                return new InternalServerError(ex);
            }
        }
    }
```

You'll notice that we end up doing the same thing each time, wrap everything in a try catch to return an `internal server error` or check the return value of the request for `null` and then return `NotFound` from the controller. These sorts of actions are required to make a feature happen, but aren't part of the acceptance criteria. They're a necessary evil that are a required part of the implementation of the "real feature" which is getting the purchases in this case. We call those sorts of things "cross cutting concerns" and it's usually best to find an available opportunity to abstract them away so we don't need to deal with them. In this case, we could probably use global filters to catch specific exceptions `EntityNotFound` or the general case `Exception` and form the appropriate response to the caller.

-------------------------------------
