---
layout: post
title: Put your ASP.NET Core controller on a diet with middleware
published: true
description: One of the most important responsibilities of RESTful web services is that they translate business logic outcomes to the correct HTTP response code.
---

One of the most important responsibilities of RESTful web services is that they translate business logic outcomes (and unexpected errors) to the correct HTTP response code. When it comes to accomplishing this in ASP.NET Core, this logic usually ends up in the controller and looks something like this:

``` csharp
public ActionResult Put(int id, [FromForm] string value)
{
    if (string.IsNullOrEmpty(value))
    {
        // don't do this!
        var responseObject = new {
            message = "value is a required parameter"
        };
        string responseBody = JsonConvert
            .SerializeObject(responseObject);
        return BadRequest(responseBody);
    }

    _repo.Update(id, value);

    return Ok();
}
```

Or even worse:

``` csharp
public ActionResult Put(int id, [FromForm] string value)
{
    try
    {
        _repo.Update(id, value);
    }
    catch (ArgumentNullException ex) // ew
    {
        var responseObject = new {
            message = ex.Message;
        };
        string responseBody = JsonConvert
            .SerializeObject(responseObject);
        return BadRequest(responseBody);
    }
    catch (Exception ex) // gross
    {
        return new StatusCodeResult(500);
    }

    return Ok();
}
```

The problem is that these controller methods are exceedingly thick. Every controller method requires input validation and emitting an HTTP response in the event of a validation failure. There are ways to push this validation to other places in the service layer, but the crux of the issue is that validation is in fact business logic which should really be in a domain layer. 

Ideally, we could reduce the controller methods down to something much simpler, like this:

``` csharp
public ActionResult Put(int id, [FromForm] string value)
{
    if (string.IsNullOrEmpty(value))
    {
        throw new BadRequestException("value is required");
    }

    _repo.Update(id, value);

    return Ok();
}
```

Or even better:

``` csharp
public ActionResult Put(int id, [FromForm] string value)
{
    _repo.Update(id, value); //internally throw BadRequestException

    return Ok();
}
```

Where `BadRequestException` is an exception that lives in our domain layer:

``` csharp
public class BadRequestException : Exception
{ }
```

Getting `throw BadRequestException("...")` out of the controller and pushing it down into the domain layer is desirable, because as mentioned before, much of the business logic is captured in validation. And almost everyone can agree business logic shouldn't be in the HTTP layer. The HTTP layer is complicated enough without having to worry about business logic!

> In case you're balking at the idea of using exceptions this way, allow me the opportunity to allay your concern:
>
> You may want this exception to be called `ValidationException` or something less HTTP-oriented, and by all means feel free to do that. I personally think it's fine for the domain layer to have some notion of an HTTP response since requirements are so usually so coupled with HTTP responses. To each their own. However you name it, the key is that we're using exceptions for the validation failure workflow.
>
> You also might be wondering if it's okay to use exceptions for business logic. I personally think this is fine for one big reason: Exceptions are an *immensely* powerful construct for implementing validation logic because they let you write functions as if a validation failure will never happen. They enable you write clean, happy-path function signatures that are blissfully unaware of the possibility that a validation failure may occur. And this is infinitely better than passing validation lists via reference into every function or stuffing your controller with validation logic.

Anyway, back to the point of this post. Since the controller and domain layer won't catch exceptions, then who will? You guessed it -- the answer is [ASP.NET Core middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/). I'm telling you, it's SO comforting knowing middleware will always be there to catch you. Seriously, once you start using this pattern you'll never go back. So let's get to it.

First, we need to create a marker-like interface which will let our middleware discern between exceptions thrown by our business logic and the other, more nefarious exceptions like NullReferenceException, etc:

``` csharp
public interface IHttpException
{
    int HttpStatusCode { get; }
}
```

And then we'll fix up our `BadRequestException` to use that interface:

``` csharp
public class BadRequestException : Exception, IHttpException
{
    public int HttpStatusCode => StatusCodes.Status400BadRequest;
}
```

And, without further adieu, here's the middleware that will handle all uncaught exceptions:

``` csharp
public static IApplicationBuilder UseExceptionHandlingMiddleware
    (this IApplicationBuilder app)
{
    return app.UseExceptionHandler(options => 
    {
        options.Run(async context => {
            var ex = context.Features.Get<IExceptionHandlerFeature>();

            context.Response.ContentType = "application/json";

            string responseMessage;
            if (ex.Error is IHttpException)
            {
                context.Response.StatusCode = 
                    (ex.Error as IHttpException).HttpStatusCode;
                responseMessage = ex.Error.Message;
            }
            else 
            {
                context.Response.StatusCode = 
                    StatusCodes.Status500InternalServerError;
                responseMessage = "internal server error :(";
            }

            var responseObject = new {
                message = responseMessage
            };
            string responseBody = JsonConvert
                .SerializeObject(responseObject);

            await context.Response.WriteAsync(responseBody.ToString());
        });
    });
}
```

> ***Note*** 
>
>Notice how exceptions which don't match the `IHttpException` marker will emit an internal server error. Internal server errors are certainly inconvenient, but we owe it to our service consumers to be transparent when something very bad has happened! This way, we can quickly respond to critical errors in a consistent way (for example, by logging the stack trace, sending a notification to support, reaching out to the user, etc). And it's pretty nice not having try/catches all over the place.

If anything, I hope this post demonstrates the power of ASP.NET Core middleware to handle [cross-cutting concerns](https://en.wikipedia.org/wiki/Cross-cutting_concern) such as exception handling. As a rule of thumb, pretty much any time you have code repeating in many controller methods there's usually a way to move that code into middleware.

Feel free to [clone the source from GitHub](https://github.com/jamejone/http-response-exception-middleware) and see the pattern in action. The unit tests use `Microsoft.AspNetCore.Mvc.Testing` to test the entire HTTP pipeline, including the middleware. Enjoy.