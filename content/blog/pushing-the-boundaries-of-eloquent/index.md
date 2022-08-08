---
title: Pushing the boundaries of Eloquent
date: "2021-06-04T21:58:14.737Z"
description: My experience building advanced reporting with Eloquent
---

> Check out my short series on Laravel Reports:
>
> Part 1: [Dynamic Page Size](https://blog.deleu.dev/laravel-report-dynamic-page-size/).
>
> Part 2: [User Defined Sorting](https://blog.deleu.dev/laravel-report-user-defined-sorting/).
https://blog.deleu.dev/laravel-report-date-range/
>
> Part 3: [Laravel Report: Date Range](https://blog.deleu.dev/laravel-report-date-range/)

A few months back I sent a slack message to my team celebrating
3 years since I started a brand new Laravel project dedicated
for our reporting service. Over the course of 3 long years
this project has accumulated approximately 175 APIs and
a rich set of non-functional requirements that includes date
filtering, property filter, user-bound data restriction,
dynamic pagination, CSV export, Trend (group by time), Stack
(group by properties), Year to Year comparison and maybe
a few more that I may be forgetting about. Nowadays I'm able
to design a fully-functioning set of reporting functionalities
that includes several of these features in a matter of hours
thanks to the set of reporting components that I have designed
over the years and today I would like to share a bit of my
experience when doing these things.

> This post will detail my personal view of many things in
> the development of a reporting service while leveraging
> Laravel and Eloquent. I'm not trying to convince anyone
> of anything. This is simply how I've been working for the 
> past 2 years. Feel free to agree, disagree, learn or abhor
> anything and everything in chunks or bulks.

#### What Eloquent has to do with it?

SQL is hard for me. But I don't mean writing, reading and
understanding one query. I mean managing dynamic queries,
those that have tons of variable in the middle of them.
In the past I have tried to diagnose and fix queries
that had anywhere between 5 to 50 variables in the middle of them
and it was unimaginably hard to read, understand and reproduce.
Here's an attempt to illustrate what I wanted to avoid:

```sql
SELECT 
    SUM(IF(type = 1, revenue, 0)) as monthly_recurring_revenue,
    SUM(if(type = 2, revenue, 0)) as one_time_revenue,
    $columns 
FROM 
    $revenue_table 
    $join_accounts 
    $maybe_join_tickets
WHERE
      1=1
      $date_clause
      $access_restriction_clause
      $filter_clause
$maybe_group_by_clause
LIMIT $page_size OFFSET ($page * $page_size)
```

This is one of the easiest to understand because it's kind of
well structured, but even then you still have lots of code
branches deciding whether to join a table or not, a bunch
of loop accumulating columns to select from the database
or operations to aggregate and sometimes we even had queries
that surpassed 65k in characters, which could not be stored
within the database for background exporting due to the
limit of the `TEXT` column type.

I set myself on a journey to starting a much more manageable
reporting service that would allow for fast, powerful and
highly dynamic reporting capabilities and Eloquent was
the single most powerful tool that allowed me to write code
that was incredibly easy to reason about and manage.

#### A Trend Report

This is a type of reporting that reaches beyond just the 
limited context in which I work on. There are tons of services
out there that offer some sort of trending reporting in their
own way, so it seems a good place to start. Let's look at
the requirements.

- Given an interval, each data point represents a different time.
- Data MUST be continuous (no lack of data points in a given interval).
- Users may choose any of the datetime fields of a specific object.
- Trends may be daily, weekly, monthly, quarterly or yearly.
- Sometimes data may cross more than just one table (correlation).

That's a reasonable list of business requirements. The hardest
one was "Data MUST BE continous", so let's dig deeper on that one,
consider the following query:

```sql
SELECT COUNT(*), DATE_FORMAT(`opened_at`,'%Y-%m') as `period`
FROM `tickets` 
GROUP BY `period`
```

This essentially represents a monthly trend. However, MySQL will
not fill in the gaps. Let's say we're looking into a 3 year
period, which represents 36 points in a line chart. If there
are any months which no tickets were opened, MySQL will not
provide those rows, so the dataset will not be continuous.
In this particular case, we actually want to plot that 0
tickets were opened if there were any months with no new opened
ticket.

Before we dig into an implementation, let's look at another
somewhat hard requirement: multiple data sources. In this
Trend Report, let's plot the correlation between number of
open tickets and number of finalized sales.

```sql
SELECT COUNT(*), DATE_FORMAT(`purchase_at`, 'Y%-%m') as `period`
FROM `purchases`
GROUP BY `period`
```

Now let's look into how Laravel can tackle this.

```php
final class PurchaseTicketCorrelationTrendRepository
{
    public function __construct(
        private PurchaseTicketCorrelationTrendInput $input,
        private Purchase $purchase,
        private Ticket $ticket,
    ) {}

    public function trend(): Table
    {
        $purchaseTrend = $this->purchase->newQuery()
            ->selectRaw('COUNT(*) as `aggregate`')
            ->tap($this->input->trend)
            ->get();

        $ticketTrend = $this->ticket->newQuery()
            ->selectRaw('COUNT(*) as `aggregate`')
            ->tap($this->input->trend)
            ->get();

        return $this->input->trend->datapoints(function (string $period) use ($purchaseTrend, $ticketTrend) {
            $ticket = $ticketTrend->firstWhere('period', $period);
            $purchase = $purchaseTrend->firstWhere('period', $period);

            return [
                'period' => $period,
                'tickets_count' => (int) $ticket?->aggregate,
                'purchases_count' => (int) $purchase?->aggregate,
            ];
        });
    }
}
```

This is not a lot of code, but there is quite a lot to unpack here.
Let's start with the easy ones. `Purchase` and `Ticket` are our
Eloquent Model and we're aggregating their count and executing
a `get()`. We don't need to `paginate()` this result because
we are doing a Trend. Remember from the requirements that the
trend will be either daily, weekly, monthly, quarterly or yearly.
Even if we have an interval of 10 years with a `GROUP BY` day,
that's `365 * 10 = 3650` rows. Regardless of how many records the
table have, we won't run out of memory if we need to hydrate
3650 tickets model and 3650 purchases model.

The `PurchaseTicketCorrelationTrendInput` class is a 
Data Transfer Object that represents user input and a helper
to get the right set of parameters for the reporting.

```php
final class PurchaseTicketCorrelationTrendInput
{
    public EloquentTrend $trend;

    public function __construct(TicketPeriod $period, Interval $interval)
    {
        $this->trend = new EloquentTrend($interval, $period);
    }
}
```

> The `TicketPeriod` object here is a Value Object that is automatically
> populated by `request('period.ticket.start')` and `request('period.ticket.end')`.
> Read more about it at [Laravel Report: Date Range](https://blog.deleu.dev/laravel-report-date-range/).

The interest bits are located in the `EloquentTrend` class.
From the Repository class we have`->tap($this->input->trend)`
which is the trigger for the `GROUP BY` and later we have
`$this->input->trend->each()` which will help us build an
array with every single data point in the date interval
provided by the user.

```php
final class EloquentTrend
{
    public function __construct(
        private Interval $interval,
        private Period $period,
    ) {}

    public function __invoke(Eloquent|Builder $builder)
    {
        $builder->tap($this->trendInterval())->tap(new EloquentPeriodScope($this->period));
    }
    
    public function datapoints(Closure $callback): array
    {
        $rows = [];
        
        $carbonPeriod = CarbonPeriod::create($this->period->start(), $this->increment(), $this->period->end(), CarbonPeriod::IMMUTABLE);

        /** @var CarbonImmutable[] $datapoints */
        $datapoints = $carbonPeriod->toArray();

        foreach ($datapoints as $datapoint) {
            $rows[] = $callback($datapoint);
        }
        
        return $rows;
    }
    
    private function trendInterval(): TimeSeries
    {
        return match ($this->interval->toString()) {
            'day' => new GroupByDay($this->period->field()),
            'week' => new GroupByWeek($this->period->field()),
            'month' => new GroupByMonth($this->period->field()),
            'quarter' => new GroupByQuater($this->period->field()),
            'year' => new GroupByYear($this->period->field()),
            default => throw new InvalidArgumentException("Invalid parameter interval with value [$interval]"),
        };
    }
    
    private function increment(): string
    {
        return match ($this->interval->toString()) {
            'day' => '1 day',
            'week' => '1 week',
            'month' => '1 month',
            'quarter' => '3 months',
            'year' => '1 year',
            default => throw new InvalidArgumentException("Invalid parameter interval with value [$interval]"),
        };
    }
}

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

final class GroupByDay implements TimeSeries
{
    public function __construct(private string $field) {}

    public function __invoke(Builder $builder): void
    {
        $builder->selectRaw("DATE_FORMAT($this->field,'%Y-%m-%d') as `period`")
            ->groupBy('period');
    }
}

final class GroupByWeek implements TimeSeries
{
    public function __construct(private string $field) {}

    public function __invoke(Builder $builder): void
    {
        $builder->selectRaw("DATE_FORMAT($this->field, '%x w%v') as `period`")
            ->groupBy('period');
    }
}

final class GroupByMonth implements TimeSeries
{
    public function __construct(private string $field) {}

    public function __invoke(Builder $builder): void
    {
        $builder->selectRaw("DATE_FORMAT($this->field, '%Y-%m') as `period`")
            ->groupBy('period');
    }
}

final class GroupByQuarter implements TimeSeries
{
    public function __construct(private string $field) {}

    public function __invoke(Builder $builder): void
    {
        $builder->selectRaw("CONCAT(YEAR($this->field), ' Q', QUARTER($this->field)) as `period`")
            ->groupBy('period');
    }
}

final class GroupByYear implements TimeSeries
{
    public function __construct(private string $field) {}

    public function __invoke(Builder $builder): void
    {
        $builder->selectRaw("DATE_FORMAT($this->field, '%Y') as `period`")
            ->groupBy('period');
    }
}
```

The EloquentTrend is not an extensive class. It's essentially
a helper class that will bind the EloquentPeriodScope and
a TimeSeries in a single operation. The EloquentPeriodScope will
bind the user-provided `start` and `end` date in a `WHERE` clause
with `BETWEEN`. The GroupBy clauses will add a new column to
be brought from the database that represents the exact period
which the data point falls into and also applies the `GROUP BY`
clause into it.

This concludes how the query is written and executed. With the
help of the extremely powerful `tap()` method we can delegate
the responsibility of adding purpose-built modifications to
the query for specific classes. Let's circle back to the `datapoints`
method now. With the help of `CarbonPeriod`, we can create an
array that has every single data point necessary for the trend
respecting the continuity requirement. We then run a callback
for each datapoint so that the Repository class can format
the array that will represent the calculated response of
the trend. That's it! With this set of components, building
a Trend or Correlation Report is a matter of writing a few
lines of Repository class that will tie together the Table
and the user parameters with the `EloquentTrend` class.
Returning this to a frontend application allows for some
very nice trend charts.

#### User-bound Data Access Restriction

In the corporate world, and specially with GDPR and data privacy,
the ability to limit the user to a subset of data is not just a
nice-to-have functionality, but rather a must-have. Fortune 500
enterprises often has a journey through data security and restrictive
access throughout a large user base. As I showed on the previous
topic, Eloquent's `tap()` is a wonderful and powerful functionality
that allows for purpose-built query modification that are explicit.

Data restriction is a subject that was part of the very early design
of the application I've been working for more than 3 years. In the
early days I tried Global Scopes, Local Scopes and Macros. They all
have somewhat a similar drawback: 

- Complex static analyses which reduces IDE code completion
- Implicit behavior that is hard to debug
- Lack of flexibility / reusability.

I could never get used to debugging Global scopes. I would look at
the database and see the data, look at the code and see no reason
for why a specific record would simply not come up. Sometimes in
a large query it's hard to notice one WHERE clause added by a
Global Scope. But they are highly reusable. Local Scopes are the
opposite. Putting Local Scopes on traits just doesn't feel like a
nice development style for what I was doing. At some point I decided
to give macros a chance, although I also remember having a conversation
with my team that "they felt like runtime inheritance with direct access 
to global scope". And it's the option that least had decent IDE support.
I spent weeks dreaming of having the ability to implement Laravel's `Scope`
interface but declare it's usage like a local scope. At some point [I even
sent a pull request to Laravel](https://github.com/laravel/framework/pull/30299)
attempting to allow invocable classes to modify the Query Builder, but
that's when I learned about `tap`. With `tap()` it's possible to do
the things I'm trying to accomplish without any of the drawbacks brought
by the other options. It's explicit and right in the developer's face
which invocable class is being used (like local scopes). It's highly
reusable for multiple reports (like global scopes) as seen by the Trend
implementation on the previous topic. And it has no impact on static analyses
/ code autocompletion like macros. Here's the implementation that I settled
with nearly 2 years ago:

```php
final class EloquentRestrictionFilter
{
    private ?string $table;

    public function __construct(private User $user) {}

    public function qualify(string $table): self
    {
        $this->table = $table;

        return $this;
    }

    public function __invoke(Builder $builder)
    {
        $filters = $this->user->enforcedFilters();

        foreach ($filters as $filter) {
            $column = $this->qualify($filter->foreignKey());
            
            $builder->whereIn($column, $filter->values());
        }
    }

    private function qualify(string $field): string
    {
        if ($this->table) {
            return $this->table . '.' . $field;
        }

        return $field;
    }
}
```

This class will be used on any reporting that must 
enforce an automatic filter criteria that is assigned
to the authenticated user. If we go back to the Repository
from the Trend report, we can modify it to accommodate this
restriction like this:

```php
        $purchaseTrend = $this->purchase->newQuery()
            ->selectRaw('COUNT(*) as `aggregate`')
            ->tap($this->input->trend)
            ->tap($this->input->restriction)
            ->get();

        $ticketTrend = $this->ticket->newQuery()
            ->selectRaw('COUNT(*) as `aggregate`')
            ->tap($this->input->trend)
            ->tap($this->input->restriction)
            ->get();
```

In some cases, I have queries that join multiple tables 
and the user restriction may conflict between columns
from multiple tables. If you've worked with a few
join queries in MySQL you might know the 'Column x is ambiguous'
error. There's essentially two ways to attempt to handle that.
One of them involves doing `$builder->qualify()` within the
`EloquentRestrictionFilter` class. Laravel's Builder will
automatically qualify the column with it's own table name.
However, sometimes I've had issues with that format because
of `UNION ALL` clauses where the primary Eloquent Model
differs from the secondary table being unionized. For
that reason I tend to make it explicit within the
invocable class itself:

```php
$builder->tap($this->input->restriction->qualfiy('tickets'));
```

I understand this model is far from sophisticated. 
The reason it actually works for me is because
for the past 15 years everyone that has changed
the database which I work on has consistently
made sure that a few key attributes are present
on all relevant tables. Hence user access to
parts of the data model are "simplified" in a way
that users can be configured to have limited access
and the limiting attribute will be present throughout
the system where it's relevant. The beauty of
`tap()` is that when I'm working on reports that
don't handle sensitive data, I can simply not apply
the restriction scope at all and it will be clear
and explicit.

#### Lazy Union

Perhaps the example that I'll use for this topic
might be questionable at best, but I kindly ask for
your patience and bear with me. Remember that I'm talking
about a system that started about 15 years ago in an era
that nobody could even dream of `composer`. Parts of the
database design is that much old and has years of code
relying on such design. So even if the example looks a bit
far fetching, it resembles a real scenario that I have
worked on and I can proudly say that I'm satisfied with
the current reporting we have in place.

Picture 4 tables: `purchases`, `stale_purchases`, 
`holiday_purchases` and `invalid_purchases`. They all
have the same set of columns. In fact, they could all
be a single table with a `status` column in them.
Unfortunately they were designed with the mindset that
a single table indefinitely growing over time would
lead to so many records that it would take forever
to report on them. So they were split into smaller
chunks to make it "more manageable". The irony of it
all is that nowadays the biggest performance problem
we have is caused by `UNION ALL` clauses because
MySQL loses the ability to use indexes and require
a full materialization to perform outer modifications.

I learned that if  I can push down clauses to the
individual tables, then I get to better manipulate
how the database will handle these operations by
working with composite indexes.

Before we move on to implementation, there's another
subject I want to talk about: the `OR` devil. When
working with a large dataset and trying to load a high
variety of data, `OR` can be a database killer.
This is because MySQL will lose it's ability to navigate
through the indexes. Picture a MySQL index like a PHP
array. If you know what you're looking for, then the
access to $data['specific_index'] will give you a jump
start on where to locate your data. But what if the
data we're looking for can either be on 
`$data['specific_index'] OR $data['some_other_index'] OR $data['yet_another_index']`.
MySQL makes a best-effort in trying to judge whether to
use an INDEX or not. Imagine if the time it takes to
make a decision between INDEX or FULL SCAN table is
higher than just doing a FULL SCAN. Any time wasted
in trying to decide one or another is time that could
be spent on the worst-case scenario. For complex `OR` clauses,
with high diversity, MySQL will simply make a fast decision
of going with a full table scan. There's a workaround for
that. If my query makes use of several `UNION ALL` instead
of `OR` clauses, then each individual query can target
a specific high-performance index.

Keeping these two use case in mind, here's a nice little trick
we can do with Eloquent.

```php
final class ValidPurchaseUnion
{
    private Builder $purchase;

    private Builder $stale;
    
    private Builder $holiday;

    public function __construct(Builder $purchase, Builder $stale, Builder $holiday)
    {
        $this->purchase = $purchase;
        $this->stale = $stale;
        $this->holiday = $holiday;
    }

    public function purchase(): Builder
    {
        return $this->purchase;
    }

    public function stale(): Builder
    {
        return $this->stale;
    }
    
    public function holiday(): Builder
    {
        return $this->holiday;
    }

    public function unionize(): LazyUnion
    {
        $statusFilter = fn($status) => (clone $this->holiday)->where('holiday_purchases.status', $status);

        $holidayBuilders = collect(HolidayPurchase::VALID_PURCHASES)->map($statusFilter)->toArray();

        return new LazyUnion($this->purchase, $this->stale, ...$holidayBuilders);
    }
}
```

Before we talk about the LazyUnion class, let's unpack the 
ValidPurchaseUnion first. Here I'm designing a report
that will be looking at every valid purchase data.
As a imaginary complexity, I threw in some invalid records
within the `holiday_purchases` table - remember my introduction
in this topic, it's not a complete fake requirement, it's just
adapted. We have tried doing `WHERE status IN(...)` as well
as `WHERE status = ? OR status = ? ...` and it all falls into
the same issue that MySQL loses the ability to run effective
indexes very quickly. Here's how we can make use of this
class:

```php
final class PurchaseRepository
{
    public function __construct(private ValidPurchaseUnion $union, private User $user) {}
    
    public function details()
    {
        $this->union->purchase()->selectRaw('AVG(total_amount) as avg_amount')->where(...);
        
        $this->union->stale()->selectRaw('0 as total_amount`')->where(...);
        
        $this->union->unionize()
            ->tap(new EloquentTrend(...))
            ->tap(new EloquentRestrictionFilter($this->user))
            ->after(fn (Builder $builder) => $builder->orderBy(...))
            ->union()
            ->get();
    }
}
```

At this point we've seen two important factors about 
the `ValidPurchaseUnion` class: access to each builder
individually for targeted query manipulation and
the ability to generate multiple union clauses to
avoid `OR` or `IN()` clauses for performance-focused
indexes. 

At the end of the Repository I'm also doing a `tap()`
to build a trend report with the average purchase cost
over time and adding the user restriction. This essentially
builds on top of the 2 previous topics. Let's look at
the LazyUnion to understand how this will play out.

```php
/**
 * @method self select(string|array $columns)
 * @method self selectRaw(string $expression, array $bindings = [])
 * @method self distinct()
 * @method self join(string $table, string $first, string $operator, string $second)
 * @method self whereColumn(string $first, string $second)
 * @method self groupBy(string $column)
 * @method self tap(callable $callback)
 */
final class LazyUnion
{
    use ForwardsCalls;

    private array $builders;

    private array $after = [];

    public function __construct(Builder ...$builders)
    {
        $this->builders = $builders;
    }

    public function union(): Builder
    {
        $firstBuilder = array_shift($this->builders);

        foreach ($this->builders as $builder) {
            $firstBuilder->unionAll($builder);
        }

        foreach ($this->after as $callable) {
            $firstBuilder->tap($callable);
        }

        return $firstBuilder;
    }

    public function after(callable $callback): self
    {
        $this->after[] = $callback;

        return $this;
    }

    public function __call($name, $arguments)
    {
        foreach ($this->builders as $builder) {
            $this->forwardCallTo($builder, $name, $arguments);
        }

        return $this;
    }
}
```

LazyUnion will hold several Builder instanes and all us to modify
them all at once. Remember when I said that applying query 
modifications to the outer UNION **sometimes** lead to MySQL
having to materialize a much bigger dataset than necessary?
With Lazy Union we get to apply things like `EloquentRestrictionFilter`
on all queries at once, consistently. Differently than the
start of the repository, which applies dedicated modifications
to each builder individually. Once we're done making
modifications to all individual queries we can still
make use of the `after()` to append some final modification
to the outer query as a whole. Granted, this example isn't
fully useful because we're doing `union()` immediately after
the `after()` method. But the goal is actually that we can
make use of `after()` as a way to attach query modifications
to the final query at any given moment when we have a `LazyUnion`
and often it's not the same place that will perform the final
`union()->get()`.

This is an extremely powerful tool that allows for code that
is easy to reason about and navigate without sacrificing
the developer's ability to make decision about how the
final query will play out.

#### Period Comparison

This is one of my latest artwork. After years working with
reports that only work on a single date range, I finally
got time to invest in period comparison. The neat trick
was to sacrifice the `WHERE date BETWEEN` clause and 
move it up to the `SELECT SUM(IF(date BETWEEN, column, 0))`.
The obvious drawback is loss of indexes so we can only
do this when the dataset is small enough or if we can
enforce some filters that will end up making use
of the table indexes. Let's look at a Repository.

> Disclaimer: I have this running in production just
> for a few days and maybe I may come back to this
> later and feel that the syntax isn't friendly/nice.
> What I want to disclaim here is that I have not
> seen how debugging Period Comparison will play out yet.
> Everything else in this post so far have been in
> production for several months or even years.

```php
    private function aggregateTicketsBuilder(): Builder
    {
        $expression = new AggregatePeriodComparisonExpression($this->period);

        return $this->builder->tickets()
            ->tap($expression->count('aggregate'))
            ->withCasts([
                'value' => 'int',
                'current_aggregate' => 'int',
                'previous_aggregate' => 'int',
            ]);
    }

// ---------

final class AggregatePeriodComparisonExpression
{
    protected array $expressions = [];

    protected array $bindings = [];

    public function __construct(private Period $period) {}

    public function count(string $alias): self
    {
        // Read https://blog.deleu.dev/laravel-report-date-range/
        // to know how I avoid SQL Injection here.
        $field = $this->period->field();
        
        $this->expressions[] = "SUM(IF($field BETWEEN ? AND ?, 1, 0)) as `current_$alias`";

        $this->expressions[] = "SUM(IF($field BETWEEN ? AND ?, 1, 0)) as `previous_$alias`";

        $this->bindings = array_merge(
            $this->bindings,
            [$this->period->start(), $this->period->end()],
            [$this->period->start()->subYear(), $this->period->end()->subYear()],
        );

        return $this;
    }

    public function __invoke(Builder $builder)
    {
        $expression = implode(', ', $this->expressions);

        $builder->selectRaw($expression, $this->bindings);
    }
}
```

This is a super neat trick that will allow me to expand 
several reports with the capability of doing year
to year comparison or maybe even user-defined
period comparison and I'm looking forward to what
users will be doing with these capabilities. The end result
essentially allows for frontend to show those green arrows
or red arrows similar to how stock market do it. 
Has this metric grew or shrunk? Easy information and
"simple code" to reason (subjective).

#### What about testing?

I'm a subscriber of the Laravel Feature Testing mentality
where we simply let the application hit the database for
tests. I currently have a test suite with more than 1k
tests that run within 20 seconds. I use the `php artisan test --parallel`
functionality with 4~8 cores and I also use `docker-compose` tmpfs
configuration so that MySQL is running in-memory only.
Tests then consist of arring the database with known data
and asserting that the report will have the correct set
of metrics.

I do have a confession. It gets boring. SUPER BORING.
When doing the 10th report that uses the same `EloquentTrend`
class, I don't really want to test all intervals (day, week, quater, ...)
and I also don't want to test all filters, date ranges, etc.
Ultimately, I have a rich set of tools that allow for a vast
range of reports and whenever I'm designing a new report
I try to test a couple of tools I'm using. For instance,
in one report I won't write a test for GROUP BY at all, but will
focus on PeriodComparison or Date Filter while another
report will use generators to test every single GROUP BY
of a Trend. Every tool is tested more than once by having
more than 1 report testing them. Every report will have
a few tests making use of some tools. In the end
no single report covers every use case, but combined
they kill an incredibly huge amount of mutations
generated by `infection/infection`. Bugs still happen.
Whenever a bug is found, irrespective of where in the
pipeline (QA, Product or Client), a dedicated test will
be written for that report/tool combination so that
it never repeats again and it increases the strength of the
test set. I don't like the idea of testing these
_tappable_ classes individually and not testing
the user's interaction with the report.

#### Conclusion

I have a few more tools that I wanted to talk about, but this post
is already far bigger than I planned. I don't even know if anybody
will read it to it's full, so I'm going to stop here for now.
If you made it this far, I really appreciate your time!
I hope you managed to extract from my experience how
powerful it is Eloquent's `tap()` functionality and how I
leverage it to standarize certain reporting functionalities
(Aggregate, Comparison, Data Restriction, Trend, Group By).
In the end, the reporting is complex and the query will
always be complex, there's no walking around that reality for me.
But that doesn't mean the code that generates the query has
to be an impossible beast.


If you liked anything I said, consider following me on [Twitter](https://twitter.com/deleugyn).
But if you think everything I said is nonsense, then follow me on [Twitter](https://twitter.com/deleugyn).

Cheers.
