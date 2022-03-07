---
title: "3 things I learned working on a Bref Runtime"
date: "2022-03-06T23:45:27.002Z"
description:  The architecture behind running PHP on AWS Lambda.
---

I have been working on a new [Runtime for Bref](https://github.com/brefphp/bref/pull/1078)
for a while now and it has been my biggest open source journey so far. It has been incredibly
challenging and a great learning experience. In this post I want to run through some
key aspects that I faced while working on this. I received a lot of help on Twitter and learned
a lot about AWS, Makefile, Shared Objects and more.

#### PHP Extensions and Shared Objects (.so) files

PHP is very easy to get started and there are tons of tutorials on the internet teaching how to get
started with some specific aspects of the language. Beginners are able to enable an extension
by uncomment something on a php.ini file (such as `;extension=dom.so`) or installing it from a distribution
like `yum install php81-php-bcmath`. What this does is instruct PHP to load an extra extension to provide
more functionalities. When an extension is not loaded we often see a side effect such as  
`Call to undefined function iconv_strlen()` or `Class PDO not found`. When an extension is enabled on
PHP, it's Shared Object (.so) file is loaded to provide the appropriate functionality. In a way, a code
that uses an extension has a direct dependence to that `.so` file. However, it's interesting to think that
the shared object may have a dependency of it's own. Let's take PHP's `pgpsql` extension as an example:

```bash
bash-4.2# ldd /opt/remi/php81/root/lib64/php/modules/pdo_pgsql.so
        linux-vdso.so.1 (0x00007ffd33ff6000)
        libpq.so.5 => /lib64/libpq.so.5 (0x00007f5c47f23000)
        libc.so.6 => /lib64/libc.so.6 (0x00007f5c47b78000)
        libssl.so.10 => /lib64/libssl.so.10 (0x00007f5c47909000)
        libcrypto.so.10 => /lib64/libcrypto.so.10 (0x00007f5c474b4000)
        libkrb5.so.3 => /lib64/libkrb5.so.3 (0x00007f5c471d0000)
        libcom_err.so.2 => /lib64/libcom_err.so.2 (0x00007f5c46fcc000)
        libgssapi_krb5.so.2 => /lib64/libgssapi_krb5.so.2 (0x00007f5c46d80000)
        libldap_r-2.4.so.2 => /lib64/libldap_r-2.4.so.2 (0x00007f5c46b25000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f5c46907000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f5c48150000)
        libk5crypto.so.3 => /lib64/libk5crypto.so.3 (0x00007f5c466d6000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007f5c464d2000)
        libz.so.1 => /lib64/libz.so.1 (0x00007f5c462bd000)
        libkrb5support.so.0 => /lib64/libkrb5support.so.0 (0x00007f5c460ae000)
        libkeyutils.so.1 => /lib64/libkeyutils.so.1 (0x00007f5c45eaa000)
        libresolv.so.2 => /lib64/libresolv.so.2 (0x00007f5c45c94000)
        liblber-2.4.so.2 => /lib64/liblber-2.4.so.2 (0x00007f5c45a85000)
        libsasl2.so.3 => /lib64/libsasl2.so.3 (0x00007f5c45868000)
        libssl3.so => /lib64/libssl3.so (0x00007f5c4560e000)
        libsmime3.so => /lib64/libsmime3.so (0x00007f5c453e8000)
        libnss3.so => /lib64/libnss3.so (0x00007f5c450c0000)
        libnssutil3.so => /lib64/libnssutil3.so (0x00007f5c44e91000)
        libplds4.so => /lib64/libplds4.so (0x00007f5c44c8d000)
        libplc4.so => /lib64/libplc4.so (0x00007f5c44a88000)
        libnspr4.so => /lib64/libnspr4.so (0x00007f5c4484c000)
        libselinux.so.1 => /lib64/libselinux.so.1 (0x00007f5c44625000)
        libcrypt.so.1 => /lib64/libcrypt.so.1 (0x00007f5c443ee000)
        librt.so.1 => /lib64/librt.so.1 (0x00007f5c441e6000)
        libpcre.so.1 => /lib64/libpcre.so.1 (0x00007f5c43f82000)
```

Here we execute `ldd` on a Shared Object file and we are presented with all of the dependencies
of that particular file. Some of these dependencies are on the operating system / kernel level, while others
are a chain of software dependency. For instance, I can make a guess that the `pgsql` extension on PHP
depends on `libssl3.so` in order to offer SSL Encryption for the connection. 

Some shared object files are so common that they are part of `/lib64` folder even on a clean Linux installation
while others will be installed as a dependency of something we pull in. The reason why this was important
during my work on Bref is because we attempt to make the Runtime as minimal as possible to avoid hitting
AWS Lambda 250MB limit. Every file we pull in adds an overhead. Things like instruction / man files, HTML
or documentation are not necessary. We only need to pull into the layer the PHP binary, every dependency
of the PHP Binary and every dependency of the `so` files as well as the extensions itself.
Only copying `pgsql.so` is not enough as when that code runs it may crash by making a call to ther
shared libraries.

#### The more dependencies we have, the harder it is to orchestrate them

AWS Lambda Layer is a zip file, uploaded to a specific region, protected by AWS IAM that provides files
that will be included on the Lambda Execution environment. AWS alone imposes a few restrictions when
working with lambda layers.

- We need a program to write zip files
- We can only place files under `/opt` folder
- Lambda functions cannot load layers cross-region
- Lambda Layers are forcefully versioned by AWS
- AWS Lambda is limited to 250MB, therefore layers should strive for minimal sizing
- Layers are limited to it's own AWS account only unless explicitly authorized

AWS alone gives a lot of restriction already. A project like Bref seeks to provide the PHP binary (and it's
dependencies) under the `/opt` folder, limited to as little files as necessary, across all AWS regions
and publicly available for anyone that wants to use it. In order to fulfill these requirements we chose
Docker as one of the ~least worse~ best option for file isolation. With that decision we also bring
limitations from Docker such as the limited support and bugged implementation on ARM architecture.
A Dockerfile is also a declarative language and has limitations in reusing the same file for multiple
CPU architecture and/or multiple PHP versions. We ended up with multiple Dockerfile (one per PHP version)
instead of attempting to use environment variables / arguments to bypass the declarative language
into conditional execution.
The requirement to publish layers on all region means we can't just upload the zip file to S3 once
and cross-publish it on all regions. We must actually upload the same content over and over again. This
limitation stumble on a potential network limitation because trying to upload the same file to 20+ AWS
regions often leads to a network error.
Speaking of dependencies, we could not leave Composer out of the conversation. Composer does not accept
one GitHub Repository to host multiple packages so separating the Runtime code from the Bref Package is
something that must be orchestrated within the code itself. There is also a direct dependency between
the layer version published and the Composer Package version installed. Since we don't have control
over the Lambda Layer version, we must keep a table of compatibility internally. This is then automatically
offered to the user through the Serverless Framework plugin that allows us to map between the composer
version installed and the lambda layer version that will be used. Relying on a "reliable" incremental
version number is not an option because sometimes 1 AWS region fails to receive a layer while others succeed.
Versions cannot be overwritten. Even if the process was perfect and would never fail, it is still possible
for AWS to launch a new AWS Region and the first layer version there would be out-of-sync with other
regions.
Preparing 3 PHP versions and publishing 2 layers (function and fpm) results in 6 layers that must be uploaded.
This can be a slow process and parallelization could help speed things up at yet another dependency cost.
Makefile is something available on anybody's computer and extremely versatile, but at the same time
slightly complex. Something like Taskfile could be easier to read, but with the cost of sacrificing
the universal availability of Makefile.

#### Dockerfile Stage and TDD Docker

The biggest driver for me to attempt replacing the existing Bref Runtime was the need to replicate / reproduce
layers for debugging purpose. Prior to opening a Pull Request on an Open Source project, I would like to
test my changes so I don't spend an open source maintainer's valuable time debugging something
I may have broken. I started designing the new Runtime with the mindset of easily reproducible on your
own AWS account as well as somewhat test-driven development of the Dockerfile.
I went through more than 7 implementations until landing on the current one and a helpful feature was
Docker build stage. It allows to build a small image, start a container with it and run a partial test.
A practical example used was to build a layer with just the PHP binary and then run a test that attempts
to execute a PHP file and grab the PHP version. If this test fails, we don't need to wait more time
until the entire image is built. It increases the feedback loop for the developer trying to debug
the inner engine of the Runtime. Once the first stage is built and approved we can move to other stages.
At the end of everything we can use an Alpine Linux image to install `zip` since we need to zip the entire
contents that we expect to upload to AWS.

#### Conclusion

I still believe that all my work may never really see the light of day. I never underestimated the amount
of work necessary to replace an existing system that is already in place and powering **billions** of PHP 
invocations. Recently I've been hopeful that the work is coming to an end and in reality it could very well
help developers all around the world run their workload on AWS Lambda even powered by Graviton2. In any case
I do not regret going down this rabbit hole as it increased my knowledge about so many pieces of technology
at a great level. The more limitations I hit from different vendors/providers/software, the more aware
about where they may fail I am and consequently a better developer I become.

Hope you enjoyed the reading. If you have any questions,
send them my way on [Twitter](https://twitter.com/deleugyn).

Cheers.
