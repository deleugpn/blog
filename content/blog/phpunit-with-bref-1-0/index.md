---
title: PHPUnit with Bref 1.0
date: "2021-03-06T21:03:37.350Z"
description: Running your tests with the php binary compiled by Bref
---

Last year [I wrote about how I use the PHP binary compiled
by Bref to run automation tests on my CI pipeline](https://blog.deleu.dev/phpunit-with-bref/).
The primary drivers for wanting that is so that I can 
guarantee that if my tests are passing on my pipeline,
then chances are the code is more likely to work on
production than if I run it against a different php binary
that could have been compiled with different extensions
or have some configuration discrepancy.

With the release of Bref 1.0, I looked further into that
process and I managed to come up with a better strategy
to achieve the same thing.

#### Docker setup

I start with the `docker-compose.yaml` file which contains
the configuration necessary to start a brand new container
with my source code.

```yaml
version: '3.7'

services:
  phpunit:
    build:
      context: php/build
    entrypoint: /usr/bin/tail
    command: ["-f", "/dev/null"]
    user: root
    working_dir: /var/task
    volumes:
      - .:/var/task
      - ./php/build:/root/build
      - ./tests/bref.ini:/opt/bref/etc/php/conf.d/bref.ini  # Disable opcache
    environment:
      - APP_ENV=testing
```

This configuration will start a container that stays on
forever by running `/usr/bin/tail -f /dev/null` on start up.
The source code has to be mounted on `/var/task` as it's
where AWS place the source code on Lambda.

The `/tests/bref.ini` is a stripped down copy of 
[Bref default php.ini](https://github.com/brefphp/bref/blob/master/runtime/layers/function/php.ini)
without opcache.

```ini
; On the CLI we want errors to be sent to stdout -> those will end up in CloudWatch
display_errors=1

; Since PHP 7.4 the default value is E_ALL
; We override it to set the recommended configuration value for production.
; See https://github.com/php/php-src/blob/d91abf76e01a3c39424e8192ad049f473f900936/php.ini-production#L463
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT

; This directive determines which super global arrays are registered when PHP
; starts up. G,P,C,E & S are abbreviations for the following respective super
; globals: GET, POST, COOKIE, ENV and SERVER.
; We explicitly populate all variables else ENV is not populated by default.
; See https://github.com/brefphp/bref/pull/291
variables_order="EGPCS"
```

As for the Docker image, this is how the Dockerfile looks
like:

```Dockerfile
FROM bref/php-80:1.0.2

COPY --from=bref/extra-redis-php-80:0.8 /opt/bref-extra /opt/bref-extra

COPY --from=bref/extra-gmp-php-80:0.8 /opt/bref-extra /opt/bref-extra
COPY --from=bref/extra-gmp-php-80:0.8 /opt/bref/ /opt/bref
```

This is an example of a Dockerfile that makes use of
Redis and GMP extensions. More extensions are available 
here: https://github.com/brefphp/extra-php-extensions.

As soon as the container starts, I can interact with it
like any other container and run `phpunit` using a PHP
binary that greatly resembles how AWS Lambda would be running
my PHP code, including the extensions that I choose to enable
via Lambda Layers.

#### Conclusion

I know this can be considered nitpicking or a bit extreme, 
but part of the CI's job is to give me confidence and
comfort to easily release code changes faster. Part of
that confidence comes from the fact that I know that
the PHP binary Bref is providing is at least capable of
executing my code. If I suddenly add an extension to my
local environment for development and write code that relies
on it, my CI will make sure to remind me that I need to do
the same thing for CI/Production setup.
Differently than how I used to do it (documented in my
previous post), this is a much easier and efficient way
to add/remove extensions. 


As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.