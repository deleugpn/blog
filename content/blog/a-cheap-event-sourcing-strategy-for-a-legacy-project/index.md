---
title: A cheap event sourcing strategy for a legacy project
date: "2019-07-17T20:41:30.284Z"
description: How to strangle a project full of technical debt towards microservices
---

One of the best talks I've attended in my life was given at Schiphol Airport
during the AmsterdamPHP Meetup by Jeroen van der Gulik. I left that talk
thinking he had prepared it to speak directly to me. It was also the first
time I've heard the term **strangler pattern**, which in simple terms means
sucking the life out of your legacy monolith by slicing it into autonomous
pieces of microservices. But his talk was about a much bigger problem
than mine and he talked about how Apache Kafka helped him through this
journey.

The legacy project I've been working on is nearly 10 years old and packs
a tremendous amount of technical debt. My small team agreed with me after
some attempts that publishing SNS topics from within the legacy was nearly
impossible as there isn't one reliable place to publish the events. The 
spaghetti code in combination with extremely dynamic execution makes it
overly complex to make a reliable decision of when a domain event happened.

Enter Amazon Aurora with triggers capable of executing Lambda Function.
Database triggers are one of these things that I tried and hated when I 
was in university. It sounds great until you're debugging the effects
of something which you have no idea why it's happening or where it comes
from. What made me more accepting of using triggers was trying to set them
up as a dump event sourcing. I had the idea of putting a dumb and generic
trigger on all of the tables we were interested in getting the data from.
The trigger would be one line: invoke a Lambda and tell which table and
which record id changed (created, updated or deleted). This strategy
allowed for reusing pretty much the same trigger code on all tables of
interest. The Lambda would be the one containing the logic of what data
should be extracted from that invocation. In possession of the record
id, the Lambda could connect back to the database and make a Domain object,
joining to as many tables as necessary to represent the Domain and then
publish an SNS topic about the change that happened.

This strategy means that we can source all of the data from the monolith into
different sets of microservices with different bounded contexts. At some point
one microservice take over the ownership of the business logic related to 
a subset of data, as well as the SNS topic that represents the Domain Object.
Since Lambda functions are extremely cheap and Triggers are executed upon
data change, the combination makes for a cheap event sourcing mechanism.
Before diving into this strategy, my team considered: 

- SNS
- MySQL binlog + Kinesis Stream
- Apache Kafka

The SNS strategy was a problem because of the structure of the legacy project.
The MySQL binlog + Kinesis seemed a bit over-complex because it required
us to maintain a middleware project that would understand and parse the
binlog into usable information. Apache Kafka is something that nobody
on the team have experience, the AWS managed service is extremely expensive and
not available in all regions we would need. Managing it on an EC2 cluster
wasn't a great option because we had just finished a 1-year project
migrating 72 outdated EC2 instances into Fargate. With a small team,
we're avoiding bringing maintenance into our tasks.

After much consideration and discussion on the pros/cons of all available
options, a combination of Aurora triggers with Lambda functions seemed
the best option. 

- Easy to distinguish the events
- The Lambda is the one responsible for building up the domain object, not a trigger
- Near real time synchronization across microservices
- Extremely cheap

I released the first version of this strategy about 2 months ago, with 
coverage of only 3 tables representing 3 objects. The most recent release
covered 7 different domain objects with 32 tables. We have 10% of the
customers using data from microservices being kept in sync by this project.
As coverage grows, the ability for microservices to take over more and
more responsibilities grows while the monolith slowly dies. 

This has been one of the most interesting projects I've worked on and
it's been satisfying to see it shaping the next generation of the company
software. I'm proud of the architecture, simplicity, benefits and costs.
At the same time, I understand this is only useful when dealing with a
impossible-to-work-with monolithic application, as a simple SNS being
published by the original source code as a way of distributing events
is far simpler.

Hope you enjoyed the reading. If you're interested in more of my crazy
ideas, follow me on [Twitter](https://twitter.com/deleugyn).

Cheers.