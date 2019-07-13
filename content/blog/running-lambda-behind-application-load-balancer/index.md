---
title: Running Lambda Function behind an Application Load Balancer
date: "2019-07-13T14:30:10.284Z"
description: Saving costs for high-traffic services
---

When talking about APIs on Lambda Function, Api Gateway is the most common
subject. AWS did a pretty good job providing a service to run code (Lambda)
extremely decoupled from the reason that the code is being executed.
Lambda functions can be invoked by a handful of services such as SQS,
SNS, Api Gateway (API and Websites), S3, IoT, Aurora RDS, etc. Due to it's
low cost for execution and **no hourly cost**, Lambda with API Gateway makes
a perfect match for low-demand services. But Api Gateway can be more expensive
than a Load Balancer if you have enough requests. A Load Balancer isn't ideal
for low traffic APIs or websites because it has a 24/7 cost even when not
in use, which is roughly around $20/month. Api Gateway is much more *serverless*
in this aspect, charging only per request. If your request count is enough to
justify the minimum cost of the Load Balancer, this is how you can set it up.

```yaml
  Api:
    Type: AWS::Serverless::Function
    Properties:
      Role: !GetAtt LambdaExecutionRole.Arn
      CodeUri: .
      Handler: public/index.php
      Timeout: 30
      MemorySize: 1024
      Layers:
        - !Sub "arn:aws:lambda:${AWS::Region}:209497400698:layer:php-73-fpm:7"
      Runtime: provided
      VpcConfig:
        SecurityGroupIds: [!ImportValue DefaultSecurityGroup]
        SubnetIds: !Split [',', !ImportValue PrivateSubnets]

  GrantLoadBalancerInvokeToApi:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !GetAtt [Api, Arn]
      Action: lambda:InvokeFunction
      Principal: elasticloadbalancing.amazonaws.com

  HttpsListenerRule:
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Properties:
      Actions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup
      Conditions:
        - Field: host-header
          Values:
            - my.domain.com
      ListenerArn: !ImportValue HttpsListener
      Priority: 10
      
  Certificate:
    Type: AWS::CertificateManager::Certificate
    Properties:
      DomainName: my.domain.com
      DomainValidationOptions:
        - DomainName: my.domain.com
          ValidationDomain: my.domain.com
      ValidationMethod: DNS
      
  AttachCertificateToListener:
    Type: AWS::ElasticLoadBalancingV2::ListenerCertificate
    Properties:
      Certificates:
        - CertificateArn: !Ref Certificate
      ListenerArn: !ImportValue HttpsListener

  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      TargetType: lambda
      Targets:
        - Id: !GetAtt [Api, Arn]
```

The first non-obvious thing for me was the permission for the Load Balancer
to invoke Lambda. This is done on the resource `GrantLoadBalancerInvokeToApi`.
It authorizes the load balancer principal to invoke the specific lambda you
wish to place behind the Load Balancer. The HttpsListener and the TargetGroup
are the resources that links a Load Balancer to the Lambda function. If you
want to know more about setting up a Load Balancer for multiple lambda functions,
[check my previous post on this subject](https://blog.deleu.dev/one-load-balancer-to-rule-them-all/).

I've proposed a syntax for the AWS SAM support for Load Balancer 
[here](https://github.com/awslabs/serverless-application-model/issues/721#issuecomment-461555861)
so that it reduces the complexity of running a Lambda function behind
a load balancer. You may wish to subscribe to that issue if you want to be
notified when the syntax can be simplified.

That's it for this post. Follow me on [Twitter](https://twitter.com/deleugyn)
if you want to stay tuned on some of the tricks I learned while deploying
microservices on AWS written in PHP.

Cheers.