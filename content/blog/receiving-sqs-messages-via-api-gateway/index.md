---
title: Receiving SQS Messages via API Gateway Http API
date: "2021-02-27T13:02:17.157Z"
description: Highly available processing power for background APIs. 
---

API Gateway V2 (HTTP API) native integration with SQS is an incredible
tool to provide highly available APIs. AWS provides the scalability
of API Gateway while making sure that the HTTP payload being
received is automatically driven into an SQS queue. The drawback
is that the process automatically becomes asynchronous, so the
caller cannot expect a specific response.

There is a whole category of processes involving notification,
webhooks and/or data collection that is a match made in heaven
for this AWS offering. Imagine services like GitHub, Paddle,
Zapier, Zendesk, etc. The response you give back to these
services are incredibly limited. It's not like you can
return an error response and expect them to fix something
on their end. The most that they'll do is provide some retry
of the exact same payload. That means that if you want to take
data from an incoming payload, you must accept that any processing
error has to be handled by you. Redirecting these payload
directly to SQS means you can have a Dead Letter Queue to
hold failures and you can process the incoming messages at your
own pace. Another example of usage is for 
metrics/data collection/notifications type of data. I recently
used this pattern to provide an HTTP API for our frontend
team to write data into Elasticsearch and visualize it
on Kibana. Frontend makes an API call that is handled by API Gateway,
queued into SQS, worked by a Logstash worker that ingest the
contents of the message into Elasticsearch so that Kibana
can query it. A more interesting project that I also made
use of this relates to bots gathering temperature of several
hundreds of sensors and as the number of sensors deployed
increased, so did the AWS RDS bill. In order to drastically
reduce the RDS bill, we placed API Gateway with SQS in front
of the Lambda processing the sensor data to reduce the demand
into the RDS. The data is processed a bit slower while
drastically reducing the weight put into the RDS.

#### Deployment setup

The following template deploys an API Gateway V2 with a specific
domain, registers an ACM Certificate for said domain and
integrates it with an SQS Queue. Take a look at the entire
stack and then we'll walk through a few important points.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: Logstash API for Frontend

Parameters:
  Domain:
    Type: String

  HostedZone:
    Type: 'AWS::Route53::HostedZone::Id'
    
  UserPool:
    Type: String
    
  UserPoolClient:
    Type: String
    
Resources:
  LogstashHttpApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      FailOnWarnings: false
      DisableExecuteApiEndpoint: true
      CorsConfiguration:
        AllowOrigins:
          - 'https://*'
        AllowHeaders:
          - Authorization
          - Content-Type
          - Accept
          - '*'
        AllowMethods:
          - '*'
        MaxAge: 1
        AllowCredentials: true
      Domain:
        DomainName: !Ref Domain
        CertificateArn: !Ref LogstashCertificate
        EndpointConfiguration: REGIONAL
        Route53:
          HostedZoneId: !Ref HostedZone

  LogstashCertificate:
    Type: AWS::CertificateManager::Certificate
    Properties:
      DomainName: !Ref Domain
      DomainValidationOptions:
        - DomainName: !Ref Domain
          ValidationDomain: !Ref Domain
          HostedZoneId: !Ref HostedZone
      ValidationMethod: DNS
      
  LogstashPayloadRoute:
    Type: AWS::ApiGatewayV2::Route
    Properties:
      ApiId: !Ref LogstashHttpApi
      RouteKey: POST /
      Target: !Sub 'integrations/${LogstashPayloadRouteIntegration}'
      AuthorizerId: !Ref LogstashAuthorizer
      AuthorizationType: JWT

  LogstashAuthorizer:
    Type: AWS::ApiGatewayV2::Authorizer
    Properties:
      Name: logstash-authenticated-user-authorizer
      ApiId: !Ref LogstashHttpApi
      AuthorizerType: JWT
      IdentitySource:
        - '$request.header.Authorization'
      JwtConfiguration:
        Audience:
          - !Ref UserPoolClient
        Issuer:
          Fn::Sub:
            - https://cognito-idp.${Region}.amazonaws.com/${UserPool}
            - Region: !Ref AWS::Region
              UserPool: !Ref UserPool

  LogstashPayloadRouteIntegration:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId: !Ref LogstashHttpApi
      Description: Proxy incoming HTTP Payload into Logstash SQS
      IntegrationType: AWS_PROXY
      IntegrationSubtype: SQS-SendMessage
      PayloadFormatVersion: '1.0'
      CredentialsArn: !GetAtt LogstashHttpApiRole.Arn
      RequestParameters:
        QueueUrl: !Ref LogstashQueue
        MessageBody: $request.body # Send the body of the HTTP request into SQS

  LogstashQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 1800
      MessageRetentionPeriod: 604800
      ReceiveMessageWaitTimeSeconds: 20
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt [LogstashDeadLetterQueue, Arn]
        maxReceiveCount: 5

  LogstashDeadLetterQueue:
    Type: AWS::SQS::Queue
    Properties:
      MessageRetentionPeriod: 1209600

  LogstashHttpApiRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: apigateway.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: ApiWriteToSQS
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              Action: sqs:SendMessage
              Effect: Allow
              Resource: !GetAtt LogstashQueue.Arn
```

##### The SQS Queue

The following snippet is responsible for creating an SQS
queue to hold the incoming HTTP payload and a Dead Letter Queue
to hold messages that failed to be processed. Keep in mind
that the worker of an SQS message cannot take longer than
the `VisibilityTimeout` to process a message, otherwise
the worker runs the risk of taking the same message 
multiple times. We can adjust the number of times the message
is allowed to fail in the `maxReceiveCount`.

```yaml
  LogstashQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 600
      MessageRetentionPeriod: 604800
      ReceiveMessageWaitTimeSeconds: 20
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt [LogstashDeadLetterQueue, Arn]
        maxReceiveCount: 5

  LogstashDeadLetterQueue:
    Type: AWS::SQS::Queue
    Properties:
      MessageRetentionPeriod: 1209600
```

##### The Api Gateway

I was not able to deploy this using `FailOnWarnings: true`
and to be honest I didn't think the warning was relevant.
I wanted to DisableExecuteApiEndpoint because I wish to
have this API Gateway available only via my own domain.
I also attempted to enable CORS for the specific domain
that I was working on, unfortunately without much luck.
At least I managed to force the origin to be HTTPS.
For `AllowHeaders`, it seems that `*` is not working,
so I ended up having to list the specific headers I'm looking
to use. Keep in mind that this is a fairly new resource
and AWS usually evolves resources over time.

```yaml
  LogstashHttpApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      FailOnWarnings: false
      DisableExecuteApiEndpoint: true
      CorsConfiguration:
        AllowOrigins:
          - 'https://*'
        AllowHeaders:
          - Authorization
          - Content-Type
          - Accept
          - '*'
        AllowMethods:
          - '*'
        MaxAge: 3600
        AllowCredentials: true
      Domain:
        DomainName: !Ref Domain
        CertificateArn: !Ref LogstashCertificate
        EndpointConfiguration: REGIONAL
        Route53:
          HostedZoneId: !Ref HostedZone
```
##### The Authorizer

Since this is supposed to allow the frontend team to write
logs into Elasticsearch, API Gateway was a good choice, but
I wanted to be careful with a publicly available API that
could be called from anywhere in the world. I borrowed
the application authentication so that the frontend will
only be able to write logs if there is an authenticated
user attached to the call. We're not really interested
in user data for this, so I didn't try to extract anything
from the JWT token itself. I only made sure that the token
is valid before redirecting the payload into SQS. Guest areas
of the system are currently not able to write frontend logs
into Kibana, but we don't really need it.

```yaml
  LogstashAuthorizer:
    Type: AWS::ApiGatewayV2::Authorizer
    Properties:
      Name: logstash-authenticated-user-authorizer
      ApiId: !Ref LogstashHttpApi
      AuthorizerType: JWT
      IdentitySource:
        - '$request.header.Authorization'
      JwtConfiguration:
        Audience:
          - !Ref UserPoolClient
        Issuer:
          Fn::Sub:
            - https://cognito-idp.${Region}.amazonaws.com/${UserPool}
            - Region: !Ref AWS::Region
              UserPool: !Ref UserPool
```

##### The Route

For this project we only needed one route and I decided to 
make it `POST /`, which is the root level of the domain.
The route is protected by the Authorizer explained above
and will target a `integrations` with a Route Integration.

```yaml
  LogstashPayloadRoute:
    Type: AWS::ApiGatewayV2::Route
    Properties:
      ApiId: !Ref LogstashHttpApi
      RouteKey: POST /
      Target: !Sub 'integrations/${LogstashPayloadRouteIntegration}'
      AuthorizerId: !Ref LogstashAuthorizer
      AuthorizationType: JWT
```

##### The SQS Integration

The Integration is where we specify which SQS Queue will receive
the payload and how the MessageBody of SQS will look like.
AWS has this `$request.body` syntax to represent the entire
body of the HTTP Request. We also need to provide an IAM Role
to authorize API Gateway (an AWS service) to write into a
specific queue located in my private AWS account.

```yaml
  LogstashPayloadRouteIntegration:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId: !Ref LogstashHttpApi
      Description: Proxy incoming HTTP Payload into Logstash SQS
      IntegrationType: AWS_PROXY
      IntegrationSubtype: SQS-SendMessage
      PayloadFormatVersion: '1.0'
      CredentialsArn: !GetAtt LogstashHttpApiRole.Arn
      RequestParameters:
        QueueUrl: !Ref LogstashQueue
        MessageBody: $request.body # Send the body of the HTTP request into SQS
```

#### Conclusion

This post focus solely on the part of getting HTTP payload
into SQS so that the API is highly available while the processing
time can be phased out according to the availability of
internal resources. Sometimes extremely high volume of 
incoming data might force you to increase your costs on
services like AWS RDS or AWS Fargate while in reality
you don't really need that high speed. An extreme high
cost reduction is a good motivator to process the data
a little bit slower. Once the payload is in SQS, it's up
to your use case whether a Fargate Container will work it
or an AWS Lambda. For the frontend case I described, I have
Logstash as a Fargate Container 
(see https://blog.deleu.dev/el4k-my-journey-through-aws-elk-stack/)
while for the temperature sensor collector I have an AWS Lambda
function as the worker of SQS.

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.