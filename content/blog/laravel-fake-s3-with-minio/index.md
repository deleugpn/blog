---
title: Laravel Fake S3 with Minio
date: "2020-10-16T23:26:30.284Z"
description: How I write test code against an S3 compatible storage
---

Most of the time when I'm writing test code in Laravel I take
advantage of the great `Storage::fake()` provided by Laravel
Test Suite. However, I usually like to have at least 1 test
that _feels_ a bit less "mocky". That's where Minio comes in.
Minio is a storage service fully compatible with S3. Here's
how simple it is for me to write test code with it.

#### Docker

I use Minio's docker image to run the service in a container
linked with my source code through docker-compose. The service
declaration looks like this:

```yaml
  minio:
    image: minio/minio
    environment:
      - MINIO_ACCESS_KEY=customergauge
      - MINIO_SECRET_KEY=phpunit123
    command: server /data
```

As long as I have a php container with my Laravel code in the
same docker-compose file, I'm able to communicate with minio
using `http://minio:9000`.

#### Test Code

My Test Code will look like this:

```php
$minio = new Minio();

$minio->disk('my-bucket', function (S3Client $client, string $bucket) {
    $this->post('/my/endpoint/that/interacts/with/s3', [])
        ->assertSuccessful();

    $object = $client->getObject([
        'Bucket' => $bucket,
        'Key' => "/my/expected/s3/key"
    ]);

    $content = $object['Body']->getContents();

    $this->assertStringContainsString('partial-file-content', $content);
});
```

When instantiating the Minio class, I have the opportunity
to overwrite some defaults. For instance, if for some reason
the container name is not `minio`, I can specify a different
host:

```php
$minio = new Minio('my-host', 9000, 'access-key', 'secret');
```

Minio's `disk` method will take a bucket name and a callback.
In the callback we're free to do whatever we need to 
test our code against S3. It usually involves calling
a Laravel endpoint and making assertions against the file
that was uploaded to S3. The S3Client provided to the callback
will already be configured to communicate with Minio.

#### The Minio Class

For convenience, I opened source the minio class at 
https://github.com/cgauge/laravel-s3-minio and you call grab
it by running `composer require customergauge/minio`, but if
you don't want to rely on a package, the class is pretty simple
and straightforward:

```php
<?php declare(strict_types=1);

namespace CustomerGauge\Minio\Testing;

use Aws\S3\S3Client;
use Illuminate\Contracts\Config\Repository;
use Throwable;

class Minio
{
    private $host;

    private $port;

    private $key;

    private $secret;

    private $config;

    public function __construct(
        string $host = 'minio',
        int $port = 9000,
        string $key = 'customergauge',
        string $secret = 'phpunit123',
        ?Repository $config = null
    ) {
        $this->host = $host;
        $this->port = $port;
        $this->key = $key;
        $this->secret = $secret;
        $this->config = $config ?? config();
    }

    public function disk(string $disk, callable $callback)
    {
        $endpoint = 'http://' . $this->host . ':' . $this->port;

        $client = new S3Client([
            'region' => 'local',
            'version' => '2006-03-01',
            'endpoint' => $endpoint,
            'use_path_style_endpoint' => true,
            'credentials' => [
                'key' => $this->key,
                'secret' => $this->secret,
            ],
        ]);

        $bucket = "$disk-bucket";

        // Let's go ahead and configure Laravel filesystem with the provided disk
        // so that the code being tested can properly interact with minio.
        $this->config->set("filesystems.disks.$disk", [
            'driver' => 's3',
            'region' => 'local',
            'bucket' => $bucket,
            'endpoint' => $endpoint,
            'use_path_style_endpoint' => true,
            'key' => $this->key,
            'secret' => $this->secret,
        ]);

        try {
            // If the bucket already exists, it will throw an exception which we can ignore.
            // If the bucket doesn't exist yet, let's create it so that the code will be
            // able to properly interact with it.
            $client->createBucket(['Bucket' => $bucket]);
        } catch (Throwable $e) {

        }

        try {
            $callback($client, $bucket);
        } finally {
            // Whether the developer's code fails or not, we can make a best effort
            // into cleaning up the bucket and deleting it so thatnext time the
            // test runs, we can successfully create an empty bucket.
            $iterator = $client->getIterator('ListObjects', ['Bucket' => $bucket]);

            foreach ($iterator as $object) {
                $client->deleteObject([
                    'Bucket' => $bucket,
                    'Key' => $object['Key']
                ]);
            }

            $client->deleteBucket(['Bucket' => $bucket]);
        }
    }
}
```

#### Conclusion

Most of the time, `Storage::fake()` will do just fine, but
sometimes we want to make sure our code is compatible with
S3 or even just get an extra guarantee that our code's
interaction with AWS SDK is property tested. Minio
is a great way of testing the code's compatibility
with S3 without actually having to use an actual S3 bucket. 

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.