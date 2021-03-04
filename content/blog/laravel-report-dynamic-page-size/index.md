---
title: "Laravel Report: Dynamic Page Size"
date: "2021-03-06T21:51:49.521Z"
description: Letting API caller define page size
---

I work on the reporting service of a large SaaS application
where we offer analytics-as-a-service. One of the biggest
challenge in this type of application is that reporting
is extremely customizable. With this post I'm interested
in starting a new series of Laravel Report where I will
introduce a few key aspects of how I achieve repetitive
functionality across several reporting APIs by standarizing
important features. The simplest feature I could think
of to kickstart this series is user-controlled page size.

#### Reporting Context

When working with reporting-as-a-service, I noticed that
there are usually 2 types of report and 1 derivative:

- Aggregate
- Stack
- Detail (Derivative)

An aggregate report is one that compute COUNT, SUM, AVG, etc
on a dataset. Examples could be total number of sales in the last
15 days for brand MyBrand. Total revenue in transactions on 
the last 24 hours on a specific country. Average time to close
support tickets for the last quarter for Level 1 Support
employees.

More often than not, when users navegate these aggregate
numbers they end up interested in double-checking the
underlying data that makes up such a number. These are
referenced in this post as a derivative report. The dataset
is exactly the same as the primary report with a slight
difference that instead of doing a COUNT, SUM or AVG, we will
instead do a paginate on the entire dataset and let the user
navigate through it.

#### Dynamic Page Size

After building several dozens of derivative reports that
requires dynamic page size, I came up with a system
that I'm very happy with that allows the API caller
to have some control over the aspect of the reporting
capabilities without losing the ability to prevent
shady users from abusing said control.

Let's start by defining a Service Provider where we'll
standarize how we receive the information about the
page size.

```php
    public function register()
    {
        $this->app->bind(Page::class, function () {
            $request = $this->app->make(Request::class);

            return new Page((int) $request->input('per_page', Page::DEFAULT));
        });
    } 
```

Now let's define the `Page` input.

```php
<?php declare(strict_types=1);

namespace App\Components\Report\Input\Page;

final class Page
{
    public const DEFAULT = 25;

    public const LIMIT = 1000;

    public function __construct(private int $size) {}

    public function size(): int
    {
        return min($this->size, self::LIMIT);
    }
}
```

With just these two classes I'm now able to inject a new
object into any `Repository` class in my application that
will instruct my Query Builder how many records the pagination
should use.

```php
    public function __construct(private Account $account, private Page $page) {}
    
    public function accounts(): Collection
    {
        return $this->account->newQuery()
            ->paginate($this->page->size());
    }
```

#### Conclusion

This process has 2 main advantages for me. The first is
the standard parameter that becomes the project-wide format
to define page size. This will allow several dozen of 
reports to be consistent in a `per_page` input responsible
for defining how many records to retrieve at a time.
The 2nd advantage is that the Repository is still isolated
from the Http context since the `Page` object act as a
Data Transfer Object.
I know this probably looks more complex than it needs
for a small accomplishment, but as I extend on this series
for more reporting functionality it will become clearer
that having this standard can help a vast set of reporting
APIs to share common capabilities.

As always, hit me up on [Twitter](https://twitter.com/deleugyn) with any
questions. 

Cheers.