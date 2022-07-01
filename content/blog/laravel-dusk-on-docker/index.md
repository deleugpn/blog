---
title: Laravel Dusk on Docker
date: "2022-07-02T01:01:17.813Z"
description: Reproducible environment without sacrificing visibility 
---

I used Laravel Dusk quite a lot when it was first released.
It was a great way to get started with automation testing.
Taylor always takes very good care of the developer experience
provided and I remember at the time it was quite a revolutionary
way to manipulate Google Chrome with an easy syntax.

One drawback that I always had is that I could never really
see the browser executing the actions I designed it to do.
This is because I don't run PHP on my own machine. I run it
on a Docker Container. I used to install google-chrome
on an Alpine Linux container and use docker-compose to
orchestrate it in such a way that Dusk could communicate
with the Chrome Driver running on a Docker service.

When tests failed, all I had was the last screenshot that
Dusk would take to represent the failed test. This wasn't
great, but I was kinda new to tests and I could somewhat
figure out where to go from there. 

As time went by I distanced myself from Dusk and got into a
lot more of PHPUnit testing. Recently one of my clients asked
me to help out with setting up Dusk on a Docker-based infrastructure.
However, he also wanted to be able to see the browser actions
for easy local development. He pointed me to the following
[Laracasts post](https://laracasts.com/discuss/channels/testing/possible-to-show-browser-window-for-dusk-tests-via-sail).
I looked into the `selenium/standalone-chrome` image and
I was baffled at how amazing it is.

First, it comes with Selenium & Google Chrome setup and
ready for use. Second, it provides VNC as a webserver so that
we can see what's going on with the browser running tests
by simply navigating to `localhost:7900`. Lastly it's super
easy to setup.

```yaml
version: '3.8'
  selenium:
    image: selenium/standalone-chrome:4.3
    volumes:
      - /dev/shm:/dev/shm
    ports:
      - "4444:4444"
      - "7900:7900"
```

This is all we need to add to a `docker-compose` file. Then
we open `DuskTestCase` to do some small adjustments:

```php
protected function driver()
{
    $server = 'http://selenium:4444';

    $options = (new ChromeOptions)->addArguments([
        '--window-size=1920,1080',
        '--whitelisted-ips=""'
    ]);

    $capabilities = DesiredCapabilities::chrome()
        ->setCapability(ChromeOptions::CAPABILITY, $options);

    return RemoteWebDriver::create(
        $server,
        $capabilities,
    );
}
```

Here we configure Dusk to connect to the Selenium container.
After this, we can go to `localhost:7900` and the VNC server
running inside the Selenium container will ask us for an
authentication. The default password is `secret`. Once that's
done, we can run our Dusk Tests and it will play out on
the VNC Server for us to watch what's going on with the tests.

This is such a game-changer if like me you run multiple projects,
multiple PHP versions, multiple setups on the same machine.
Because of this I always run everything inside Docker and
the `selenium/standalone-chrome` image brought back the ability
to watch out Google Chrome without sacrificing the ability
to isolate projects with their own reproducible setup.

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.