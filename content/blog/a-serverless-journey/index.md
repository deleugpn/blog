---
title: A serverless journey
date: "2024-03-16T16:37:00.000Z"
---

Serverless has been one of the biggest industry innovation
in my career. I started playing with software in 2009
and I've been employed through this entire microservices era.
Recently I've had some insights that I thought would make a good
post. I hope you enjoy the reading even if you disagree with everything.

### Prologue

I'm going to start from the perspective of a 18 year old joining the software
industry in the 2010's because that's where I fit. Back then, software was
already mainstream and an established business. ATMs, credit cards, online purchase,
online banking, filing tax, ERPs. Software had solidified into society and it was not
going anywhere.

Looking back, I was just a kid during the 2YK bug and the dotcom bubble, but I'm going
to build up from the experience of the aftermath of these events. In the 90's lots of
important technologies were born. MySQL, PHP, Ruby, Java, Javascript, Python, etc. These
tools were later going to be the center of what runs the world of technology. In the 2000's
there was a need to make these tools mainstream. Knowledge needed to be spread and with the
advance of the internet, humanity saw another "gold rush" event. Companies that could build
something useful with software was going to revolutionize the way humans work and they had
a very limited time to do so - hence "rush". Combine that with the fact Software Engineering
had borrowed a lot from well established engineering practices (Mechanical, Civil) and we got
ourselves giant monoliths built from the ground up in ways not intended to be changed. After all,
once a 30-store building is finished, nobody will refactor it to make it just 20-store or 45-store.
When the product is done, it is fully done.

### Software Engineering Transition

The digital world isn't ruled by some of the laws of the physical world. It's brand new and eager 
to make mistakes and learn from them. In it, we actually can make exact and perfect replica 
of things. When we get to my period (2010's), the world has already built a lot of monoliths 
based on engineering practices like "Waterfall". These monoliths weren't built in the Cloud
or for the Cloud because cloud didn't exist either. Not every household had a home-computer.
Smartphones weren't something that nearly every human had one. When companies invested thousands
of dollars buying servers and setting up a server room, developers would assume that the server was
just that, nothing more and nothing less. Of course they would assume the local disk was always
present and it was where files would be stored. AWS S3 was a baby born in 2006 and not an industry
standard yet. Horizontal scaling was a multi-thousand dollar project that required months of 
buying more servers. Depending on what equipments were bought, Vertical Scaling could be a possibility
if that meant buying some more plug-and-play CPU, RAM or Disk.

The 2010's need was divided into 2 major categories: redesign for the cloud and redesign for flexibility.
When servers started to malfunction, companies wanted to make that a service provider problem and not have to
deal with the infrastructure necessary to power, maintain, clean and run servers. In parallel to that,
society was rapidly changing and everyone wanted to start using their smartphone devices with a smaller
screen to perform tasks that previously required a computer with a big screen. When I look back through
these lenses, microservices is the only and most obvious natural next step. They would be responsible for
lowering the risk-factor of rebuilding giant 10+ year old monoliths by delivering small, incremental value 
(agile anyone?) while allowing to take the Cloud for a test-drive.

2010's was the perfect scenario for the rise of Docker and Serverless. Developers would learn about 
the Strangler Pattern and "surgically" separate one small piece of the monolith, write down all of
its system dependencies in a recipe (Dockerfile), making it reproducible, and ship it to a Cloud
Provider. Whether this worked flawlessly or not isn't important yet because this is just a small
piece of the software needed to run the business. Most critical parts are still baked into the
monolith. The world is still constantly changing every day and the business still needs the ability
to adapt, so this is working well enough.

### The Economic Landscape (2010)

Interest rate was pretty low. Money was cheap. This is the decade marked by high investment. Companies
that survived after the 2000's dotcom bubble became billion dollar unicorns. Every investor in 2010
wanted to create the next Google, Amazon, Paypal, YouTube, etc. This economic environment was the cherry
on the cake for projects like Docker. Money were being poured into opportunities for growth.
Profitability wasn't relevant here, which is a very odd thing for the financial market, something well
established for 400 years. Docker had one of the highest growth opportunity business. Every developer
was facing a monolith that needed to be redesigned for the Cloud and partitioned into microservices.
Two major constraints were lifted during this period: availability of money and limitation of servers.

Software designed to run on a single server room had structural issues in moving to containers because
basic assumptions are completely different: the local disk is not permanent, the process restarts on
every deployment, graceful shutdown becomes highly important, anything that requires state has to be hosted
outside and separated (Redis, Database, Queue, etc). Since there were many developers available to break
the monolith apart and a variety of technology available, it was not uncommon for separate teams to develop
using different strategies, different technologies and different programming languages.

### The Present

Money played an important role in the rise of microservices because it created many software development roles
and allowed for well-established companies like AWS and Google to invest heavily in making working with
microservices easier. Docker, AWS ECS, Kubernetes and AWS Lambda are all products of the 2010's decade.
Which brings us to 2020's. The decade started off with COVID and a huge boom in technology. Every company
in the world needed to adjust for remote work. Be it VPN, laptops, internet or whatever, the world became
remote-first and drove more money into the tech industry. Meanwhile, a public health disaster takes place,
creating uncertainty and driving business that cannot survive in the digital world to the ground. Unemployed
people need help from their family, from government, etc. The uncertainty of the moment reduces circulation
of money since it's safer to hold onto what you have.

Tech still seem to be thriving, but within 2 years of COVID and the spread of the vaccine, things are about 
to change drastically. I don't know exactly the cause or if there is even just one cause. But these are
some of the important topics:

- COVID relief may have impacted government tax
- Unemployment reduces economic activity
- US Section 174 comes into effect
- Inflation grows and affects availability of capital
- Russia invades Ukraine and Europe energy cost skyrockets
- Israel at war with Hamas

I can't pinpoint the most relevant fact, but we know that the Tech industry goes through massive layoffs,
causing more unemployment which impacts economy even more. Smaller companies that may have been profitable 
for a long time might have a chance to grab a Google Engineer or a Microsoft Engineer at this point.
A lot of folks will still go unemployed for several months.

At this point, we have companies that used to have several developers using different programming languages
and developing microservices now are faced with big consequences: if the entire Java team is laid off, folks
at the Python team has to take over their microservices development and maintenance. Ironically, the 2000's
monolith created for server rooms that were broken down into microservices for the cloud now lack the manpower
needed to keep them afloat and the next obvious step is to merge microservices back together.

### Open mindedness

I have advocated for Docker, AWS Lambda and Microservices. But I also think it's important to re-evaluate
our choices based on the reality we currently live in. If I were to go back to 2010, I would likely not
do much different in terms of adopting these practices. They were absolutely needed to take us from where
we were to where we are. But I think they served their purposes and now we're facing a different reality.
Development teams are smaller, companies need to be profitable and stop burning through cash and the
overhead created by microservices makes it very hard to manage and improve. Instead of closing our eyes
to the monster we've created, I'd rather be critical about it and wonder if there is a better way.

We're currently living through the result of our work and folks that create technical content are already
reflecting on that. Google has recently published a scientific paper talking about the challenges and
overhead of managing microservices. New businesses are starting from scratch in 2024, building 
a 10k MRR running on a $7/month Digital Ocean droplet, serving 2 million requests a month and asking
why we need serverless. Some companies had to let go their DevOps team and developers are now faced with 
the huge burden that is to orchestrate their microservices.

This thought process drives the conversation to a stressful and predictable path: most folks when thinking
about servers and monoliths will only have the 2000's experience to reflect and compare. As if the world
has been stuck in 2000 and the only options are high complexity server management or 2020 containers and
serverless functions. **This couldn't be further from reality.** If you're still interested, let's talk about
an option that we might be too close to consider.

### The world is not static

Anything that you haven't touched in 5 years is hardly the same today as it was when you last tried it.
Unless it's an abandoned project. Folks that bash PHP because in 2010 they hated it are an example of that.
Things that you abandon don't suddenly freeze in time. Let's run through this journey now.

#### Automation testing

Although automation testing is not a new practice and it wasn't a new practice in 2010, I get the sense
that it solified itself into the industry through the last decade. Let's recap the perspective: 2000's
dotcom bubble and the rush of capitalism towards the digital age. Things had to be built fast and with
little time for research and development. Monoliths are the result of it. Changes are inevitable and
agile practices become mainstream. Making changes and revalidating that they don't break anything is
expensive and tedious. Businesses start to learn that even though it takes longer to develop with
automation testing in the beginning, it certainly empowers faster development later on. In 2010, educational
content around automation testing were lacking to say the least. Nowadays my (biased) opinion is that
it's much easier and common to teach folks starting their career in tech how crucial automation tests are.
They will be what empowers constant changes. Some folks might disagree with this statement because it's not
hard to find projects in 2024 without any automation testing whatsoever. But bear in mind that I'm looking
at a vast pool of projects here. Compared to 2000's I truly believe we have an order of magnitude more
automation testing.


#### Predictability

We've learned that software projects are not like buildings and they require constant change and adjustments.
Something that I think mark the learnings from 2000's is the regular cadence of software updates. Back then,
releases were made according to what each individual company could manage and this is reflected in Office 2003,
Office 2007, etc. Programming tools were no different. PHP, MySQL, Java, etc, didn't have the most predictable
release cadence. But it seems the software industry learned that it's best to set a hard date and keep the
schedule at all costs. Because changes to library code and programming tools affect the entire industry
and everyone needs time to adjust to it. PHP has an yearly release cycle and so does Laravel and Symfony.
Laravel actually adjusted its release cycle in order to match the one from PHP and this is only possible
because PHP has a predictable release.

Semantic Versioning, bug and security fixes, major verions and breaking changes are all well understood
practices and play a major role in maintaining the stability of the software industry.

#### Performance

Hardware has greatly improved and became cheaper since the 2000's. One important piece of history that
I like to look back are smartphones. They started to become widespread in the 2010's and they had no
backward compatibility requirements. Software that ran on computers wouldn't run on smartphones. They 
would need specific implementations. They were also very limited in size, so efficient battery consumption
and CPU were of utmost importance. This is where ARM microchips come to life. CPU architecture could
be implemented breaking backward compatibility because there was no past to be compatible with. Mobile
software were brand new. ARM advanced so much that by 2018 AWS come up with Graviton microchips
and by 2020 Apple came out with Mac M1. These two combined made a huge turning point in the software industry.
Projects running in production in the cloud needed to be compatible with ARM in order to benefit from Graviton
and software engineers working on Macbooks needed software compatible with ARM in order to, well, develop it.
This forced PHP, MySQL, Docker, Node and anything tech-related to work towards ARM compatibility.
This advancement helped make applications more efficient and performant, reducing energy consumption
and cost overall.

#### Servers

The big one, where things all come together. Serverless is now about 10 years old. It's a well established
practice and has reached high potentials. I don't know if we will ever see huge innovations and advancements
on serverless, but as it stands now it's a great piece of technology and with high volume of content.
But folks that have been working exclusively with serverless for years might be blinded by how the
other half lives. It's not hard to discuss with a serverless advocate and suddenly feel like you're
comparing serverless in 2024 against servers in 2010. It's pretty clear how servers in 2010 completely
loses this battle, but for a more productive conversation to flow, we need to bring it to parity.
Software built with automation testing practices, relying on predictable release cadence of dependencies
(Operating System, Programming Language, Database Engine, etc) and running on top of highly performant
microchips are hardly comparable to monoliths at the end of the 2010 era running on purpose-built 
single server in a physical server room. Operating system crashes are more uncommon, CVEs and security 
practices are well established, semantic versioning helps in preventing BC breaks, release schedules
makes it predictable when updates are due. All of this empowers the most important part of server management:
**automation**. A good security strategy is to make sure the system can automatically install security fixes.
And we have service providers that are able to provide such automation because of all improvements that
we've seen in the software industry (hello SSM Patch Manager).

### Conclusion

I'm an advocate for serverless and I have deep hands-on experience with it. But I think the next
decade will have more innovation in the server and monolith space than with Serverless. Serverless
has been crucial in the trajectory towards cloud computing, breaking monoliths apart, providing
scalability and improving performance by scaling horizontally. When money is a constraint, innovation
plays a big role in making things simpler, efficient and focused on what's essential. I expect the software
industry to start a movement towards slimming down. Too many technologies spread across too many teams
is a liability without profitability. Horizontal scaling and system orchestration can be one, too.
An application running on a single server with a single tech stack and focus on a business strategy
can accomplish a lot nowadays. But that doesn't mean we need to throw out every good practice discovered
in the last 20 years and go back to 2000's. Even when running on a single server, we can still rely on S3,
use a managed database service, use a load balancer and stateless implementations, dedicate time to automation
testing and release everyday with CI/CD strategies.

Let's keep ourselves open to what come next!

Follow me on

- https://twitter.com/deleugyn
- https://fosstodon.org/@deleugpn
- https://threads.net/@marcodeleu

Cheers.
