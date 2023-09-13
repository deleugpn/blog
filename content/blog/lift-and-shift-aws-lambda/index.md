---
title: "3 years of lift-and-shift into AWS Lambda"
date: "2022-08-14T18:55:37.719Z"
description: What to watch out for when moving your existing project into AWS Lambda. 
---

Let's set the scene. We're looking for scaling a PHP 
application. Googling around take us to find out that
AWS Lambda is the most scalable service out there. It doesn't
support PHP natively, but we got https://bref.sh. Not only
that, we also have [Serverless Visually Explained](https://serverless-visually-explained.com/)
which walk us through what we need to know to get PHP up
and running on AWS Lambda. But we have a 8 year old
project that was not designed from the ground up to be
serverless. It's not legacy. Not really. It works well,
has some decent test coverage, a handful of engineers
working on it and it's been a success so far. It just has
not been designed for horizontal scaling. What now?

Bref has two primary goals: Feature parity with other
AWS Lambda runtime (the Event-Driven layer) and also
to be a replacement for web hosting (the Web Apps layer).
In a way, with the Web App layer, Bref allows us to lift-and-shift
from our current hosting provider into AWS Lambda and have
PHP-FPM working the same way as we're all used to love.
So if we just take a large codebase and redeploy it on
AWS Lambda, will everything just work?

Here are some caveats to pay close attention to help with
this journey.

#### 30 seconds API Gateway Timeout

When deploying to AWS Lambda, it's very common to use API
Gateway as the routing solution. It's like an nginx for
our PHP-FPM, but completely scalable and managed by AWS.
It does come with a hard limit of timeout at 30 seconds.
If your application never takes longer to process any
HTTP request, then this is not a big deal, but if it does,
even if very rarely, API Gateway will kill the request.

If this is a deal-breaker for your application, a workaround
could be to use Application Load Balancer instead.

#### PHP Extensions

Not every PHP extension is available easily. [Tobias Nyholm](https://twitter.com/TobiasNyholm)
maintains a great community project for PHP Extensions at
https://github.com/brefphp/extra-php-extensions. A lot of
great extensions are available by default 
(as documented in https://bref.sh/docs/environment/php.html#extensions),
but if you need an extension that is not available by default
or not provided by Tobias, you'll either have to build an
AWS Lambda layer yourself or you'll have to find a way 
without that extension.

#### Incoming Payload limit

When someone sends a HTTP request to your application,
they're usually sending in some raw data with it, typically
on the body of the request. If the body of the request
goes above AWS limit, your code will never even be executed
and AWS will kill the request upfront. The limits
are currently as follows:

- 1MB for Application Load Balancer
- 10MB for API Gateway

A very big portion of use cases can fit in both of these 
offerings. The most common use case that may pose a threat
is file uploads. If the file is bigger than the allowed
size, AWS will not accept the request. A common workaround
is to refactor the application so that the backend returns
an S3 Signed Url for the frontend to do the file upload
directly to S3 and then notify the backend once it's done.
Without going into complex use-case, S3 can take 100MB
of upload in a plain and simple request, but it also
support multi-part upload with a much bigger limit.

#### HTTP Response Limit

AWS Lambda has a limit of 1MB for the Http Response. When
I started getting 502 Gateway Timeout due to large response
size, the workaround that worked best for me was to gzip
the Http Response. That brought down the response size
from 1MB to roughly 50~100kb. I never had to work on
a use-case where gzipped responses would be bigger than
1MB, so if you have such a case, let me know on Twitter
because I'm very interested in it!

#### AWS Lambda execution limit

AWS Lambda is capped at 15 minutes of execution. If you
use API Gateway, that's capped at 30 seconds anyway, but
if you use Application Load Balancer, then it's possible to
have Http requests going up to 15 minutes. Although that's
not very common / likely to be useful, one thing that
does come up is background jobs. Sometimes they can go
above 15 minutes. One use case I worked on was importing
CSV file. The Http layer would return a Signed Url, frontend
would upload to S3 and notify the backend of a new file
ready for processing. A message for SQS would be produced
and a new AWS Lambda would pick up the message as the worker.
Downloading the CSV file from S3 and processing each row
individually could take more than 15 minutes. The way
I handled that was to create a recursive Job. What I did
was:

- Download the file from S3
- Process 1k rows on the file (for loop)
- Each row processed increments a record on the database
- Produce a new SQS message IDENTICAL to the one currently being worked on
- Finish the job successfully

With this approach, the SQS message that gets picked up will
finish before 15 minutes and it will produce a new SQS message
with the exact same work unit. The job can always load
the pointer from the database to know where it left off
(the starting point of the for loop). If we don't have 1k
records to run through, we can stop producing SQS messages
because we reached the end of the file. If there are
still rows to be processed, the new IDENTICAL SQS message
will make a new Lambda start, which means another 15 minutes
limit.

#### 5 Layers per function

If your project uses too many exotic PHP extensions, you
may end up hitting the limit of 5 layers per function.
Bref itself will consume 2 of 5 layers. Each extension
is a separate layer. If you need, e.g., GMP for verifying
JWT tokens, Imagick for image processing and Redis, you've
reached your limit and the next extension you need might
be problematic.

#### 50 MB of zipped Codebase / 250 MB of unzipped codebase

Your project source code cannot have more than 50MB of
zipped content or 250MB of unzipped. The 250MB unzipped also
include the Lambda layers extracted into your Lambda environment.
It sounds like a lot of code, but there are some PHP packages
out there that put binaries on your vendor folder which
may take a lot of space. There are some workaround
documented by Bref which involves zipping your vendor,
uploading it to S3 and deploying your code without
the vendor folder. The vendor then gets downloaded
on-the-fly when your Lambda starts. It adds a little
bit of overhead on your cold-start but overcome AWS
limitation on the codebase size.

#### read-only file-system

The entire file-system on AWS Lambda is read-only with
the exception of `/tmp`. Any process that is writing
files into disk will need tweaking to work with the temporary
folder. Furthermore, two different requests will not be
guarantee to hit the exact same Lambda, so relying
on local disk is not a safe bet. Any data that
may need to be accessed in the future must go to S3
or a durable storage.

#### 500MB on `/tmp`

Another thing to keep in mind is that `/tmp` only has
500MB of disk space. If you fill that up, the Lambda environment
will start to get errors for full disk.

#### SQS messages cannot have more than 256kb of payload

Maybe on your server you use Redis as the Queue Storage
for your background jobs. If your serialized job class
has more than 256kb, it will not fit into AWS SQS.
A common workaround is to pass to the job a reference
to something big instead of the big thing itself.
For instance, if you have a job class that takes a collection
of 1 million rows of a database, that can be mitigated
by passing the instructions to query the database instead.
An SQL Query would be extremely small and produce a somewhat
similar result.

#### Every-minute CRON

When you have a server, putting a CRON to run every
minute and doing some clean up or tidy-up tasks sounds
trivial. But since AWS Lambda charges per code execution,
having a CRON running every minute can add up a bill
and/or be an unwanted side-effect. AWS offers
Scheduler for AWS Lambda, which means code can still
run on a regular basis, but it's important to keep in
mind that everytime Lambda starts, it adds up to your
bill. Things like cleaning up old/stale files might not
even be necessary since every request may hit a clean
AWS Lambda environment.

### Conclusion

This may not be an extensive list of gotchas, but I tried
to include everything that I may have faced while working
with AWS Lambda for the past 3 years. A lot of these limitations
have acceptable workarounds and overall hosting a Laravel
application on a single fat AWS Lambda has brought more
benefits than problems, so I would still advise in favour
of it if you're looking to scale and ditch server maintenance.

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.