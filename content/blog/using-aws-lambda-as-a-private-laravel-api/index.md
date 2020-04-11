---
title: Using AWS Lambda as a private Laravel API
date: "2020-04-11T13:05:19.284Z"
description: A cheap trick to migrate an existing private API 
---

About two years ago I was working on a service that was suppose to
expose some private APIs. These APIs did not require any authentication
process, but were meant to be consumed exclusively by other services
that we own. We could have used some sort of authentication process or
signed keys for this. But at the time, I found Route 53 private DNS
functionality integrated with AWS ECS and went with it. What this meant
was that our services could make API calls to `my-private-service.internal`
and this would resolve to some sort of private IP address (e.g. 10.10.3.27).
Everytime AWS ECS spins up a new container for `my-private-service`, it
automatically updates AWS Route 53 with the container's private IP.
So long the consumers are running on the same VPC and has proper
Security Group, they're able to privately communicate with `my-private-service`
by simply querying `my-private-service.internal` DNS, which meant the
code needed nothing special.

#### Implications of moving a private AWS ECS container to AWS Lambda

Fast-forward 2 years and AWS has improved the AWS Lambda cold start
**drastically**, which means that we no longer need containers running
24/7 to provide such a private API. We can move this to Lambda. At least
I thought it would be that straight-forward. You see,
I didn't remember I had done all this crazy stuff regarding private DNS
and that the service had no authentication mechanism whatsoever. I only
realized that after I had a working prototype running on Lambda function
and it was time to change the DNS to point to my Lambda Function. To my
surprise, the DNS was not behind the Load Balancer which is when I 
realized the issue. To put it behind the Load Balancer, I would need
to develop an authentication mechanism and think about the security
implications of it. The alternative was to change the consumers to
use AWS SDK Lambda Invoke instead of an API call. I ended up implementing
option #2 here. Being invoked by AWS SDK meant that the execution
would require an IAM Role with such permission, which was easy to
grant to the ECS containers consuming the API.

#### The source code structure

While digging through the source code, I noticed that the developer
that had implemented the APIs did it using Laravel Form Request
objects with all of the necessary validations tied to the Http
context. Some people may argue that this is a bad design or that
"a Request object shouldn't know how to validate itself" and that
this bad design lead me to a coupling with the Http Context that 
I couldn't get rid easily. While all this may be true, I still think
it's far more beneficial than damaging. After all, this is a handful of
APIs among thousands of APIs that actually ended up needing to swap
Http context with something else. And in the end, I got creative
and kept the source code pretty much as-is.

#### Lambda Invoke to Http Request

The mindset behind this problem that lead me to this solution was this:
Laravel has an `index.php` file which captures the Request information
from the PHP Superglobals and handles the request through the Http Kernel.
If I write a different "`index.php`" that simulates the exact same behavior,
but instead of extracting the request data from the Superglobals it
extracts from the Lambda Event, everything would continue to work
as-is. Here's the `request.php` that I wrote for Bref:

```php
<?php

use Symfony\Component\HttpKernel\Exception\HttpException;

define('LARAVEL_START', microtime(true));

require __DIR__ . '/../../../../vendor/autoload.php';

/** @var \Illuminate\Foundation\Application $app */
$app = require_once __DIR__ . '/../../../../bootstrap/app.php';

return function (array $event) use ($app) {
    $kernel = $app->make(\Illuminate\Contracts\Http\Kernel::class);

    if (! isset($event['LARAVEL_ROUTE'])) {
        throw new HttpException(404, 'The LARAVEL_ROUTE variable was not specified.');
    }

    if (! isset($event['LARAVEL_ROUTE_METHOD'])) {
        throw new HttpException(404, 'The LARAVEL_ROUTE_METHOD variable was not specified.');
    }

    $response = $kernel->handle(
        $request = \Illuminate\Http\Request::create(
            $event['LARAVEL_ROUTE'],
            $event['LARAVEL_ROUTE_METHOD'],
            $event['LARAVEL_REQUEST_BODY'] ?? []
        )
    );

    $kernel->terminate($request, $response);

    // We don't "send" the response as we need to return it to the Lambda. This means
    // headers will not be sent since this is not an actual HTTP Request. If you
    // need full HTTP support, use the actual `public/index.php` file instead.
    return $response->getOriginalContent();
};
```

With the exception of Headers not being sent because of the execution
model, everything else is pretty similar to what it used to be.
Particularly for this use case, headers were not important anyway.
All we had to do was change the consumers from making an actual
API call to `my-private-service.internal` to using the AWS SDK
to invoke Lambda with a `InvocationType => RequestResponse` and the body of the
request as the invocation `Payload`. We could move the path from
the domain to the `LARAVEL_ROUTE` attribute of the payload and the
HTTP Verb (GET, POST, PUT) to the `LARAVEL_ROUTE_METHOD` attribute.
The body of the request would be inside `LARAVEL_REQUEST_BODY` and
we're done.

#### Conclusion

The reason I thought about writing this down was mostly about the
journey than the actual code. Thinking through the problem and 
optimizing the trade-offs is an important skill. When I started 
working on this project, I didn't expect to hit so many interesting 
challenges and what was even better was being able to overcome said
challenges in such a short time. This entire project took me between
1 and 2 working days from conception to production. Granted, I was 
not tasked with changing any consumer code, I relied on my team 
to do that. I may have broken a few "best practices" along the way,
but we're no longer paying AWS Fargate containers 24/7, all consumers
are happy, the teammates that handled the consumer code thought I
made it easy enough for them, the security aspect of the service
was kept intact and the main technical cost was a custom Lambda
handler with a few lines of code that mostly mimics Laravel's default
`index.php`. Overall, 10/10 satisfaction with this job. 

Hope you enjoyed the reading. If you have any questions,
send them my way on [Twitter](https://twitter.com/deleugyn).

Cheers.