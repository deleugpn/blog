---
title: FrankenPHP 502 Bad Gateway behind AWS ALB
date: "2024-03-22T18:42:00.000Z"
---

This week I took FrankenPHP for a spin with AWS Fargate.
I've been using AWS Lambda for so long that I forgot
one hard lesson about Application Load Balancers.
I also learned another lesson today.

Before I get started, let me tell you this: my container
was working. I knew this because I launched it with
a Public IP and if I used `http://{container-public-ip}`
my FrankenPHP would respond with my application successfully.

Why the hell is my Web application working correctly,
but when using an Application Load Balancer, 
it gives back 502 Bad Gateway?

### Lesson 1: Unhealthy targets do not receive traffic

I used to know this, but I forgot about it a long time ago.
When we register a target behind AWS ALB using Target Groups,
if the target doesn't respond the healthcheck successfully, 
ALB doesn't even waste time trying to forward 
requests to the target, it replies back with 
502 Bad Gateway right away.

Next I needed to investigate why the AWS ALB healthcheck 
was failing.

### Lesson 2: HTTP/2 has an upgrade request from HTTP/1.1

I'm not going to deep dive into this because I didn't do my
research. This post is mostly informational in the hopes
to help anyone spend hours debugging this issue.

I had configured my Target Group to talk to my target
container using HTTP/2 Protocol. Healthcheck was always
tagging my container as unhealthy. I decided to start FrankenPHP
with the `-v` flag (debugging) and the `--access-log` flag.
This helped me notice that my target container was
receiving an odd traffic: it had no `host` header,
the `uri` was set to `*` and the HTTP Method was `PRI`.
I've been working with Web applications for 15 years
and I have never heard of a `PRI` HTTP Method, so I
did a quick googling and I came across the shallow
information that it has something to do with requesting
a web server to upgrade the communication process to
HTTP/2.

This is a bit of a sad situation. HTTP/2 is 10 years
old (give or take?) and it was supposed to work fine.
In fact, when I made plain HTTP request straight
to my container, Google Chrome would establish an
HTTP/2 connection with the container and the Web Server
would just work.

There is something funky going on either in FrankenPHP,
Caddy or AWS ALB. Even though I explicitly configured
AWS ALB to use HTTP/2 Protocol, the ALB would still
send a HTTP/1.1 request to my container asking it
if it could upgrade to HTTP/2. This request was
failing with a 404. Instead of actually talking
HTTP/2 with my container, the `PRI` HTTP request
was failing to upgrade to HTTP/2, which lead
to AWS ALB tagging my container as unhealthy,
making the whole particularly hard to debug.

After changing the Target Group to use HTTP/1.1
everything worked perfectly.

---

Follow me on

- https://twitter.com/deleugyn
- https://fosstodon.org/@deleugpn
- https://threads.net/@marcodeleu

Cheers.
