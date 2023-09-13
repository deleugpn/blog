---
title: A Powerful Webhook Offering
date: "2021-12-23T21:50:36.523Z"
description: Sending out webhooks with Laravel HTTP Client 
---

Webhooks are a common integration mechanism between systems. A small detail I don't enjoy about them is
their ambiguity, so let me start by specifying what I'll be talking about. At my organization we call
them Outgoing Webhooks when we are a Webhook Provider and we're sending out our system's data out.
And we call Incoming Webhooks when we're the receiver of data from other systems. This post will describe
my journey around being a Webhook Provider.

#### The Context

When talking about providing real-time data out of the system, we have a few scope definition established
upfront. It's a process that goes well with Pub/Sub systems where we internally notify that an event
happened and have a subscriber loop through all webhooks configured to send out the event.
Webhooks can have different forms depending on whether the project is developer-facing like Algolia, GitHub,
AWS or Netlify. In my context, the goal is to provide Outgoing Webhooks for business people that are
looking to save Engineering time and hook things up themselves as easily and simple as possible. This means
that it's more important to be able to send out data to off-the-shelf API solutions. Nobody is going to be
writing custom code on the receiving end to be able to receive my webhook calls. The business value is
that users can request our system to send one of their other system a piece of data in real time.

#### The Stack

The ingredients for this project are:

- Pub/Sub
- HTTP Client
- Payload Transformation
- 3rd-party Authentication

For the sake of understanding, let's give a concrete goal for the pub/sub: Everytime a user logs in, we should
be able to notify another system. The event here may be meaningless in a lot of contexts, but the
implementation for other events would roughly be the same. With the event published, we can then
prepare an HTTP Client so that we may perform an authentication on a 3rd party system and send
an API call providing the data configured. Here the most powerful tech decision I made was to support
OAuth2. This means a business user can log into our platform, fill out an OAuth2 Form which looks
exactly like Postman OAuth2 configuration and configure an endpoint from a system like Microsoft Dynamics
to be the receiver of the Webhook call. Microsoft Dynamics doesn't need to be prepared to be a Webhook
Receiver from my small company. Their public native REST HTTP APIs will do.

#### The Flow

We first identify which part of our system  is responsible for fulfilling the event and then publish 
that as an event. We may use Laravel Pub/Sub or something else. In my case we opted for AWS SNS.
Once the subscriber kicks in we need to load all webhooks stored in our database and trigger one by one.
This is a simple data query and looping.

```php
    $webhooks = $this->configuration->newQuery()->where('enabled', 1)->get();

    foreach ($webhooks as $webhook) {
        $this->runEventThroughWebhook($event, $webhook);
    }
```

For a great User Experience, I like to create a record of the webhook execution prior to starting the
actual execution. This means that we can wrap the execution on an eager try/catch and awlays
update the delivery execution with how it went. If something goes wrong on the remote server,
we can record the 400/500 so that our users can beware of what's happening.

```php

    private function runEventThroughWebhook(EloquentModel $event, WebhookConfiguration $webhook)
    {
        $delivery = $this->delivery->newQuery()->create([
            'webhook_id' => $webhook->id,
            'status' => 'processing',
            'request_method' => $webhook->method,
            'request_url' => $webhook->url,
        ]);

        $bag = $this->executeWebhook($webhook, $event);

        $delivery->update([
            'status' => $bag->status,
            'request_body' => $bag->body,
            'response_status' => $bag->responseStatus,
            'response_body' => $bag->responseBody,
        ]);
    }
```

To actually execute the call to an external system, we'll need an HTTP Client. The beauty here is to
either factory a clean standard HTTP Client or, if needed, factory an HTTP Client with an OAuth2 Bearer
Token already configured. We can do that roughly in the following way: 

```php
        if ($webhook->relationLoaded('oauth2') && $webhook->oauth2) {
            try {
                $client = $this->httpFactory->oauth2($webhook->oauth2);
            } catch (RequestException $exception) {
                return WebhookExecutionBag::auth($exception->response);
            }
        } else {
            $client = $this->httpFactory->client();
        }
```

Here we're checking if there is a OAuth2 relation on the webhook configured and if so we try to
load a Bearer Token to be used in the Authorization Header. But if the authentication fails, we can
already return a failed webhook delivery. If there is no OAuth2 configured, we can deliver a regular
API call.

If a customer wants our system to make an API call with a signature for verification (much like GitHub does)
we can easily allow for that with the following snippet:

```php
if ($webhook->secret) {
    $headers['X-ACME-SIGNATURE'] = hash_hmac('sha256', $body, $webhook->secret);
}
```

PHP's hash_hmac will compute a signature of the array `$body` using the user's secret if they provided one.
Here I stored the user provided secret [encrypted with AWS Secret](https://blog.deleu.dev/swapping-laravel-encryption-with-aws-kms/).

#### The Body of the Request 

The most interesting part is the body of the request. Here I combined Eloquent with Twig. The choice of
Twig as opposed to Blade is because [Twig has an Array Environment](https://twig.symfony.com/doc/3.x/intro.html#basic-api-usage)
and a [secure Sandbox](https://twig.symfony.com/doc/3.x/api.html#sandbox-extension).
Eloquent is great for it because we can write Accessor methods that will act as the data source and
transform the data into an array so that Twig can parse it. Here is a sample of a Request Body:

```json
{
  "email": "{{ event.email }}",
  "country": "{{ event.country }}",
  "login_at": "{{ event.happened_at }}",
  "organization": "{{ event.team_name }}"
}
```

This is what we store as the body of the webhook request. A frontend application can offer some
dropdown options and build this JSON automatically for the users. In the backend we will parse it
with the following script:

```php
    private function body(string $template, EloquentModel $event): string
    {
        $loader = new \Twig\Loader\ArrayLoader(['template' => $template]);

        $twig = new \Twig\Environment($loader);

        $policy = $this->twigSecurityPolicy();

        $twig->addExtension(new \Twig\Extension\SandboxExtension($policy));

        return $twig->render('template', ['event' => $event]);
    }

    private function twigSecurityPolicy(): TwigSecurityPolicyForWebhooks
    {
        $tags = [];

        $filters = [];

        $methods = [];

        $properties = []; // We allow Eloquent Properties on TwigSecurityPolicyForWebhooks

        $functions = [];

        $policy = new \Twig\Sandbox\SecurityPolicy($tags, $filters, $methods, $properties, $functions);

        return new TwigSecurityPolicyForWebhooks($policy);
    }
```

Twig's ArrayLoader is perfect for inline/user-provided template and unfortunately Laravel Blade
is very tied to the file system so this is what led me to choose Twig over Blade. The Security Policies
will disallow any attempt at remote code execution or scripting attack from users. In order for this out
to work we only need the Eloquent model to have attributes called `email` or accessors such as the following:

```php
public function getTeamNameAttribute() {
    return $this->team->name;
}
```

The last important bit is the `TwigSecurityPolicyForWebhooks`. I wrote it because Twig doesn't have any
sort of wildcard `*` character to allow property access.

```php
final class TwigSecurityPolicyForWebhooks implements SecurityPolicyInterface
{
    public function __construct(private SecurityPolicy $policy) {}

    public function checkSecurity($tags, $filters, $functions): void
    {
        $this->policy->checkSecurity($this, $filters, $functions);
    }

    public function checkMethodAllowed($obj, $method): void
    {
        $this->policy->checkMethodAllowed($obj, $method);
    }

    public function checkPropertyAllowed($obj, $method): void
    {
        // Let's allow any property on any Eloquent Model
        if ($obj instanceof \Illuminate\Database\Eloquent\Model) {
            return;
        }

        $this->policy->checkPropertyAllowed($obj, $method);
    }
}
```

With all of this setup, all we have left to do is to actually send out the webhook call and record
any result from it. The next snippet represents that portion:

```php
try {
    $response = $client->withHeaders($headers)->send($webhook->method, $webhook->url, ['body' => $body]);
} catch (HttpClientException|GuzzleException $t) {
    return WebhookExecutionBag::exception($body, $t);
}

return WebhookExecutionBag::processed($body, $response);
```

#### Conclusion

I enjoyed working on this project A LOT. It combines a lot of simple and straight-forward tech to give
a huge business benefit with easy drag-and-drop system integration. Users are able to pick virtually any
API out there and call them from our system in real-time as things happens. The webhook configuration
consist of allowing users to define the Body of the Request, custom headers, an endpoint and merge tags
for body transformation. Our users can call an API that expects a URL token, Basic Auth, OAuth2 or custom
headers. We are also able to sign the body of the request in case the receiving end wants/is able to
validate it. Twig security policies protect us against code injection and our users are able to build
their request body as the target system expects it. And finally the Webhook Delivery list will
always be up-to-date with every API call we made and their response status, timestamp and any relevant
diagnostic information.

This is an extremely simple and powerful webhook provider implementation that empower businesses to
seamlessly integrate data flows without having to write code. 

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.