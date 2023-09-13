---
title: "AWS Lambda, PHP, Bref and Datadog"
date: "2022-10-29T12:28:40.121Z"
description: Getting Datadog to trace your PHP application on AWS Lambda 
---

I was recently working on a project where we're pretty much lifting a Laravel application 
from EC2 into AWS Lambda. The project has been using Datadog with the 
[PHP Extension for Datadog](https://github.com/DataDog/dd-trace-php). The
Datadog Agent is installed inside the EC2 instance and everything is working fine.

Although Datadog does not document this, I had a hunch that I could get this to work with
AWS Lambda without making any crazy changes. The first thing that helped was the fact
we had decided to run [Docker on AWS Lambda](https://bref.sh/docs/web-apps/docker.html).
I know this is a last resort (I was the author of that Bref documentation), but we had
good reasons to pursue this. The application takes about 600MB of disk space and AWS
Lambda is limited to 250MB unzipped (50MB zipped). We also would need to install
wkhtmltopdf, imagick, redis and gd, besides Datadog.

I ran into [this issue](https://github.com/DataDog/dd-trace-php/issues/1207)
on their GitHub repository which made me think perhaps this was not going to work,
but as far as I understood, they were trying to use the Datadog Forwarder while 
I had something else in mind.

My thought process was:
- We're using Docker for Lambda so we pretty much have an Amazon Linux
- We need to figure out how to install the Datadog PHP Extension on Amazon Linux
- The [Datadog Lambda Extension](https://github.com/DataDog/datadog-lambda-extension) would probably be the Agent itself
- PHP running on Lambda would use the Extension to talk to the Lambda Extension
- The Extension uses TCP to talk to the Datadog Agent (localhost on lambda)

#### Datadog PHP Extension

So I went chasing this process. Following [this part of the docs](https://docs.datadoghq.com/tracing/trace_collection/dd_libraries/php/?tab=containers#install-the-extension)
I managed to get Datadog PHP Extension installed on Docker with the following modifications:

```Dockerfile
RUN yum install -y tar gzip \
&&  curl -LO https://github.com/DataDog/dd-trace-php/releases/latest/download/datadog-setup.php \
&&  ln -s /sbin/ldconfig /usr/local/bin/ldconfig \
&&  php datadog-setup.php --php-bin=all \
&&  rm -f datadog-setup.php
```

I ended up with this snippet because the Datadog extension installer was giving me some errors related to `tar` and `gzip`
as well as `ldconfig` not being found.

I then had to make sure I could place some `ini` files for both `cli` and `fpm`. Bref already loads `ini` files from your
project `php/conf.d` directory, so I created 2 folders here:

- `php/conf.d/cli`
- `php/conf.d/fpm`

Then in my Dockerfile I have one stage for CLI and one for FPM:

```Dockerfile
RUN mv /var/task/php/conf.d/cli/* /var/task/php/conf.d/
```

```Dockerfile
RUN mv /var/task/php/conf.d/fpm/* /var/task/php/conf.d/
```

#### Datadog Lambda Extension

Next I found [this documentation](https://docs.datadoghq.com/serverless/installation/python/?tab=containerimage) which
explained how to install Datadog Lambda Extension for containers. This is the snippet I have for Bref and PHP Extensions:

```Dockerfile
FROM bref/php-74 as extensions

COPY --from=bref/extra-redis-php-74:0.11 /opt/bref-extra /opt/bref-extra

COPY --from=bref/extra-gd-php-74:0.11 /opt/bref-extra /opt/bref-extra
COPY --from=bref/extra-gd-php-74:0.11 /opt/bref /opt/bref/

COPY --from=bref/extra-imagick-php-74:0.11 /opt/bref-extra /opt/bref-extra
COPY --from=bref/extra-imagick-php-74:0.11 /opt/bref /opt/bref/
COPY --from=bref/extra-imagick-php-74:0.11 /opt/bin/gs /opt/bin/gs

COPY --from=public.ecr.aws/datadog/lambda-extension:31 /opt/extensions/ /opt/extensions
```

Note that the most important aspect for Datadog here is the last line where we copy the extension into our Docker image.

The last thing I needed was some environment variables to stitch this all together. I put the following
environment variables on my Lambda in order to be able to figure out how things were working:

```yml
        DD_API_KEY: !Sub '{{resolve:ssm:/${Environment}/app/services/datadog/key}}'
        DD_SERVICE: app-lambda
        DD_ENV: !Ref Environment
        DD_VERSION: 1.0.0
        DD_AGENT_HOSTNAME: 127.0.0.1
        DD_TRACE_CLI_ENABLED: true
        DD_TRACE_ENABLED: true
        DD_TRACE_DEBUG: true
        DD_SITE: datadoghq.com
        DD_LOG_LEVEL: trace
```

The `DD_LOG_LEVEL` and the `DD_TRACE_DEBUG` are not necessary though. They make the logs a bit noisy. I just used
them when I was still debugging and figuring out how things were going. By default, the Datadog Agent will listen
on port `8126` and the Datadog PHP Extension will also talk to `localhost:8126`.

### Conclusion

This was quite interesting for me because I was stitching together multiple sources of Datadog documentation.
Some were PHP-related docs and others were Serverless-related docs. I had this hunch that the Datadog Extension
was actually the Agent and that the PHP Extension would communicate with the Agent via TCP which meant that
there was nothing that would prevent me from having this running on AWS Lambda. I was super happy when everything
was working correctly and one of the nice things is that the CloudWatch logs already get automatically forwarded
to Datadog.

Hopefully the folks at Datadog can use this to write some dedicated documentation for it.

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.