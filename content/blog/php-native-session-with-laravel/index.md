---
title: PHP Native Session with Laravel
date: "2020-10-18T13:10:48.284Z"
description: authenticating users against an existing $_SESSION
---

Completely replacing legacy code is hard. Rebuilding years of
work doesn't happen overnight. The strangler pattern offers
a strategy to slowly and gradually replace a legacy codebase
with a fresh one piece by piece. One of the key aspects of 
getting a successful project get started in replacing a legacy
codebase was keeping the authentication system working.
Let me set the scene.

#### Context

A 15-year old codebase written in PHP using `$_SESSION` across
several thousands of files lives behind a web server running
under `api.mydomain.com`. It acts as a backend api for a
frontend application that establish these API calls using
PHP browser's cookie (PHPSESSID). We can start a brand new project
and call it `api.v2.mydomain.com` or we can use path-based
routing and define an ALB routing on AWS where 
`api.mydomain.com/v2` automatically routes to this second project.
The new project can take advantage of modern php development
(a public framework, composer, tdd, etc), but developing
a brand new authentication system is a huge undertake at this
point. It requires changes to the authentication process,
the frontend and the backend project. A less disruptive
process would be to simply focus on making the v2 project
viable while keeping as much backward compatibility as possible.

#### Laravel Authentication with $_SESSION

[Laravel PHP Session](https://github.com/cgauge/laravel-php-session)
is a package that helps achieve exactly that. We install it
via `composer require customergauge/session` and configure
`auth.php` with a new authentication mechanism.

```php
    'guards' => [
        'php' => [
            'driver' => \CustomerGauge\Session\NativeSessionGuard::class,
            'provider' => \CustomerGauge\Session\NativeSessionUserProvider::class,
            'domain' => '.mydomain.com',
            'storage' => 'tcp://my-redis-address:6379',
        ],
    ],
```

The `domain` attribute is directly mapped to PHP's `COOKIE_DOMAIN`
ini setting and the `storage` is mapped to `SESSION_PATH`.

The next step is to implement the `UserFactory` interface
and bind it into Laravel's Container.

```php
final class NativeSessionUserFactory implements UserFactory
{
    public function make(array $session): ?Authenticatable
    {
        // $session here is the same as $_SESSION
        if (empty($session['id'])) {
            return null;
        }

        return new MyUser(
            (int) $session['id'],
            $session['my_session_attribute']
        );
    }
}
```

Laravel's auth middleware allows us to specify a `guard` when
declaring a route as authenticatable. See more on https://laravel.com/docs/8.x/authentication#protecting-routes.
As long as our route is protected by `auth:php`, the authentication
process will be triggered.

#### Test Code

Laravel already provides the `actingAs` functionality in
it's TestCase classes. That functionality will allow us
to bind a specific user object into the container so that
the application treat the execution test as authenticated.
To avoid any problems with the PHP Native session, we can also
bind a mock instance of SessionRetriever into the container

```php
    protected function setUp(): void
    {
        parent::setUp();

        $fakeSession = ['id' => 'my-fake-$_SESSION-content'];

        $this->app->instance(SessionRetriever::class, SessionRetriever::fake($fakeSession));
    }
``` 

This bind will prevent any execution path that might lead to
SessionRetriever calling `sesion_start()`, which would fail
under phpunit as there's no `headers` to be sent.

#### Conclusion

I've been using this strategy for well over 3 years now and
it empowered my team to strangle more and more our legacy
codebase into a modern application in small, incremental
steps. The more code we move to services designed with this
mindset, the less `$_SESSION` spread across a codebase we have
and the closer we get to having a consistent authentication
mechanism that might easily be replaced by another Laravel
Guard. As a matter of fact, we already have a [Laravel Guard
for AWS Cognito that I wrote about here](https://blog.deleu.dev/authenticating-aws-cognito-with-laravel/).

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.