---
title: PHPUnit with Bref
date: "2020-05-04T20:46:30.284Z"
description: Running your tests with the php binary compiled by Bref
---

One thing I learned to love with Docker was the easy ability to run
phpunit with the exact same environment that will be used in production.
Whenever I'm setting up a release pipeline, I like to build a Docker
image, start a container from that image, enter it and run the project
test code. If everything passes, I can `composer install --no-dev`,
push the image to a docker registry and release it. Yes, if one of
my dev dependencies is not actually a dev dependency things can still
break. Another source of production bugs in this scenario revolves
around environment variables. But the point is to minimize, as much
as possible, the difference between production and the test environment.
By doing so, we catch some obvious bugs early in the process. In this
post I want to walk through my process of doing the same thing when
aiming to deploy to AWS Lambda using Bref.

#### The Test Case

I want to start this off with a test case. Let's suppose we have the
following test case to run before any deployments:

```php
<?php declare(strict_types=1);

require 'vendor/autoload.php';

class MyTest extends \PHPUnit\Framework\TestCase
{
    public function test_gmp(): void
    {
        \PHPUnit\Framework\Assert::assertSame('2', gmp_strval(gmp_add(1, 1)));
    }
}
```

Of course on a real project, we'd be interacting with the project's
source code from our tests in order to gauge whether the codebase
is in a working state. For brevity, let's assume we have some
code that depends on PHP GMP extension and this test is suppose
to cover such case. We can go about running this on a local machine
with the following Dockerfile and phpunit configuration:

```dockerfile
FROM alpine:3.11

RUN apk add composer php7-dom php7-xmlwriter php7-xml php7-gmp php7-tokenizer

RUN composer require phpunit/phpunit

COPY MyTest.php tests/MyTest.php

COPY phpunit.xml .

CMD ["vendor/bin/phpunit"]
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/9.1/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         executionOrder="depends,defects"
         forceCoversAnnotation="true"
         beStrictAboutCoversAnnotation="true"
         beStrictAboutOutputDuringTests="true"
         beStrictAboutTodoAnnotatedTests="true"
         verbose="true">
    <testsuites>
        <testsuite name="default">
            <directory suffix="Test.php">tests</directory>
        </testsuite>
    </testsuites>

    <filter>
        <whitelist processUncoveredFilesFromWhitelist="true">
            <directory suffix=".php">src</directory>
        </whitelist>
    </filter>
</phpunit>
```

By placing these 3 files in a folder, I can run the following command
and receive the expected output.

```shell script
 docker build . -t bref/phpunit && docker run bref/phpunit
```

```shell script
# docker build . -t bref/phpunit && docker run bref/phpunit
Sending build context to Docker daemon  4.608kB
Step 1/6 : FROM alpine:3.11
 ---> e7d92cdc71fe
Step 2/6 : RUN apk add composer php7-dom php7-xmlwriter php7-xml php7-gmp php7-tokenizer
 ---> Using cache
 ---> c4c49e71d793
Step 3/6 : RUN composer require phpunit/phpunit
 ---> Using cache
 ---> c713de154e43
Step 4/6 : COPY MyTest.php tests/MyTest.php
 ---> Using cache
 ---> 1e41fb02f5d7
Step 5/6 : COPY phpunit.xml .
 ---> Using cache
 ---> 9c30786b9ea6
Step 6/6 : CMD ["vendor/bin/phpunit"]
 ---> Using cache
 ---> 6ffd750bd407
Successfully built 6ffd750bd407
Successfully tagged bref/phpunit:latest
PHPUnit 9.1.4 by Sebastian Bergmann and contributors.

Runtime:       PHP 7.3.17
Configuration: /phpunit.xml

.                                                                   1 / 1 (100%)

Time: 00:00.034, Memory: 4.00 MB

OK (1 test, 1 assertion)
```

However, if we were to deploy the actual code to AWS Lambda, it
would very likely fail to execute with an error saying
`PHP Warning:  Uncaught Error: Call to undefined function gmp_add()`.

#### PHPUnit & Bref

Bref comes with a handy Docker image for tests which we can find
at https://hub.docker.com/r/bref/php-74. The awesomeness about
this image is that it is built with the exact same php binary
that Lambda will use to run the deployed code. For more on this image,
check https://github.com/brefphp/bref/blob/master/runtime/base/php-74.Dockerfile.

So how do we go about using this image for PHPUnit testing? For
this I like to use a `docker-compose` file. Here's how we can
start:

```yaml
version: '3.7'

services:
  my-service:
    image: bref/php-74
    entrypoint: /usr/bin/tail
    command: ["-f", "/dev/null"]
    user: root
    working_dir: /var/task
    volumes:
      - .:/var/task
      - ./php/build:/root/build
      - ./tests/setup/bref.ini:/opt/bref/etc/php/conf.d/bref.ini  # Disable opcache
``` 

The idea behind setting `tail` as entrypoint is so that the container
stays alive in a long running process. This is extremely helpful
in CI servers when we need our tests to wait for the database container
to be ready. I wish I had come up with that genius hack, but credit 
for that goes to [Abdala Cerqueira](https://abda.la/).
We mount our code on `/var/task` which matches the expectation of
AWS Lambda. I also like to include a `tests/setup/bref.ini` in my
projects to override the default `bref.ini` because it comes with
opcache enabled.

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

This file is a slim down version of the original file that comes with
Bref, the only change is removing opcache extension. Check out the
file at https://github.com/brefphp/bref/blob/master/runtime/layers/function/php.ini.

Lastly, we have the volume for `php/build`. The idea behind this
one is that Bref suggests you to have a `php` folder at the root
of your project and place any `ini` file you want to be included
by AWS Lambda inside of it. I decided to use this folder for
extra things, such as installing composer. The thing is, since
AWS Lambda base operating system is Amazon Linux, which itself
is based off of RedHat, we lose all of our convenience of Alpine
such as `apk add composer`. Inside this folder I usually put
two files: `dev-composer.sh` and `prod-composer.sh`.

```shell script
#!/usr/bin/env sh

cd /tmp

php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"

php composer-setup.php --filename=composer

php composer install --working-dir /var/task --no-interaction
```

```shell script
#!/usr/bin/env sh

cd /tmp

php composer install --no-dev --working-dir /var/task --no-interaction
```

The idea behind all of this is that in my pipeline I'll have a stage
that will do roughly the following:

```yaml
# Start services
- docker-compose up -d my-service

# Install dependencies
- docker-compose exec -T my-service chmod +x /root/build/dev-composer.sh
- docker-compose exec -T my-service /root/build/dev-composer.sh

# Run phpunit
- docker-compose exec -T my-service ./vendor/bin/phpunit

# Remove dev dependencies
- docker-compose exec -T my-service chmod +x /root/build/prod-composer.sh
- docker-compose exec -T my-service /root/build/prod-composer.sh

# Remove any unnecessary file
- rm php/build -rf

# Package and Deploy the Lambda 
```

However, if we run this right now, we get an error with the following
output:

```shell script
# docker-compose exec -T my-service ./vendor/bin/phpunit
PHPUnit 9.1.4 by Sebastian Bergmann and contributors.

Runtime:       PHP 7.4.2
Configuration: /var/task/phpunit.xml

E                                                                   1 / 1 (100%)

Time: 00:00.065, Memory: 4.00 MB

There was 1 error:

1) MyTest::test_gmp
Error: Call to undefined function gmp_strval()

/var/task/tests/MyTest.php:9

ERRORS!
Tests: 1, Assertions: 0, Errors: 1.
```

This is because Bref **does not** come with PHP GMP by default, we
need to use an additional layer to get that. You can read a bit
about using Bref layers locally at https://github.com/brefphp/extra-php-extensions/issues/9.

The short version is that we're going to `docker pull bref/extra-gmp-php-74`,
start a new container with it and then copy all of the necessary files
into the `php/build` folder. That way, we'll have all of the necessary
dependencies ready inside the project for when the pipeline runs
and at the end of the pipeline we can run `rm php/build -rf` to get
rid of it before actually uploading the source code to S3 for deployment
so that we don't upload several MB of data that will not be used.
Remember, when the code is actually running inside AWS Lambda, we'll
have the Layer ARN including this extension. The docker image is
just a convenience for local development / CI server to keep parity.

The following sequence of commands will bring the files that we need
outside of the container

```shell script
docker run --entrypoint /bin/bash --user root -w /tmp/php-ext -v $(pwd)/php/build/ext-gmp-php74:/tmp/php-ext -it bref/extra-gmp-php-74
cp /opt/bref-extra/ . -R
cp /opt/bref/include/ . -R
cp /opt/bref/lib64/ . -R
```

Now that we have all of the necessary files copied into our host machine,
we can modify the `docker-compose.yaml` file to include some extra
volumes for our local testing:

```yaml
version: '3.7'

services:
  my-service:
    image: bref/php-74
    entrypoint: /usr/bin/tail
    command: ["-f", "/dev/null"]
    user: root
    working_dir: /var/task
    volumes:
      - .:/var/task
      - ./php/build:/root/build
      - ./tests/setup/bref.ini:/opt/bref/etc/php/conf.d/bref.ini  # Disable opcache
      
      # php7-gmp exntension: https://github.com/brefphp/extra-php-extensions/tree/master/layers/gmp
      - ./php/build/ext-gmp-php74/bref-extra/gmp.so:/opt/bref-extra/gmp.so
      - ./php/build/ext-gmp-php74/bref-extra/gmp.so:/opt/bref/lib/php/extensions/no-debug-zts-20190902/gmp.so
      - ./php/build/ext-gmp-php74/lib64/libgmp.so:/opt/bref/lib64/libgmp.so
      - ./php/build/ext-gmp-php74/lib64/libgmpxx.so:/opt/bref/lib64/libgmpxx.so
      - ./php/build/ext-gmp-php74/include/gmp-mparam-x86_64.h:/opt/bref/include/gmp-mparam-x86_64.h
      - ./php/build/ext-gmp-php74/include/gmp-mparam.h:/opt/bref/include/gmp-mparam.h
      - ./php/build/ext-gmp-php74/include/gmp-x86_64.h:/opt/bref/include/gmp-x86_64.h
      - ./php/build/ext-gmp-php74/include/gmp.h:/opt/bref/include/gmp.h
      - ./php/build/ext-gmp-php74/include/gmpxx.h:/opt/bref/include/gmpxx.h
```

Now that we included GMP extension (from the original layer) into
our local container, all we need is to enable the extension with
a `php.ini` inside `./php/conf.d/php.ini` as follows:

```ini
extension=/opt/bref-extra/gmp.so
```

After shutting down the container and starting it again from the
modified `docker-compose.yaml`, I can successfully run the test

```shell script
# docker-compose exec -T my-service ./vendor/bin/phpunit
PHPUnit 9.1.4 by Sebastian Bergmann and contributors.

Runtime:       PHP 7.4.2
Configuration: /var/task/phpunit.xml

.                                                                   1 / 1 (100%)

Time: 00:00.037, Memory: 4.00 MB

OK (1 test, 1 assertion)
```

#### Conclusion

I know this is a little bit of work, but I honestly feel like it is
worth it. After running all of my phpunit tests against my code
using the exact same binary as the one that will be in production
and making sure that the extensions are an exact match, I can feel
more confident in deploying changes. I only have to pay extra
attention when depending on new environment variables because
we usually define those on the docker-compose file, but have to
replicate them on the `serverless.yaml` template for AWS. The PHP
version is the same and the extensions are also the same. That way,
if I need to enable a new extension on production, I have to make
sure my pipeline will have the same extension available so that
I can test my code against it before making any changes to the
production lambda. The fact that Bref offers all of these Docker
images and keep them up to date is wonderful and makes it easy
to grab some new extensions for local development.

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.