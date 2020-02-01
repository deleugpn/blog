---
title: AWS VPC for Software Engineers
date: "2020-02-02T09:43:30.284Z"
description: A helpful guide on AWS VPC components and how they work together. 
---

I've been working with AWS for about 3 years now. Going to the cloud can
be extremely overwhelming and stressful. I jumped head first on a company
that wanted to setup a brand new AWS account from scratch, setup pipelines
for CI/CD, dockernize all services and use CloudFormation for provisioning
everything. When I joined the company I had never logged into AWS console before
and during my first meeting people kept talking about EC2 and I couldn't for the
life of me figure out what that means. I ended up leading most of this migration
and now, 3 years later, we're about 95% migrated. In this post I want to talk about
how I finally managed to understand AWS VPC from a developer's perspective.

#### VPC

When I was a teen we used to have a lot of places called "LAN HOUSE"
where you would pay per hour to use computers. They were connected
to the internet and to each other, allowing games like Counter Strike
to be played in LAN (Local Area Network) mode.
A Virtual Private Cloud is a fancy name for a network of computers.
It helped me associate these terms and assume a VPC is almost like
an Internet Café from the 90's, early 2000's: computers in the same
physical place directly connected to each other, forming a network
and sharing an internet access point.

#### Subnets

Let me start by saying something that might be obvious for some,
but took me longer than I'd like to admit to understand: There's nothing
special about a subnet that makes it public or private. Every subnet
is pretty much the same. The difference is in the Route Table, not the
subnet itself.

Now that I got that out of the way, let's go back to the beginning.
If you've been fiddling with computers in the 90's, you've probably
configured a network card once in your life where you had an IP address
of `192.168.1.1`, `255.255.255.0` and a default gateway `192.168.1.254`.
Perhaps you didn't have these exact numbers, but if you did it's not
coincidence that I know them. A subnet is like a boundary for a local
network. On Linux, it's common to refer to the subnet mask as `192.168.1.0/24`
and that's how AWS refers to it as well.

Travelling back to the Internet Café example, I remember once going to 
the mall and there was a cyber café there, the biggest I've ever seen.
There was 3 floors and maybe more than 70 computers. It was amazing for
a 13 year old kid. Now let's pretend for the sake of fun that each floor
has it's own local network. People on the 3rd floor cannot play together
in LAN mode with people on the 2nd floor. They are in different subnets.
Computers in the 1st floor use IP addresses like `192.168.1.1` and the 3rd
floor uses `192.168.3.1`. The owner of the cyber cafe decided to define
3 subnets, one for each floor and restrict communication between them.
One subnet would have a mask of `192.168.1.0/24`, the 2nd would have 
`192.168.2.0/24` and the 3rd would be `192.168.3.0/24`. On it's own,
these subnets would not be able to communicate with each other.

#### Route Table

The route table contains rules and patterns on how a network package
is dispatched through the local network area. AWS offers a special rule
called `local`, which means the VPC itself. This is how Subnets are able
to communicate with each other. When something on subnet 1 wants to communicate
with an IP that belongs to subnet 2, the `local` entry in the Route Table
defines how that communication will be established.
I think the Route Table doesn't get the attention it deserves. There
are so many content talking about public and private subnet that doesn't
touch on this: The Route Table is responsible for defining whether
a subnet is `public` or `private`. These are human readable constructs.
All subnets are practically the same. What AWS defined as a public
subnet is a subnet that has an Internet Gateway in it's Route Table.
In other words, a device running on a public subnet can communicate
with the internet. A private subnet, to communicate with the internet,
will require a NAT Gateway in the  Route Table.

#### NAT

NAT stands for Network Address Translation and is used to translate
local network addresses into public routable addresses. Understanding
IPv4 is key to understanding why NAT even exists. Let's remember.
IPv4 has 3 classes of private ip addresses: `10.0.0.0/8`, `172.16.0.0/12` and `192.168.0.0/16`.
Any local computer network will have IP addresses that are inside
these subnet definition. They are reserved as private ips and can 
never be routed on the internet. Put differently, a website will never
have an ip address of `192.168.56.1` on the internet. With the exception
of loopback interface, pretty much all other ip addresses on IPv4 is 
public. It means that your internet device at your home is capable
of sending a request to `55.55.55.55` and getting a response back.

The easiest way to understand NAT is to use your home as an example.
Your smartphone or laptop is assigned a local ip address from your
internet modem. When you try to communicate with the internet, the network
card in your laptop sends a request to your internet modem containing
information you're trying to retrieve. The modem will then make a note
of your computer's local ip address and use it's own public ip address
to talk to the internet. That is the essence of translating network
addresses. It translates a private one with a public one. Once
the response from the internet comes back to the modem, it can then
check who had requested that information (your laptop) and translate
back to local address so that the package reaches your computer.

Things running on a `private subnet` will have a `private` ip address
and will need NAT to communicate with the internet. Nothing on the internet
can communicate directly with a device in a private subnet.

Things running on a `public subnet` will also have a `private` ip address.
And this was partially the confusing part for me for a long time. The reality
is that any compute device you launch on a VPC will either be part
of a public or private subnet but will have a private ip address either
way. The key difference is that **if you auto-asign public ip** to said device,
it will be able to communicate directly to the internet without NAT.
The Route Table of a public subnet contains an `internet gateway` rule
and not a NAT Gateway rule.

Bonus confusion: A NAT Gateway is a compute device. as such, it has a network
card and is in a subnet. We place the NAT Gateway in a public subnet
because it needs to be able to communicate with the internet.
When a device in a private subnet tries to reach the internet, it goes
more or less like this:

1. Your device with a private IP prepares a network packet
2. The Route Table says the packet should go to the NAT Gateway
3. The NAT Gateway translates the private ip into a public IP
4. The NAT Gateway's Route Table routes the packet to the Internet Gateway
5. The packet reaches the internet and comes back to the NAT Gateway
6. The NAT Gateway translates the IP back to your device's private ip.
7. The packet reaches back to your device.

#### Availability Zones

AWS is much more complex and interesting than a cyber café with a few
computers connected directly via a network cable. A VPC exists within
an AWS Region and is available on several availability zones.
This means AWS has to establish a secure channel between 2 different cities
to establish communication between two subnets in the same VPC.
This, however, is completely abstrated away and not that much relevant
for a day to day work with AWS. When configuring the Route Table with
`local` construct, AWS will handle any communication between 2 subnets
 in different AZs.


#### VPC Endpoint

Once I finally understood subnets and how the Route Table is responsible
for internet communication (either via IG or NAT), it was mind-blowing
to understand how AWS leverage this to offer VPC Endpoint.

Some AWS services are VPC-agnostic, such as S3, SQS, SNS, DynamoDB, etc.
They are _computeless_ resources from the account perspective. Any
compute resource you want to run on your AWS Account has to run inside
a VPC (except Lambda, but that's a whole other story). Services that are
API-driven and does not require any compute resources on our side does not
live inside our VPC. The only way to access them is via the public internet.
What if AWS bought a range of public ip addresses and assign it to 
a specific service? For instance, S3 could be running on `54.231.0.0/17`.
This means we can leverage our Route Table for a specific rule.
If an internet packet is directed to an IP address that matches this rule,
use this VPC endpoint. Instead of falling back to the Internet Gateway,
AWS can actually identify that you're going to communicate with S3 and
use a dedicated infrastructure for that so that you don't have to pay for
NAT. It is still a NAT that is making it all possible, but from AWS perspective,
they built a special kind of NAT just for S3 so that they can charge less
and use a structure optimized for that.
  
#### Conclusion

This post covers many components such as AWS VPC, Subnet, Internet Gateway, 
NAT Gateway, VPC Endpoint, etc. Perhaps one or two terms used in the text
might be semantically incorrect or from a Network Administrator perspective
I might be saying a lot of crap, but I decided to publish this anyway because
for me, a developer, this knowledge helps a lot to debug and understand
AWS infrastructure.

Hope you enjoyed the reading. If you have any questions,
send them my way on [Twitter](https://twitter.com/deleugyn).

Cheers.