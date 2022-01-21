---
title: PHP for PHP Recruiters
date: "2022-01-21T13:35:28.000Z"
description: A summary of words that could come up while interviewing a PHP candidate. 
---

Recently I saw a post on LinkedIn about a recruiter that is primarily working on the PHP Market
asking for input on what are important factors to know about PHP. That question made me dive
deep into myself because I wasn't sure how to even begin to answer it. It's rather _easier_
to describe and teach about things for people with a common goal. If you want to learn PHP,
you look for existing professionals so that you can learn what they do. However, globalization
brought a lot of crossover into our modern world. I know close to nothing about recruitment
and it is an intriguing challenge to figure out how much is important for someone that will
not work or write PHP code. This post is me attempting something I never did before: explain
PHP to a recruiter from my point of view. 

> This post will detail my personal view and I can be factually wrong or partially wrong.
> Still, it's how I perceive the world I work in and can be at least an interesting read
> even if you disagree with me.

#### The Language

PHP came to existence in the '90s when Rasmus wanted to write a Personal Home Page. Since I was 
a baby back then I'm going to skip quite a large history as I don't feel confident to talk about
it. I met PHP on version 5, and I vaguely remember the existence of version 4. For a senior engineer
with 12 years of experience today, the universe of PHP largely revolves around PHP 4, 5, 7 and 8.
Version 6 was due to launch, but never really made it to production. The short story I was told about
it is that since books came out about "PHP 6", it was better to launch the next version as PHP 7 as to
not confuse anyone reading a PHP 6 book. Imagine PHP 7 actually being called PHP 6 and someone reading
the book not understanding what is going on. An important part of the work done for PHP 6 was released
under PHP 5.4. PHP 5.6 was the last in the PHP 5 series and ended it's life in December 2018, bringing
to an end the longest PHP series in my career time (since 2009). The language has stabilized it's release
cycle for several years now and is due to launching a new minor version every year and a new major
version every 5 years. If this does not change, we are expected to have PHP 8 series going up to 
PHP 8.4 with a release schedule for December 2024 and the rise of PHP 9 in the winter of 2025.

#### Frameworks & Community Ecosystem

I imagine (hope?) it would not be hard to convince people that Composer is probably the most important
factor in the history of PHP community ecosystem. Every programming language has a Package Manager which
is widely used to distribute open source packages that help you do something on that language.
Python has pip, Ruby has Gem, Javascript/Node has NPM, Rust has Cargo and PHP has Composer. Although we're
probably heavily biased, a lot of PHP engineers will say that Composer is by far one of the best package
managers a language could have. I had a couple of hours of experience with Rust and found Cargo to be
interesting, but Composer (and the large OSS community) is definitely a key point in PHP.

Zend 1 is the oldest framework my career time allows me to remember. At the time I learned/was told that
Zend was for enterprise-grade software, very complex and extremely hard. Everyone was better off writing
Vanilla PHP, which is why a lot of companies with 12+ years of age today had/have a legacy in-house framework.
Laravel and Symfony became the largest frameworks in the ecosystem, followed by some small players like
Cake, Yii, CodeIgniter, Slim, Laminas (successor of Zend) and a few others. 

Given it's history of vanilla / each company it's own standard background, PHP had an important innovation
with the rise of the PSR Working Ground (PHP Standards Recommendations). They started by defining standards
on how to organize file structure, coding style (spaces vs tabs war, etc), logging, etc. They grew to
defining highly important (controversial) standards such as Caching, HTTP and more. Like it or not,
they brought some maturity level to PHP to the point that enterprises like AWS, Google or Elastic could
write some PHP open source packages (SDK, infrastructure, etc) based off of PSR.

#### Popularity

Imagine working in the 2000~2010 era in the software industry and facing PHP. (I can only imagine because
my professional career only started in 2010). Going from company to company and having to learn PHP all over
again because everyone would do things differently. From what people use to write back then the language
didn't seem to be the safest. One could argue that a lot of the backlash PHP received was somewhat concerning.
PHP 7 and PHP 8 Series, combined with Composer, PSRs and a small number of highly successful frameworks
brought a lot of stability, security and maturity to the PHP world. We usually see people saying that
if you complain about PHP but hasn't touched it since 2012, your opinion is highly outdated due to how
much the language grown. From my small bubble/perception of the world, I would place a large part of PHP
success to be around WordPress and how easy it was for Web Hosting companies to provide website hosting
powered by PHP. It created a lot of SysAdmin jobs and a lot of small companies could provide physical
machines for renting at an age where Cloud Computing was just an idea. As for developers, PHP has always
been very forgiving an easy to get started / learn. Unfortunately I don't know about what it's like to
learn PHP in 2022, but I know that learning it in 2009 was extremely easy because I didn't need to bother
with so much complexity that shapes the Software Industry today. I just had to write something like

```php
<some html here>
<some more html>
<?php echo "Hello", $_SESSION['first_name'] ?>
</some more html>
</some html here>
```

and I had a working website in PHP. Something like this would still work to this day, but usually with a
different context like Laravel Blade or Symfony Twig, which bring some learning curve (Composer, Templating, etc).

#### Internals

It is known as PHP Internals where the development of the language itself happens. You can read the
discussion openly at https://externals.io. The language is composed by a democratic process called PHP RFC.
A change proposed to the language has to be approved by 2/3 of the people voting. Voting rights are given
on a case-by-case basis provided the person has proven to be an important member of the continuity of
existence of PHP. Anybody in the world can participate in internals debates and/or propose changes to
the language. If accepted, the change is merged into the source code of PHP and will be shipped at
the appropriate time with the help of Release Managers. Bugfixes does not go through the RFC process
and can also be proposed by anyone in the world. There is no corporation that actually maintains
PHP (like Oracle for Java or Microsoft for Typescript). The most corporations can do is donate money
to the newly founded [PHP Foundation](https://opencollective.com/phpfoundation).

#### Tooling

PHP gained a lot of tooling to help improve it's maturity level. There is an incredibly large amount of
tools that can help developers work with PHP everyday. From my extremely limited perspective and point
of view, I would say that at least knowing the existence of the following tools is important:

- [Composer](https://getcomposer.org/): PHP Package Manager
- [PHPUnit](https://phpunit.de/): PHP Testing Framework
- [PHPStan](https://github.com/phpstan/phpstan): PHP Static Analysis Tool
- [PHPStorm](https://www.jetbrains.com/phpstorm/): JetBrains IDE offering for PHP
- [Rector](https://github.com/rectorphp/rector): Automated Refactoring of PHP Code
- [Psalm](https://psalm.dev/): PHP Static Analysis Tool
- [Bref](https://bref.sh): Serverless PHP
- [ReactPHP](https://reactphp.org/): Event-Driven Async I/O PHP (not to be confused with [reactjs](https://reactjs.org/))
- [PHPCSFixer](https://github.com/FriendsOfPHP/PHP-CS-Fixer): PHP Coding Standards Fixer

#### Conclusion

I tried to reflect on how my conversations go with recruiters and how sometimes I end up talking crazy
things because I forge they're not part of my context. Sometimes I find it a hard balance because I never
want to offend anyone by assuming they don't know something, specially women recruiters. On the other hand,
I only assume they don't know it because I wouldn't expect it to be a requirement for them to do their job.
It's not like we could look at the past 500 years and learn how it was done before. Tech recruitment nowadays
seem like such an impossible task. If a recruiter wants to go above and beyond and learn a bit about PHP,
I hope my post can be helpful.

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. For this post I will also link [LinkedIn](https://www.linkedin.com/in/deleu/) as it may be a 
better/easier platform given the audience.

Cheers.