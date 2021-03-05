---
title: "Laravel Report: Date Range"
date: "2021-03-08T21:48:30.024Z"
description: Applying Date Range on Reporting queries
---

> This is the third post on a Laravel Report series
> 
> Part 1: [Dynamic Page Size](https://blog.deleu.dev/laravel-report-dynamic-page-size/).
> 
> Part 2: [User Defined Sorting](https://blog.deleu.dev/laravel-report-dynamic-page-size/).

Nowadays it's very common to have more data than you can
actually act on. Filtering is a natural part of reporting.
The most common type of reporting filter is around date.
It allows users to disregard data that became stale or 
irrelevant and focus on /recent/fresh/actionable data.

For a ticketing system, that may be the number of open
tickets in the last quarter. For an accounting system maybe
that could be the total revenue on the previous calendar-year.
These type of filters usually carry a start and an end date
provided by the user.

#### The Period Interface

On a single system, there's likely multiple date fields on
multiple objects that could be important to provide filtering
capabilities. For that reason, I decided to start with a
`Period` interface that will dictate how I want to work
with these input.

```php
<?php declare(strict_types=1);

namespace App\Components\Report\Input\Period;

interface Period
{
    public function valid(): bool;

    public function field(): string;

    public function start(): string;
    
    public function end(): string;
}
```

The good thing about defining an interface is that we're
then able to write a Builder manipulator class that will
be able to apply a date range clause to any query as long
as the interface is implemented by the Data Transfer Object.

```php
<?php declare(strict_types=1);

namespace App\Components\Report\Input\Eloquent;

use App\Components\Report\Input\Period;
use Illuminate\Database\Eloquent\Builder as Eloquent;
use Illuminate\Database\Query\Builder as Builder;

final class EloquentPeriodScope
{
    public function __construct(private Period $period) {}

    public function __invoke(Eloquent|Builder $builder): void
    {
        $builder->when($this->period->valid(), function (Eloquent|Builder $builder) {
            $builder->whereBetween(
                $this->period->field(), 
                [$this->period->start(), $this->period->end(),
            );
        });
    }
}
```

> Important: Watch out for SQL Injection on the dynamic
> field definition. $this->period->field() should ONLY
> return strings that have been explicitly allowed.

#### Data Transfer Object

Following the same pattern of the Dynamic Page and the
Dynamic Sort, we can now start defining objects that will
hold date range for objects that make sense to the data
we're trying to report. For the Ticket System example
we could define a TicketPeriod object.

```php
<?php declare(strict_types=1);

namespace App\Components\Report\Input\Period;
use Carbon\CarbonImmutable;

final class TicketPeriod implements Period
{
    public function __construct(
        private ?string $field,
        private ?CarbonImmutable $start,
        private ?CarbonImmutable $end,
    ) {}
    
    public static function null(): self
    {
        return new self(null, null, null);
    }
    
    public function valid(): bool
    {
        // By explicitly specifying which fields can be used we're protecting ourselves
        // against SQL injection when the field is used by Eloquent. Always remember
        // that Prepared Statement only work for values and never for field names.
        $fields = ['created_at', 'closed_at', 'first_interaction_at'];
        
        if (! in_array($this->field, $fields)) {
            // use abort(422, '') if applying a valid date filter is mandatory.
            // Return false if applying a filter is not mandatory.            
            return false;
        }
        
        if (! $this->start || $this->end) {
            return false;
        }
        
        return true;        
    }

    public function field(): string
    {
        return $this->field;
    }    

    public function start(): string
    {
        return $this->start->startOfDay()->format('Y-m-d H:i:s');
    }
    
    public function end(): string
    {
        return $this->end->endOfDay()->format('Y-m-d H:i:s');
    }
}
```

Now all we need is to bind this class into the Container
for usage:

```php
    public function register()
    {
        $this->app->bind(TicketPeriod::class, function () {
            /** @var Request $request */
            $request = $this->app->make(Request::class);
    
            $range = $start = $end = null;
    
            $start = $request->input('period.ticket.start');
            
            $end = $request->input('period.ticket.end');
            
            $field = $request->input('period.ticket.field');
    
            if ($start && $end && $field) {
                $start = CarbonImmutable::parse($start);
                
                $end = CarbonImmutable::parse($end);
                
                return new TicketPeriod($field, $start, $end);
            }
    
            return TicketPeriod::null();
        });
    }
```

#### Usage

Now let's build on top of the previous parts and see how
usage would look like:

```php
    public function __construct(
        private Ticket $ticket,
        private TicketPeriod $period, 
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
            ->tap(new EloquentPeriodScope($this->period))
            ->paginate($this->page->size());
    }
```

This report will now protect the database against heavy
load by paginating the result set in a dynamic page size
(user-provided), sort by user-provided field and also
filter on a date range that the user get to choose
start date, end date and which field they can use
for date range.

#### Conclusion

This post closes a simplified version of reporting components
that can make the reporting capabilities on your Laravel
application extremely powerful. Pagination allows the
system to load efficiently, Sorting allows users to focus
first on high-priority records and Date Range allows
users to filter out stale data that may no longer be
relevant.

I might follow up this series with a few more example
of interesting components for reporting, but I believe
these 3 together already give a basic insight on how
Data Transfer Objects allows for an application-wide
standard of communication between frontend and backend
and a consistent user experience when navigating
the reports on the application.

Follow me on [Twitter](https://twitter.com/deleugyn) to
stay tuned with this series.

Cheers.