# NotCodingStandards
This is meant to be a guide, a way to do things. There are no requirements here, just a possible solution.


# REST API

I recommend using C#

### 1) Controller

Your controller should do two things 1) route to the appropriate section of your core application and 2) return an acceptable response

1) Route to the appropriate section of your core application
This involves matching the request from the client to a controller method and then sending a mediated query or command.

Try to think about your routes from the client's perspective, and remember to try and keep it REST. 
REST means describing your API as though it is a collection of resources being modified. Lets start with an example:
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

This is pretty straightforward. CRUD operations are easy to fit into the idea of a REST API endpoint. But once you venture beyond CRUD operations, things get confusing pretty fast. For example, what if I want to give the user the ability to purchase groceries. Well, in that case, I'll have to make up a resource to accomplish that which is still pretty straightforward. For example:

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

Now I've taken a user action and turned it into a resource. The user wants to purchase groceries, so I've made a `purchase` resource that the client application would post to the controller. The `purchase` object would contain the data I need about the user's purchase. There may or may not be a `purchases` database table, it's possible that the idea of this resource only exists as far as this controller.

Let's suppose we have a store application, and the store has locations, and each location has groceries / aisles / employees / inventory / purchase orders / delivery schedules / etc. Using a RESTful approach, we could model our routes as:
Focusing only on `GET` actions:
* `api/stores` -> Get all the stores
* `api/stores/{storeid}/` -> Get information about one store
* `api/stores/{storeid}/groceries` -> Get all the groceries for that store
* `api/stores/{storeid}/aisles` -> Get all the aisles in the store

Our data here exists in a hierarchy, and we are drilling down that hierarchy to get to nested resources. How would we find out what aisle to go to to look for coffee? 
We could query to get all the stores `api/stores` then for each store we could query all the groceries to find the ones that had coffee:
* `api/stores/1/groceries?category=coffee`
* `api/stores/2/groceries?category=coffee`
* `api/stores/3/groceries?category=coffee`
and then assuming the grocery items had a reference to the aisle they were in, we could get that data from there. That's kind of a pain. Why cant we just have one query instead? You may be tempted to do something like this:
`api/findgroceries/coffee` but that's not very RESTful.

Instead, we can create another hierarchy with the same resources:
* `api/groceries/coffee/stores/`
We have inverted the relationship here as `groceries/{groceryid}/stores` but we're still describing a hierarchy of resources so it's still REST.
