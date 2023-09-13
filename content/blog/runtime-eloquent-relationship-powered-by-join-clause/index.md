---
title: Runtime Eloquent Relationship powered by Join clause
date: "2020-06-29T20:45:14.284Z"
description: How to make a Runtime relationship to support a specific report 
---

A big part of my work involves writing effective report. The special
thing about the reports I write is that they're awfully dynamic.
I attribute a large portion of it to something I've been calling
"analytics as a service". There's tons of analytical tools out
there to help you run analytics on your business. Most of them
are either over-the-moon expensive for my use case or they just
don't fit. That's because after customers upload their data, I build
reports for them based on data that I have no idea what it is or
what will be used for. One customer's data is completely different
than another customer's data. Different mappings, different dimensions,
different focus point. Users use an intuitive and overly simplified
UI to build complex database queries. Eloquent has been incredibly
helpful in that regard. It empowers automation tests that actually
hit a database, a practice viewed as bad by some developers, and
it's Query Builder provides OOP syntax for building complex
database queries.

#### Scenario

Today I had to change an existing report to add a new field. For the
sake of oversimplifying it I came up with a similar use-case that
could illustrate the scenario: Imagine a `payments` table full of
entries related to payment records. A `disputes` table refers to
disputes opened relating `payments`. A payment can only have one
dispute and a dispute only relates to one payment. The feature 
request was to add the name of the support team responsible for
that dispute, relating it to a `teams` table. 

#### Desired behavior

The desired behavior was to add something along the following
lines to the report: 

```php
$payments->newQuery()
    ->join(...) // Joining to `disputes`
    ->... // Complex operations
    ->with('teams:id,name')
    ->paginate();
```

The obvious problem for me was the fact that the report starts
on the `Payment` model and not on the `Dispute` model. Changing
it to start from the `Dispute` was not an option as it would
break other parts of the report.

Another possibility would be to use `HasOneThrough`. That would
mean changing the snippet to use `with('disputes.teams')`, however
that would completely ignore the `JOIN` that has been used for
performance reasons. The performance degradation of such `with()`
clause would be a relevant negative impact on the report.

Knowing all of this, I knew that I could, in theory, add a new
relationship to the `Payment` model that would link it to the
`teams` table directly. But that would be a completely fake
and broken relationship for any case where the join is not
established.

#### Runtime Relationship

[iamgergo's pull request to Laravel 7](https://github.com/laravel/framework/pull/33025)
makes a great addition to Query Builder's abilities. It allows
to define a relationship during runtime without having to
actually change the Model class. Here's a snippet:

```php
$payments->newQuery()
    ->join(...) // Joining to `disputes`
    ->... // Complex operations
    ->tap(new TeamRelationship)
    ->paginate();

class TeamRelationship
{
    public function __invoke(Builder $builder)
    {
        $builder->addSelect('disputes.team_id')->with('teams:id,name');

        $builder->getModel()->resolveRelationUsing(
            'teams', 
            function (Model $model) {
                return $model->belongsTo(Team::class, 'team_id');        
            }
        );  
    }
}
```

Now when the execution of the Builder takes place, Eloquent will
try to resolve the `teams` eager load relationship in the 
`Payment` model and will not find it. It will then give it
another chance by looking at the relations added via runtime
through the `resolveRelationUsing` method and will run the 
`teams` callable. That relationship will work because the Join
clause brings the `team_id` field into the resultset, making
Laravel interpret it as if there was a relationship between
the resultset and the `teams` table.

#### Conclusion

A few nice things came into play while I was working on this
earlier today. The pull request that added this functionality
had so little code change into Laravel's core which made it
easier for Taylor to take on the maintenance burden of such
feature. Credit goes to [iamgergo](https://twitter.com/_iamgergo)
for it. Secondly, adding a relationship to the `Payment` class
itself would be just flat out wrong even though it would work.
The problem with that approach would be that the `Payment`
class would have wrong instruction that does not work except
when joining with the `Dispute` model under a specific condition.
Thirdly, it's extremely easy to specify how the relationship
should be handled right before the query gets resolved.
This took me about 2 hours from reading the pull request until
full implementation covered by tests, including the extraction
to a invocable class via `tap`.

Hope you enjoyed the reading. If you have any questions,
send them my way on [Twitter](https://twitter.com/deleugyn).

Cheers.