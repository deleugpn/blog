---
title: Deploying Laravel Artisan on AWS Lambda
date: "2019-07-14T20:37:14.284Z"
description: How to run Laravel Artisan commands using Bref
---

#### Introduction

Working with Laravel it's only natural to need `artisan` for a few tasks.
Let it be custom commands tailored to your application or default commands
like `php artisan migrate`, knowing how to use cli-based scripts on Lambda
is an important step to gain confidence to go serverless with a project.
One might argue that you can run `migrate` from your own local machine,
but if your database is locked behind AWS VPC with private connectivity only
you'll have to rely on a tunnel (bastion, jumpbox) so that your local machine
connects to an instance running within a trusted location to your private RDS.
Where I work, we enforce a security rule that a bastion host will only be created
in emergency situations and shall be discarded as soon as possible. Furthermore,
not everybody on the development team have access to our AWS Production Account.
Running database migrations, on the other hand, can be part of a daily or weekly
release and the security requirement would make it a constant annoyance if we
wanted to run it from our local machines.

Running Artisan from a Lambda Function is a great mechanism to maintain the
security rules as well as allowing developers to keep their autonomy for
releases.

#### Implementation

The AWS SAM template for the Lambda Function will look like the following
snippet.

```yaml
  Listener:
    Type: AWS::Serverless::Function
    Properties:
      Role: !GetAtt LambdaExecutionRole.Arn
      CodeUri: .
      Handler: artisan
      Timeout: 60
      MemorySize: 1024
      Environment:
        Variables:
          ARTISAN_COMMAND: 'my:command'
      Layers:
        - !Sub "arn:aws:lambda:${AWS::Region}:209497400698:layer:php-73:6"
      Runtime: provided
      VpcConfig:
        SecurityGroupIds: [!ImportValue DefaultSecurityGroup]
        SubnetIds: !Split [',', !ImportValue PrivateSubnets]
```  

The important aspects to notice in this snippet are the Handler pointing to 
the artisan file, the environment variable defining the command signature
to invoke, the VPC configuration that grants the ability to establish a 
connection to the RDS and the layer. 

My first (and most obvious) attempt at running artisan commands started
by defining a handler such as `artisan migrate`. The innocent child inside
of me got hit hard by a [validation rule on the handler attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-function.html#cfn-lambda-function-handler)
that states that a ` ` (space character) is not allowed. My workaround
to it brings us to the environment variable and changing the the `artisan` 
file slightly to accommodate. This is the change made:

```php
$command = $_ENV['ARTISAN_COMMAND'] ?? null;

$status = $kernel->handle(
    $input = new Symfony\Component\Console\Input\ArgvInput($command ? ['lambda', $command] : null),
    new Symfony\Component\Console\Output\ConsoleOutput
);
```

When the Lambda Function gets invoked, the environment variable will be used
to indicate to the Laravel Kernel which command is being executed. 

The Layer comes from [Bref](https://bref.sh) which provides the PHP binary
necessary to run PHP on a Lambda Function.

Next relevant part is the command itself. Here's an example of one:

```php
<?php

namespace App\Modules\MyModule\Console;

use Illuminate\Console\Command;

final class TestCommand extends Command
{
    protected $signature = 'my:command';

    public function handle(): void
    {
        $callback = function (array $event) {
            logger()->info(json_encode($event));
        };
        
        lambda($callback);
    }
}
```

The `lambda()` global function comes from [Bref](https://bref.sh) as well.
You can install it with `composer require bref/bref` and the function will
be available. The `$event` array will contain any parameters sent to the 
Lambda during the invocation.

It's important to note that you shouldn't register any Laravel-provided
commands directly with this approach. Instead, you should write a wrapper
that calls the `lambda()` helper. This is because AWS will only consider
your execution successful if there is a callback to the internal services
letting AWS know that your execution was successful. Bref does this for us
once the registered callback finishes. Here is an example of how to run
migrations on your project:

```php
<?php

namespace App\Modules\Migrations\Console;

use Illuminate\Console\Command;
use Illuminate\Database\Console\Migrations\MigrateCommand;
use Illuminate\Foundation\Console\Kernel;

final class LambdaMigrateCommand extends Command
{
    protected $signature = 'lambda:migrate';

    public function handle(Kernel $artisan): void
    {
        $callback = function (array $event) use ($artisan) {
            $artisan->call(MigrateCommand::class);
        };

        lambda($callback);
    }
}
```

#### Conclusion

Bref does most of the heavy-lifting for us when running PHP on AWS Lambda.
With minimum changes, we're able to let Artisan bootstrap our Laravel app
the way it always does and take full advantage of the container dependency
injection on an Artisan-registered command. The only thing to always keep
an eye is the use of `lambda()` helper so that Bref can notify Lambda of
the successful execution process. Without it, your function will be in an
infinite bootstrap state until your Lambda timeout is reached.

Hope this post helps any artisans out there. Don't forget to hit me up
on [Twitter](https://twitter.com/deleugyn) with any follow-up.

Cheers.