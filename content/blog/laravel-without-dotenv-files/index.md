---
title: Laravel without .env files
date: "2023-10-14T11:46:00.000"
description: A different approach for dealing with configurations
---

I worked with Laravel and Docker between 2016 and 2022. During all these
years I had a small papercut problem that involves environment variables.
I had to juggle between `.env`, `.env.testing`, `.env.dusk.{something}`,
`phpunit.xml`, `phpunit.xml.dist`, and real environment variables from
the Docker container. Jason describes it best: https://jasonmccreary.me/articles/laravel-testing-configuration-precedence/

It wasn't an everyday issue, but from time to time it would annoy me.
It was also common that whenever I would work on a Laravel project
that hired me to _dockernize_ it, the team would start to get some
suffering from defining environment variables for local, for local Docker,
for CI/CD, for Local Dusk, for CI/CD Dusk and for production.

I recently tried a novel approach for this problem and I have been loving
it for the last 3 months. So far it has been great, but it's still
early days to have a final decision on how great/bad it is.

As a side bonus, this approach allowed me to bypass `phpdotenv` performance
issue as reported here: https://github.com/vlucas/phpdotenv/issues/510.
The last thing I want is to be caching environment variables while
running phpunit tests all day (TDD) and then chasing invisible issues
caused by cached configuration.

#### What am I trying to accomplish?

I need to configure Laravel with the right configuration based on
the environment that Laravel is booting up. I want to avoid having
order of precedence and multiple ways to define these configurations
as to avoid having unexpected configuration creeping into the application.
The process needs to carer for, at minimum:

- Local Development
- Local PHPUnit
- Different developers (with or without Docker)
- PHPUnit on CI/CD (GitHub Actions and AWS CodeBuild)
- Production workload (AWS Lambda & AWS Fargate)

Defining real environment variables in Local, Docker, GH Actions, CodeBuild
and production are all different. I'm looking for something
that will work with minimum effort on these external systems.

#### The Environment Base Class

I decided to define a class with all the attributes that I need

```php
<?php declare(strict_types=1);

namespace Packages\Environment;

use InvalidArgumentException;
use Packages\Environment\Bags\Aurora;
use Packages\Environment\Bags\Cognito;
use Packages\Environment\Bags\Dynamodb;
use Packages\Environment\Bags\Logstash;
use Packages\Environment\Bags\Queue;
use Packages\Environment\Bags\RedisSession;
use Packages\Environment\Bags\S3;
use Packages\Environment\Bags\Vite;
use Packages\Laravel\Testing\Minio;

abstract readonly class Environment
{
    public RedisSession $session;

    public Logstash $logstash;

    public Aurora $aurora;

    public Dynamodb $dynamodb;

    public Cognito $cognito;

    public Queue $queue;

    public Minio $minio;

    public S3 $s3;

    public Vite $vite;

    public DynamicEnvironment $secrets;
}
```

With a base class like this, I can now extend it to build the environment
configurations that don't have to be dynamic and can be committed to
vcs.

```php
final readonly class MarcoLocal extends Environment implements Local
{
    public function __construct(
        public DynamicEnvironment $secrets,
        public Aurora $aurora = new Aurora(
            new AuroraConnection(
                driver: 'mysql',
                host: '127.0.0.1',
                port: 3306,
                username: 'root',
                password: '123456',
                database: 'my-project-main',
            ),
            new AuroraConnection(
                driver: 'mysql',
                host: '127.0.0.1',
                port: 3306,
                username: 'root',
                password: '123456',
                database: 'my-project-tenant-1',
            ),
        ),
        public RedisSession $session = new RedisSession(
            host: '127.0.0.1',
            port: 6379,
            domain: '.my-project.test',
            ssl: true,
        ),
        public Logstash $logstash = new Logstash(
            channel: 'single',
            tcp: '',
            udp: null,
            fallback: '',
        ),
        public Queue $queue = new Queue(
            export: new QueueConnection('sync'),
            import: new QueueConnection('sync'),
            webhook: new QueueConnection('sync'),
        ),
        public Minio $minio = new Minio(
            host: '127.0.0.1',
            port: 17811,
        ),
        public Cognito $cognito = new Cognito(
            userPoolId: 'local-fake-userPoolId',
            appClientId: 'local-fake-appClientId',
            privateKey: __DIR__ . '/Fixtures/jwt.phpunit.key',
            publicKey: __DIR__ . '/Fixtures/jwt.phpunit.pub',
            issuer: 'https://cognito-idp.local.amazonaws.com/phpunit-pool-id',
        ),
        public S3 $s3 = new S3(
            export: new S3Minio('export-bucket'),
            recordings: new S3Minio('recording-bucket'),
        ),
        public Vite $vite = new Vite(
            cdn: 'https://cdn.my-project.test',
            manifest: 'manifest.json',
        ),
    ) {}
}
```

To reproduce a new environment, we extend the base class
and fill out all information that we need. We can have several
of these files for team members and they are encouraged 
(but not needed) to be committed to VCS so that we have an overview
of how everyone is configuring their environment, whether it's 
Linux, Windows or Mac, with or without Docker, etc.

For the purpose of local development and CI/CD, we believe that
the MySQL secret information doesn't have to be a secret. We
install a local MySQL that is not exposed outside of our computers
(or use a MySQL container) and we use a standard username/password
for the application which our installation script creates for us.

The `Local` interface is empty and just helps us register some
local development routes in the RouteServiceProvider. Kind of
similar to how Laravel Dusk registers some login routes.

As for production, we have 3 production regions (EU, US and AU)
as well as staging environment. For those, we will have some
secrets.

Here is an example of a production one:

```php
#[IgnoreClassForCodeCoverage(CI::class)]
final readonly class EU extends Environment implements Aws
{
    public function __construct(
        public DynamicEnvironment $secrets,
        public Aurora $aurora = new Aurora(
            main: new AuroraConnection(
                driver: 'aurora',   // Custom Laravel Driver that will connect using password from AWS Secret Manager
                read: 'main.rds.my-project.internal',
                write: 'read.main.rds.my-project.internal',
                port: 3306,
                username: 'my-project-username',
                password: null,
                database: 'my-project-main-database',
                secret: 'my-project/eu/rds/main', // AWS Secret Manager identifier
            ),
            tenant: new AuroraConnection(
                driver: 'aurora',
                read: 'main.rds.my-project.internal',
                write: 'read.main.rds.my-project.internal',
                port: 3306,
                username: null,
                password: null,
                database: null,
                secret: 'customergauge/eu/rds/tenant',
            ),
        ),
        public RedisSession $session = new RedisSession(
            host: 'redis.cache.my-project.internal',
            port: 6379,
            domain: '.eu.my-project.com',
            ssl: true,
        ),
        public Logstash $logstash = new Logstash(
            channel: 'logstash',
            tcp: 'tcp://logstash.my-project.internal:9601',
            udp: 'udp://logstash.my-project.internal:9602',
            fallback: 'https://sqs.eu-west-1.amazonaws.com/XXXXXXXXXXXX/logstash-service-LogstashFallbackQueue-XXXXXXXXXXXXXX',
        ),
        public Queue $queue = new Queue(
            export: new QueueConnection(
                driver: 'sqs',
                region: 'eu-west-1',
                queue: 'https://sqs.eu-west-1.amazonaws.com/XXXXXXXXXXXX/export-queue-XXXXXXXXXXXXXX'
            ),
            import: new QueueConnection(
                driver: 'sqs',
                region: 'eu-west-1',
                queue: 'https://sqs.eu-west-1.amazonaws.com/XXXXXXXXXXXX/import-queue-XXXXXXXXXXXXXX'
            ),
            webhook: new QueueConnection(
                driver: 'sqs',
                region: 'eu-west-1',
                queue: 'https://sqs.eu-west-1.amazonaws.com/XXXXXXXXXXXX/webhook-queue-XXXXXXXXXXXXXX'
            ),
        ),
        public Cognito $cognito = new Cognito(
            userPoolId: 'eu-west-1_xxxxxxxxx',
            appClientId: 'xxxxxxxxxxxxxxxxxxxxxxxxxx',
            privateKey: null,   // AWS Cognito don't expose this information
            publicKey: 'https://cognito-idp.eu-west-1.amazonaws.com/eu-west-1_xxxxxxxxx/.well-known/jwks.json',
            issuer: 'https://cognito-idp.eu-west-1.amazonaws.com/eu-west-1_xxxxxxxxx',
        ),
        public S3 $s3 = new S3(
            export: new S3Bucket(
                region: 'eu-west-1',
                bucket: 'analytics-api-infrastructure-export-xxxxxxxxxxxxx',
            ),
            recordings: new S3Bucket(
                region: 'eu-west-1',
                bucket: 'XXXXXXXXXXXXXX-XXXXXXXXXXXXXX-XXXXXXXXXXXXXX',
            )
        ),
        public Vite $vite = new Vite(
            cdn: 'https://cdn.app.eu.my-project.com',
            manifest: 'manifest.eu.json',
        ),
    ) {}
}
```
#### Booting up the environment configuration

On my `bootstrap/app.php` file, I added the following lines:

```php
/*
|--------------------------------------------------------------------------
| My Project Environment Configuration
|--------------------------------------------------------------------------
|
| Let's make sure to detect which environment we're running on and set it
| up so that all the basic configuration such as database, queues, disk
| cache, session, etc. are properly configured for the current env.
|
*/

$environment = \Packages\Environment\Environment::fromEnvironment();

$app->singleton(\Packages\Environment\Environment::class);

$app->instance(\Packages\Environment\Environment::class, $environment);
```

Next, I need the `fromEnvironment()` method on my Environment base
class:

```php
public static function fromEnvironment(): self
{
    if (isset($_ENV['MY_PROJECT_ENV'])) {
        return self::make($_ENV['MY_PROJECT_ENV']);
    }

    if (is_file(__DIR__ . '/.env.php')) {
        $env = require __DIR__ . '/.env.php';

        if (! $env instanceof Environment) {
            dd('.env.php should return an instance of ' . Environment::class);
        }

        return $env;
    }

    $message = <<<'MESSAGE'
    Could not determine the environment. For production, set the MY_PROJECT_ENV
    environment variable. For local development, create a `.env.php` file in the
    `my-project/src/packages/backend/Environment` directory. See `.env.example.php`.
    MESSAGE;

    dd($message);
}
```

The first two lines of the method looks for a project-specific
environment variable. If it exists, we're running in a deployed
environment (production) and we will handle that with the `make`
method that can be seen below. Before I dive into that, let's look
at the `.env.php` file that is used for local development:

```php
<?php

if (defined('MY_PROJECT_PHPUNIT')) {
    return new \Packages\Environment\MarcoPhpunit(
        new \Packages\Environment\DynamicEnvironment(
            appKey: 'base64:uE6UZaCqrqHYxpWiYW+8RkL/jRtBVj/YtoqKfKyAdpE=',
            someThirdPartyApiKey: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxx',
        ),
    );
}

return new \Packages\Environment\MarcoLocal(
    new \Packages\Environment\DynamicEnvironment(
        appKey: 'base64:uE6UZaCqrqHYxpWiYW+8RkL/jRtBVj/YtoqKfKyAdpE=',
        // Fill this up if you want to test SSO locally.
        cognitoClientSecret: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxx',
        someThirdPartyApiKey: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxx',
    ),
);
```

This is a file that is git ignored since it contains secret information
and also will instantiate different classes since each developer will
have their own environment class setup.

The `MY_PROJECT_PHPUNIT` constant is one convenient way I found
to differentiate whether I'm running `phpunit` or if I'm running
the project locally. This constant is defined on my `phpunit.xml`
file and is the only configuration that exists there.

```xml
  <php>
    <const name="MY_PROJECT_PHPUNIT" value="true"/>
  </php>
```

The only difference between `phpunit.xml.dist` and `phpunit.xml` is
that the local version has been prepared to collect Test coverage
report in case the developer decides to enable pcov on their machine.

Both files are committed to VCS and don't need any special fiddling
around.

Lastly, let's look at the Environment class `make` method:

```php
final public static function make(string $environment): self
{
    return match ($environment) {
        'staging' => new Staging(DynamicEnvironment::fromEnvironment()),
        'eu' => new EU(DynamicEnvironment::fromEnvironment()),
        'us' => new US(DynamicEnvironment::fromEnvironment()),
        'au' => new AU(DynamicEnvironment::fromEnvironment()),

        default => throw new InvalidArgumentException(
            "Invalid environment [$environment]"
        ),
    };
}
```

In a live AWS environment (be it staging or production), we only
need to configure the secrets as true environment variables and
the `MY_PROJECT_ENV` variable. This means that for local development
and CI/CD, we have everything hardcoded and easy to find and read.
We don't need to configure environment variables in `.env`, Docker
containers, GitHub Actions or AWS CodeBuild. They're all pretty
much configured in a PHP file and for production we configure
only a few variables that are truly API keys, passwords or secrets.

```php
public static function fromEnvironment(): self
{
    if (! isset($_ENV['MY_PROJECT_APP_KEY'])) {
        dd('MY_PROJECT_APP_KEY is not set');
    }

    return new self(
        appKey: $_ENV['CUSTOMERGAUGE_APP_KEY'],
        cognitoClientSecret: $_ENV['COGNITO_CLIENT_SECRET'],
        SomeOtherApiService: $_ENV['SOME_OTHER_API_SERVICE'],
    );
}
```

#### Configuring Laravel with the Environment class.

Now the last piece of the puzzle is to get all these configurations
into their right place in Laravel. For that I have a `ConfigurationServiceProvider`
class which contains every configuration needed.

```php
<?php declare(strict_types=1);

namespace Packages\Environment;

use App\Application;
use Aws\SesV2\SesV2Client;
use Database\Connector\TenantContext;
use Illuminate\Console\Events\CommandStarting;
use Illuminate\Container\Container;
use Illuminate\Contracts\Mail\Mailer as MailerContract;
use Illuminate\Database\DatabaseManager;
use Illuminate\Events\Dispatcher;
use Illuminate\Foundation\Vite;
use Illuminate\Http\Request;
use Illuminate\Mail\Mailer;
use Illuminate\Mail\Transport\SesV2Transport;
use Illuminate\Support\ServiceProvider;
use Packages\Laravel\Inertia\LazyPropWatcher;
use Packages\Notifications\RegionalMailer;
use Psr\Log\LoggerInterface;
use Illuminate\Support\Facades\Gate;
use Packages\Authorization\PolicyGate;

final class ConfigurationServiceProvider extends ServiceProvider
{
    private Environment $environment;

    public function register()
    {
        $this->environment = $this->app->make(Environment::class);

        $this->registerApp();

        $this->registerCdn();

        $this->registerDatabaseMainConnector();

        $this->registerQueueConnections();

        $this->registerSessionConnection();

        $this->registerRedisConnection();

        $this->registerFileStorageDisks();

        $this->app->singleton(LazyPropWatcher::class);
       
        $this->registerNotifications();
    }

    private function registerApp(): void
    {
        $this->app['env'] = $this->environment->app;

        config()->set('app.debug', $this->environment->debug);

        config()->set('app.key', $this->environment->secrets->appKey);

        $this->app->resolving(Vite::class, function (Vite $vite) {
            $vite->useManifestFilename($this->environment->vite->manifest);
        });
    }

    private function registerCdn(): void
    {
        config()->set('app.asset_url', $this->environment->vite->cdn);

        $this->app->resolving(Vite::class, function (Vite $vite) {
            $vite->useManifestFilename($this->environment->vite->manifest);
        });
    }

    private function registerDatabaseMainConnector(): void
    {
        $this->app->resolving('db', function (DatabaseManager $manager, Application $app) {
            $manager->extend('main', function (array $config, string $name) use ($app) {
                /** @var \Illuminate\Database\Connectors\ConnectionFactory $factory */
                $factory = $app->make('db.factory');

                return $factory->make($this->environment->aurora->main->toConfig(), 'main');
            });
        });
    }

    private function registerQueueConnections(): void
    {
        config()->set('queue.connections.export', $this->environment->queue->export->toArray());
        config()->set('queue.connections.import', $this->environment->queue->import->toArray());
        config()->set('queue.connections.webhook', $this->environment->queue->webhook->toArray());
    }

    private function registerSessionConnection(): void
    {
        $region = $this->environment->region;

        // This is used by the CSRF Token.
        config()->set('database.redis.session', [
            'host' => $this->environment->session->ip,
            'port' => $this->environment->session->port,
            'database' => 0,
        ]);

        config()->set('session', [
            'driver' => 'redis',
            'connection' => 'session',
            'lifetime' => 120,
            'expire_on_close' => false,
            'encrypt' => false,
            'lottery' => [2, 100],
            'cookie' => "my-project-$region-session",
            'path' => '/',
            'domain' => $this->environment->session->domainForLaravelSession(),
            'secure' => $this->environment->session->secure,
            'http_only' => true,
            'same_site' => 'lax',
        ]);
    }

    private function registerRedisConnection(): void {}

    private function registerFileStorageDisks(): void
    {
        config()->set('filesystems.disks.export', $this->environment->s3->export->toConfig());
    }

    private function registerNotifications(): void
    {
        $this->app->bind(MailerContract::class, function (Container $container) {
            /** @var SesV2Client $client */
            $client = $container->make(SesV2Client::class);

            $option = [
                'ConfigurationSetName' => 'CustomerGaugeServiceConfigSet',
                'Tags' => [
                    ['Name' => 'service', 'Value' => 'my-project'],
                ],
            ];

            $transport = new SesV2Transport($client, $option);

            $mailer = new Mailer('regional', $this->app['view'], $transport, $this->app['events']);

            $mailer->setQueue($this->app['queue']);

            return $mailer;
        });
        
        $this->app->bind(RegionalMailer::class, function (Container $container) {
            $mailer = $container->make(MailerContract::class);

            $environment = $container->make(Environment::class);

            $logger = $container->make(LoggerInterface::class);

            return new RegionalMailer($mailer, $environment, $logger);
        });
    }

    public function boot(): void
    {
        /** @var Request $request */
        $request = $this->app->make(Request::class);

        $request->server->set('HTTPS', true);
    }
}
```

Since the `ConfigurationServiceProvider` depends on the `Environment`
class, everything is type-safe, which is a bonus.

#### Disabing phpdotenv

One of the first things Laravel do when the framework boot up is to
run a Bootstrapper that starts `phpdotenv` package. Since I don't
use `env()` function anywhere, I can completely skip that part. 
I do this in the `Kernel.php`:

```php


class Kernel extends HttpKernel
{
    public function __construct(Application $app, Router $router)
    {
        $this->bootstrappers = collect($this->bootstrappers)
            ->reject(\Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables::class)
            ->toArray();

        parent::__construct($app, $router);
    }

    // ...
}
```

The same thing can be done on Http Kernel and the Console Kernel.
With this change, Laravel will not be bootstrapping the Environment
variable package and I can 100% focus on my own environment variable
setup.

#### Conclusion

I've been running with this setup with my team for the past 3 months.
We have Mac, Linux, Docker and Windows users and so far we never
talked about this aside from the first days of setup. I take this
as a huge success because everybody just configured it once and
haven't spoken about it ever since.

A few changes has happened to the base `Environment` class over the
last couple of weeks. Whenever someone introduces a change there,
it's bound to break everybody else's setup as soon as we `git pull`.
But since everything is type-safe, we get a nice little 
`Argment of type \Packages\Environment\Bags\QueueConnection missing...`
and it's super easy to go into my personal file and set it up.

Automation tests processing time has also improved slightly because
Laravel doesn't need to run the `LoadEnvironmentVariables` process
and read several environment variables.

Overall, I'm extremely happy with it and I don't plan on going back,
however I am slightly worried about Laravel 11 skeleton changes.
Will there be some hidden breaking changes that will affect me?
If it does, I will blog about it and explain what broke and
what I did about it.

Cheers.