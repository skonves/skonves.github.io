# Correlation IDs in Practice

Logging is hard.

Ensuring that you have useful data being logged at the correct severity level *before* systems
start going down at 3am is hard enough.  Logging too much data can often increase the size of the
haystack without adding any proverbial needles.  A Microservices architecture can exacerbate
the issue by breaking up events between systems.

If your Orders service throws when getting inventory, it can be non-trivial to find
the corresponding message logged by the Inventory service.  And good luck discovering that the
problem didn't even originate with the Inventory serivce, but was from a temporary outage of the
Warehouse service.

In chapter 8 of his book [Building Microservices](https://www.amazon.com/Building-Microservices-Sam-Newman/dp/1491950358),
Sam Newman describes the concept of the **correlation ID**, a value than can be used to identify
all messages that originated from the same event. By passing these IDs between systems and including
them in log messages, discovering the circumstances surrounding errors gets a lot more sane.

## How we do it

In our implementations, the overall process for using correlation IDs looks like this:

1. Get or create an ID at the start of the event
2. Store ID in the event context
3. Pass ID to all downstream events
4. If relevant, pass the ID back upstream when the event completes

### Get an ID
In a website or web service scenario, individual web requests are events.  All log messages generated
during a request share a correlation ID.  In the case of a console app or Windows service,
an event might start when a message is pulled down off of a message queue to be processed or when a
database it queried.  In the case of the dequeued message, all messages logged while processing that
message would share a correlation ID.  In the case of the database query, the same correlation ID
could be used for *all* rows that are returned, or alternatively, one could be generated for *each* row.

The key thing to key in mind is to use the same ID for things that will be affected by the same failure.

For web services, we look for a `x-correlation-id` header on the request.  If that header isn't present,
we create a random GUID instead.  Expressjs make this easy by allowing custom middleware.  We built one
that gets the request header or create a new `uuid` if it's missing and then also adds that ID as a
header on the response object at the same time (see step 4).

It's pretty dead simple:

``` Javascript
function middleware(req, res, next) {
	var correlationId = req.get('X-Correlation-Id') || uuid.v4();
	res.set('X-Correlation-Id', correlationId);
	next();
}
```

Various message queue systems allow for adding custom properties onto a message.  The system that enqueues
the message tacks on a `CorrelationId` property.  When the message is dequeued, if a correlation ID
property is present it used, otherwise, a random GUID is used instead.

### Store an ID