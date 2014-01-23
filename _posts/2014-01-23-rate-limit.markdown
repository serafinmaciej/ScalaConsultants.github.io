# Request rate limiting in Lift

A few weeks go I've started coding in a big [Lift][1] project. One of my first tasks was to create simple HTTP API for communication with underlying Akka actors. This one was easy thanks to [RestHelper][2].

Second task was to add request rate limiting for this API. It appears that [Lift][1] despite its many features doesn't have rate limiting plugin. So, I had to roll my own.

## Overview

General idea of how rate limits should be implemented was taken from [GitHub API][3]. 
You can check status of a rate limit in HTTP headers of each response:
### Rate limit not reached:
```bash
jan@navicon ~> curl -i http://127.0.0.1:3457/hello
HTTP/1.1 200 OK
X-RateLimit-Limit: 5
X-RateLimit-Remaining: 4
X-RateLimit-Reset: 1390484684129
```
### Rate limit reached:
```bash
jan@navicon ~> curl -i http://127.0.0.1:3457/hello
HTTP/1.1 403 Forbidden
X-RateLimit-Limit: 5
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1390484708009

{
  "message":"API rate limit exceeded."
}
```

## Code
Whole example can be found in my [gist][5].
### Usage
Just wrap your handler functions with `withRateLimit`. Note that you can choose which parts of your API have rate limits enabled.
```scala
class RestHelperWithRateLimit extends RestHelper with RateLimit {
 
  val rateLimitConfig: RateLimit.Config = RateLimit.Enabled(100, 1 minute)
 
  serve {
    withRateLimit {
      case "hello_limits" :: Nil => OkResponse
    }
  }
  
  serve {
    case "no_limits" :: Nil => OkResponse
  }
 
}
```
### Error response
We had a discussion whether HTTP error response code should be `403 Forbidden` or `429 Too Many Requests` as [RFC 6585][4] suggests. I was the coder so I chose `403`. If you don't like it just override `RateLimit.errorHandler` method in your RestHelper:
```scala
  override def errorHandler(req: Req, config: RateLimit.Enabled, nextReset: Long): () => Box[LiftResponse] = {
    PlainTextResponse("", Nil, 429)
  }
```

Last but not least, this example compiles with Lift 2.6-M2.

[1]: http://liftweb.net/
[2]: http://simply.liftweb.net/index-5.3.html
[3]: http://developer.github.com/v3/#rate-limiting
[4]: http://tools.ietf.org/html/rfc6585#page-3
[5]: https://gist.github.com/whysoserious/c96b0f852b207ae162c7
