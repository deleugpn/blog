---
title: A monorepo approach to larger modules in Laravel and Lambda
date: "2021-02-28T16:57:48.021Z"
description: The mindset of modularized applications applied to monoliths. 
---

I work with a large enterprise application that's broken
down into a few teams, languages and modules. As I started
finishing one module and moved to another, the process of
going through setting up a brand new "micro-service", managing
CI/CD, cross-region deployments, application configurations and
other things started to burden me with boring repetitive
work with little benefit. Modularizing an application is
still important even though completely breaking it apart
might not necessarily be a day-one activity.

Recently I've been working on the part of the application
that will be receiving data. APIs, Webhooks, File upload,
Integrations, etc. As I knew I'd be taking one module at
a time and each of them is a large beast of it's own with
a common purpose, I took a stand in favour of macroservice
over microservice. I started one Laravel repository that
would be the center of bringing this all together. However,
for the sake of a possible fragmentation of modules in the 
future, I decided to put each module in it's own isolated
domain. This helps with the fragmentation of the frontend
team (not the same team handling all of these modules)
and goes on a best-effort to isolate them away from being
affected by a possible fragmentation that one day may come.


#### Source Code Structure

The source code structure differs greatly from a regular
Laravel application. Instead of breaking things down by
what they are, the structure focus more on separating
things by what they do.

- Library
  - Eloquent
  - Packages
      - Authentication
      - Logstash
      - Queue
      - Storage
- Modules
  - Entities
  - Files
  - Integrations  
  - Webhooks
    
The Library folder is where shared code lives. If someone
tries to extract a module to it's own microservice, the entire
"Library" folder can be replicated there. Perhaps it could even
be a private Composer package if we had a lot more PHP engineers.
With the exception of some Eloquent models that wouldn't need
to be replicated, using source code from the Library package
has to be in a more agnostic mindset. For example, the authentication
mechanism is irrespective of which module the user is currently
interacting with. Same goes for writing logs into Logstash
or uploading files into S3.

#### RouteServiceProvider

The RouteServiceProvider can register multiple routing files
and group them by domain. This will allow certain endpoints
to be available only under a certain domain. That way if we
have collision between modules, we can still replicate the
same names without leaking backend details to the frontend.

```php
    private function mapEntitiesRoutes()
    {
        Route::group([], base_path('routes/entities.php'));
    }

    private function mapWebhooksRoutes()
    {
        Route::domain("hooks.mydomain.com")->group(base_path('routes/webhooks.php'));
    }

    private function mapFilesRoutes()
    {
        Route::domain("files.api.mydomain.com")->group(base_path('routes/files.php'));
    }

    private function mapIntegrationsRoutes()
    {
        Route::domain("integrations.api.mydomain.com")->group(base_path('routes/integrations.php'));
    }
```

#### Feature Tests

Feature Testing specific domains is pretty easy and can be
done in a more automated manner. The folder structure kind
of mimics the original source code:

- Tests
  - Features
    - Entities
    - Files
    - Integrations
    - Webhooks

Each Module folder can hold it's own `{Module}TestCase` that
basically consist of the following code snippet

```php
<?php declare(strict_types=1);

namespace Tests\Feature\Files;

use Tests\Components\FeatureTestCase;

abstract class FilesTestCase extends FeatureTestCase
{
    protected function prepareUrlForRequest($uri)
    {
        $uri = trim($uri, '/');

        $uri = 'https://files.api.mydomain.com/' . $uri;

        return parent::prepareUrlForRequest($uri);
    }
}
```

By overriding the `prepareUrlForRequest` method, we can
continue to use Laravel's function `$this->get()`, `$this->post()`
`$this->put()` and `$this->delete()` as we would on a regular
Laravel application, without having to specify any information
about the domain.

#### The Lambda deployment

There are two ways to handle the infrastructure for deployment
of this strategy: separate lambdas or grouped lambdas. The
choice of which strategy to use depends heavily on the Lambda
itself. Let's recap a Lambda definition:

```yaml
  Api:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: my-macroservice-name
      CodeUri: .
      Handler: public/index.php
      Timeout: 30
      MemorySize: 768
      Role: !GetAtt LambdaExecutionRole.Arn
      Environment:
        Variables:
          LOGSTASH_PROCESSOR: http
      Layers:
        - !Sub "arn:aws:lambda:${AWS::Region}:209497400698:layer:php-80-fpm:6"
        - !Sub "arn:aws:lambda:${AWS::Region}:403367587399:layer:gmp-php-80:7"
        - !Sub "arn:aws:lambda:${AWS::Region}:403367587399:layer:redis-php-80:7"
      Runtime: provided.al2
      VpcConfig:
        SecurityGroupIds: ...
        SubnetIds: ...
```

Looking at this configuration, we can come up with some questions:
- Is there a particular module that requires more CPU/Memory/Timeout?
- Is there any need for module-specific environment variable that cannot be shared?
- Is there a necessity for isolating IAM permission between modules?
- Do we need Lambda Metrics separated per module?

More often than not we can keep it simple and share the same
Lambda for all modules. The way we do that with a Load Balancer
is with the following snippet:

```yaml
  Api:
    Type: AWS::Serverless::Function
    Properties:
      ... 
    
  GrantLoadBalancerInvokeToApi:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !GetAtt [Api, Arn]
      Action: lambda:InvokeFunction
      Principal: elasticloadbalancing.amazonaws.com

  TargetGroupForApi:
    DependsOn: [GrantLoadBalancerInvokeToApi]
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      TargetType: lambda
      Targets:
        - Id: !GetAtt [Api, Arn]

  HttpsListenerRule:
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Properties:
      Actions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroupForApi
      Conditions:
        - Field: host-header
          Values:
            - integrations.api.mydomain.com
            - files.api.mydomain.com
            - webhooks.api.mydomain.com
      ListenerArn: !ImportValue HttpsListener
      Priority: 10

  # Repeat this for every domain
  FilesDns:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneId: !ImportValue HostedZone
      Name: files.api.mydomain.com
      Type: CNAME
      ResourceRecords:
        - !ImportValue LoadBalancerDns
      TTL: 3600
```

This will grant permission for AWS ALB to execute a specific
Lambda function in my own account, setup the Target Group
for the Lambda and attach a few domains on a new 
HttpsListenerRule to a shared Load Balancer. If you're curious
about the `!ImportValue HttpsListener` aspect, check out
[One Load Balancer to rule them all](https://blog.deleu.dev/one-load-balancer-to-rule-them-all/).
Lastly, we also need to register the DNS for each service.

_Side note_: I usually put the DNS registration separate from
the Lambda deployment, that is in a dedicated template just
for the DNS. The reason for that is that  I would be able
to separate an entire module into it's own source code
repository without having to cause a downtime when deleting
the DNS from one template and recreating it in another.
Hopefully CloudFormation Import Resources will improve
to allow this sort of thing to not be an issue.

#### Conclusion

With this setup I can focus more of my time on coding and
less of it on AWS infrastructure for an entire microservice
everytime a new separate module needs to be developed. Yes,
I do run the risk of breaking the separation and coupling
code of one module to code of another module, but we try
on a best-effort to avoid that. The goal is to make it as
easy as possible to fragment the project later on if needed
but without sacrificing too much of effort on a promise of
something that may never happen. This allows me to reuse
the CI/CD strategy already in place while sacrificing the
ability of deploying one module without deploying the other.
As things progresses and the team grows, we might get some
developers to be dedicated on a specific module and hopefully
it won't be hard to separate it and let each team be on
their way.

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.