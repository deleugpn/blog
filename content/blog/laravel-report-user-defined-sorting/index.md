---
title: "Laravel Report: User defined sorting"
date: "2021-03-07T19:04:28.999Z"
description: Improving reporting capabilities with dynamic sort.
---

> This is the second post on a Laravel Report series that
> I'm starting. For part 1, check out 
> [Dynamic Page Size](https://blog.deleu.dev/laravel-report-dynamic-page-size/).

Sometimes a report is composed of several aggregates at
once. These are usually Summary or Stats reporting.
One example is a Ticket Summary report that can display
the total number of tickets on a given date range, but
also include the number of open tickets, in progress and
closed tickets. As I mentioned on the previous post,
these aggregate are usually accompained by a Detail report
so that users can see a detailed list of records that make
up that aggregate. A user can look into which individual
records make up the number of open tickets.

This type of aggregate report doesn't have pagination or
sorting. But the detailed list of records that make up
the aggregate do. If a customer handles millions of tickets
per month, we don't want to load a high number of tickets
all at once, so we paginate. As soon as we paginate, sorting
might become a natural next step. The user may choose to
sort by Staff Member responsible for tickets or maybe
for creation date (oldest to newest).


#### Dynamic Sorting

Following on the same pattern that I wrote the Dynamic Page
Size, let's modify the same Service Provider that is already
in place and add a Sort object as a Data Transfer Object.

```php
    public function register()
    {
        $this->registerPage();
        
        $this->registerSort();
    }

    public function registerPage(): void
    {
        $this->app->bind(Page::class, function () {
            $request = $this->app->make(Request::class);

            return new Page((int) $request->input('per_page', Page::DEFAULT));
        });
    }

    public function registerSort(): void
    {
        $this->app->bind(Sort::class, function () {
            $request = $this->app->make(Request::class);

            $field = $request->input('sort.field');
            
            $direction = $request->input('sort.direction');
            
            return new Sort($field, $direction);
        });
    }
```

Next, let's define the Sort class

```php
<?php declare(strict_types=1);

namespace App\Components\Report\Input\Page;

final class Sort
{
    private array $available = [];

    public function __construct(private ?string $field, private ?string $direction) {}
    
    public function available(array $fields): self
    {
        $this->available = $fields;

        return $this;
    }
    
    private function valid(): bool
    {
        if (! $this->field) {
            return false;
        }

        if (! $this->direction) {
            return false;
        }

        // MySQL columns have a limited of 64 characters and this application never
        // uses any field bigger than 32 characters. If the field provided during
        // the API call is too big, we can simply bail. 
        if (strlen($this->field) > 32) {
            abort(422, "Sorting on [$this->field] is not possible.");
        }

        if (! in_array($this->field, $this->available)) {
            abort(422, "Sorting on [$this->field] is not possible.");
        }

        return true;    
    }
    
    public function fallback(string $field, string $direction): self
    {
        // Let's check if the user provided with a valid sorting input. If they did,
        // we don't need to fallback, otherwise we'll go ahead and fallback
        // to sort by the field and direction provided by the developer.
        if (! $this->valid()) {
            $this->available[] = $field;
            $this->field = $field;
            $this->direction = $direction;
        }

        return $this;
    }
}
```

Given the binding with the Request object, we're now
able to standarize the `sort[field]` and `sort[direction]`
for any report as the parameters that will control the
sorting strategy.

If both fields are not provided, we may `fallback` to a
developer-defined option. Furthermore, before adding
a `orderBy` clause in our SQL, we **MUST** use the
`available` method to define a allowed-list of fields.
Failing to do so will open up the application to
SQL Injection because there's no way to use prepared
statement for fields, only for values.

Here's an usage example:

```php
    public function __construct(
        private Ticket $ticket, 
        private Page $page,
        private Sort $sort,
    ) {
        $this->sort->available([
            'created_at', 
            'status', 
            'user_email', 
            'staff_email',
        ])->fallback('created_at', 'asc');
    }
    
    public function tickets(): Collection
    {
        return $this->ticket->newQuery()
            ->select('tickets.*')
            ->addSelect('users.email as user_email')
            ->addSelect('staff.email as staff_email')
            ->join('users', 'user.id', '=', 'tickets.user_id')
            ->leftJoin('staff', 'staff.id', '=', 'tickets.staff_id')
            ->tap($this->sort)
            ->paginate($this->page->size());
    }
```

This repository is responsible for providing the detailed
list of tickets and some additional data about these tickets
in a paginated-list. Using the `Sort::available` method, we
can define what are the fields available for sorting.
Also, to avoid extensive use of `if ($this->sort->valid())`
we can instead opt to use `tap()`. Query Builder `tap()`
will allow us to delegate the responsibility of modifying
the Query to the object at hand. For this to work, let's
go back to the `Sort` class and add an `__invoke` method.

```php
    public function __invoke(Eloquent|Builder $builder): void
    {
        $builder->when($this->valid(), function (Eloquent|Builder $builder) {
            $builder->orderBy($this->field, $this->direction);
        });
    }
```

And that's it. Sorting will be applied when the `Sort`
object is carrying valid data or have a fallback defined.
We're protected from SQL Injection by explicitly defining
a list of `available` fields to be used and the frontend
application can provide a sortable table to users navigating
through reports.

#### Conclusion

User-defined sorting is extremely relevant at scale. 
Some users are usually looking through reports that has
hundreds of thousands of records and they can't really
go through them all. Sorting provides the ability to
prioritize based on important information available.

The sorting component presented in this post allows
for reusable code while keeping the developer capable
of customizing some important aspects that are individual
for each report, such as fields available for sorting.
This concludes two reporting features: dynamic page size
and dynamic sorting.

Follow me on [Twitter](https://twitter.com/deleugyn) to
stay tuned with this series.

Cheers.