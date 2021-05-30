---
title: What it means to run a monolith on AWS Lambda
date: "2021-05-29T16:31:14.737Z"
description: A stable software development model combined with the elasticity of serverless solutions.
---

Let's talk about the ~~elephant~~ monolith in the room. It's easy
to associate monoliths with legacy, weird, outdated, insecure code.
A beast developed for decades that nobody wants to touch because
they'll break it and it is impossible to wrap your head around
everything. Today I want to talk about monolith, but not exactly this kind.
Maybe what I mean by monolith can also be interpreted as a Macroservice.
A single source code repository worked by a team of more than one
person that aggregate many business features sometimes not related
to one another and deployed as a single artifact. Something that
10 years from now may be seen as a legacy monstrosity by a different
team in a different era, but that today is a live system
and (pardon my biased) well maintained.

> This post will detail my personal view of many things in
> the software development process. I am in no way looking
> to convince anyone of anything. This is simply how I've been
> working for the past 2 years. Feel free to agree, disagree,
> learn or abhor anything and everything in chunks or bulks.

#### But, Why?

Simplicity. I can't put it into better words. I'm looking for
the most efficient way for a complete journey of my job: 
understand, implement, test, release and guarantee continuity
of a feature. For the last 50 years the industry gathered
a lot of knowledge on how to do all of these things and serverless
is the most recent paradigm shift. It mainly targets the 
"guarantee continuity" step of a software-based company. But
that's where the biggest problem in running a software for many
years lies, and it often seems intuitive that if we change all
of our development process to fit into serverless (namely AWS Lambda)
then we're addressing the big problem. Professionals that have been
long enough in this profession knows that no new solution comes 
without its own set of problems and AWS can be a source of an
uncountable number of problems. If you're not careful, you're
suddenly spending all of your time fixing release pipelines,
orchestrating multiple containers for local development,
fighting unreliable automation tests, building resiliency against
timeouts and navigating a distributed system that is just as 
impossible to maintain as the legacy monolith but for a whole
new set of reason. The ability to utilize 50 years of monolithic
development knowledge while using the (arguably) best tool for
guaranteed continuity in the market is why I'm running a macroservice
on AWS Lambda.

#### 1 Team = 1 Service

After struggling a lot with microservices, I spent days analysing
why I was struggling, what was bad and where I went wrong. I don't
remember where, but I do remember reading the term 
"distributed monolith" somewhere, which was the perfect explanation
for all my suffering: multiple microservices coupled to each other
that had all the downsides of monoliths and none of the benefits
of microservices. I came to the conclusion that I love the idea
of microservices when my responsibility ends in one side of the
communication and another team's responsibility starts on the
other side. In simple words: if I have to make an API call from
Service A to Service B, but I'm responsible for both services,
then I'm dealing with latency, retrials, producer bugs, consumer
bugs and all the distributed nature of software. All of this while
sacrificing the ability to write a simple [Feature Test](https://blog.twitter.com/engineering/en_us/topics/insights/2017/the-testing-renaissance.html)
that would otherwise cover the functionality. My conclusion is
that if my team is in a position of maintaining a communication 
between two services, then we will actually be merging those
two services into one and making a function call instead. That way
we can easily write a feature test that guarantees the functionality
without having to orchestrate services locally or in production.

#### Hardware optimization

One positive argument around microservices are it's ability of
running on hardware that better match it's purpose. For instance,
an API Controller has different hardware needs than an SQS Worker.
However, this does not have to be sacrificed when working with
a monolith / macroservice. We may still deploy a subset of our
API endpoints into one AWS Lambda with e.g. 1536MB of RAM while
another subset of API endpoints are in another Lambda with just 
512MB of RAM and to finalize our background worker may still be
another Lambda with yet a different configuration. But for simplicity,
all 3 Lambda functions have the entire codebase loaded. Yes, this
mean that cold start is slightly worst for "no good reason", meaning
that AWS Lambda is downloading your entire source code repository
even though a huge chunk of code will never be executed there.
Personally, I never had to optimize this. I know that I could
invest time into building such chunks of code in a way that
Lambda 1 would only have relevant code for its own endpoints
while Lambda 2 would only have another set of code. And if you're
a large enough organization, you may be forced into doing it because
of the 250MB limit on AWS Lambda. I simply have not reached that
problem yet. Source code also doesn't take a lot of disk space.

The biggest advantage of this mindset is to allow for automation
tests / [Feature Tests](https://blog.twitter.com/engineering/en_us/topics/insights/2017/the-testing-renaissance.html)
to be seen as one unit of code while production infrastructure
still being sharded for better performance.

#### Queuing System / Background Workers

AWS SQS is an amazing service for offloading work that can be
done in the background. And it even abstracts the code necessary
for the worker because you can subscribe Lambda to be triggered
by an SQS message. Over the years I learned to identify two
major types of Queueing systems: asynchronously offloading
a unit of work and delegating a unit of work to another service.
If you have worked with Laravel, you're probably
familiar with the first case. A Job class will represent
the unit of work to be queued and serialized. This is an extremely
coupled scenario where the message contains PHP/Laravel-specific
information. As someone who got introduced to queues in that scenario,
I used to think that queues are only a private resource and that
a project should provide an HTTP API, validate the payload and
then offload it into it's own private queueing system. While
studying for the AWS Developer and DevOps Professional certifications,
I started to run into cloud-native scenarios where SQS is used
as the communication channel between two independent services.
Technically it means that validation of the payload in advance
is not an option, but it's much more scalable and allows for
a more resilient distributed system. The key to a successful
implementation is rather communication and upfront agreement:
Who writes messages, what are the acceptable payload format
and who works those messages. If an invalid message is located,
it can be driven into a Dead Letter Queue. Is it invalid because
of the producer? Then the producer must be fixed. Is the worker
rejecting valid messages? Then the worker must be fixed.

When aligning the idea of 1 Service = 1 Team with this mindset,
queues can be used for semi-decoupled processes. Instead of
writing an SQS Message that delegates to another service, we
can write a message to an internal queue where we're responsible
for handling it ourselves. As long as the business allows for
the fact that this makes the process asynchronous, then we're
able to decouple the producer from the consumer without separating
the codebase. Automation tests would still be able to use some
sort of `sync` driver where the job is simply dispatched synchronously
while the codebase could be seen as slightly separated. The
benefits of decoupled code is also something praised by the last
50 years of software development. In case of team growth, we can
surgically split the codebase into two distinct source code
repository and adjust the payload from an "internal job class"
into a business message. That's when a 2nd team becomes responsible
for a specific part of the project.

#### The development practices

Microservices, nanoservices and truly cloud-native application
brings it's own set of challenges. I'm not against learning
new technology and facing problems in different ways in an
attempt to look for a better outcome. However, there is one
practice that I learned in 11 years of software development
that beats anything else: Automated Feature Testing. This
is the one and only practice that has allowed me to evolve
rapidly, deploy on fridays if I really need to, upgrade PHP,
Laravel, database engine or refactor the entire codebase.
Feature tests are about representing a business need in the
exact way that the user will interact with that feature and
perform assertions on the expected outcome. They may be seen
by some people as Behavior Driven Development or Integration
Testing. However, several articles that I've read in the past
bring attention to the same problem with the existing terminology:
Are integration tests suppose to integrate several classes or
several services? Can behavior tests make use of mocks? These
questions highlight why I like the term Feature Tests. As a team
member responsible for one service, my feature tests will integrate
as many classes as needed within my service. But the process
will resemble a unit test because it will allow me to mock the
edge of another service. For instance, if I'm suppose to write
an SQS message that delegates work to a 2nd team, then my Feature
Test stops at the AWS SDK right before it writes said message.
On the other hand, if I'm writing a message for an internal job
which I'm responsible, then I can swap SQS with a Sync Driver
so that the automation test runs the entire process (end to end).
My assertion is in the result of the job being worked and not
on the ability to produce a request for a unit of work.

Since my responsibility is primarily Backend and DevOps work,
almost all of my Laravel Feature Tests will be about interacting
with the API endpoints I provide. If I ever need to make a
change to how an API is called, I know that's a breaking change.
I also know that if I'm changing the assertions being performend,
then that's a behavioral change. Changing the arrangement / state
of the application before the action is performed can be allowed
provided that we're able to automatically migrate the state
of the database / customers so that the changes are basically
not noticeable by the users. When upgrading external dependencies,
Feature Tests will demonstrate whether these dependencies
have breaking changes or behavioral changes that has to be
mitigated by the code.

On the other side of the spectrum, there are extensive debate on
how to test cloud-native applications. AWS offers DynamoDB for
local development. MinIO can be used as a drop-in replacement
for S3 in a lot of cases. MySQL may be used as an alternative
for Aurora. But the more cloud features we make use, the less
likely we'll be able to replicate them locally. Tools like
[localstack](https://github.com/localstack/localstack) make
a great effort in replicating AWS APIs for local development.
It's truly a tremendous piece of technology. But I work on
a small team of PHP developers with limited understanding
of the entire suite of AWS offerings. And a great deal of our
work can be simplified by a good set of testable APIs. APIs such
as Filesystem, SQS, Database Connection. Laravel's strategy of
swapping the implementation with an environment variable so that
PHPUnit can make assertion about files, messages or database 
changes without worrying about whether it will work or not
in production. It's definitely not a perfect system and it's
common to have a feature "broken" in its first release
because environment variables are neglected; the automation
test is making use of local environment variables while
the deployed version make use of AWS CloudFormation definition.
So if variables are not introduced into the deployment template,
the feature will not work. This type of error is easily identifiable
by interacting with the feature for the first time when launching
it, and it rarely breaks after introducing the proper set of
variables.

#### Public contracts

Part of the work that I produce involves providing public APIs
to customers. These APIs are available behind an HTTP protocol,
which involves a domain. A monolith is not forced to be offered
behind a single domain. Laravel, for instance, can even provide
routes for specific domains. With AWS Lambda, we can have
multiple domains pointing to the same Lambda function. This
setup provides the fragmented perception that allows for a good
customer experience by using purpose-built domain name that
directly map to the area of usage while source code maintenance
is still kept as a single service. It also provides a good
imaginary boundary around the codebase for a potential area
of separation.

#### Deployments

This one is a lost cause. A single source code repository
ties the deployment process into a single unit. It's often
hard - or sometimes even impossible - to decouple things
that are ready for release from things that are not ready
for release if two or more major work is started and in parallel
and both has been merged into the staging branch for QA.
Feature flags and keeping features small and releasable
can mitigate, but the winning score goes to separated services
on this subject. However, it's important to pay attention
to the distributed monolith situation again. I have been
in situations where deployment of several services had to 
be orchestrated in a specific sequence to avoid disruption
or downtime. This to me was a sign of over-engineered architecture
that had the drawback of a monolith with a sprinkle of complication
brought by separated codebase.

#### Onboarding

PHP engineers are not known for it's extensive knowledge on
Cloud Engineering. But I also have this wild guess that
this isn't limited to one language. There are a number of
software engineers that will still be good / great at their
jobs without being interested in infrastructure, be it servers,
load balancers, cloud, serverless or anything in between.
But onboarding engineers with extensive experience in cloud native
applications is not something that be taken lightly. Even though
AWS Lambda is already 7 years old it has drastically changed / evolved
since its inception. It's a new paradigm that is constantly 
growing and adapting. In the meantime, there's this known
and proven development practice of writing tests and shipping
APIs / background workers that can cover from basic webhook
functionality to an enterprise-grade online software solution.
Releasing that behind AWS Lambda not only provides a cheap
"hosting" solution, but it also comes with nullable maintenance
overhead in terms of operating system, scalability and security
patches. With millisecond billing and improved network capabilities
that brought cold start down drastically, AWS Lambda saves a lot
on system management and scalability tuning without affecting
the hiring practices and reducing the pool of professionals
available in the market. 

#### Conclusion

I first started targeting AWS Lambda as a hosting solution for
a monolithic application about 2 years ago and since then I 
adapted a lot my understanding of software development practices
for the current world we live in. I do believe that in the next
10 years Serverless will become more and more mainstream and
all of its weaknesses will be targeted and brought down. In the
meantime we can still make use of its fantastic elasticity
and capabilities by making small tweaks and adjustments to
the software deployment process so that we can combine great
tools and knowledge available for ages while creating an
ecosystem that will easily scale for the needs of the business
that will exist within the next 5~10 years. The next generation
of developers will surely look down at all of the "legacy" I'll
be leaving behind and may or may not like to work on it at all.
But they won't be held back by security updates, major releases
of external dependencies, or a constant stream of bugs that
are hard to diagnose and fix while working on their own
next generation of enterprise solution. If this holds true,
then I will be happy with the legacy monolith I will be leaving
behind.

If you liked anything I said, consider following me on [Twitter](https://twitter.com/deleugyn).
But if you think everything I said is nonsense, then follow me on [Twitter](https://twitter.com/deleugyn).

Cheers.
