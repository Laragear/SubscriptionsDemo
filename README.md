# Subscriptions

Manage subscriptions on-premises, without any payment systems!

```php
use Illuminate\Support\Facades\Auth;
use Laragear\Subscriptions\Models\Plan;

$subscription = Auth::user()->subscribeTo(Plan::find(1));

return "You are now subscribed to $subscription->name!";
```

## [Download it](https://github.com/sponsors/DarkGhostHunter/sponsorships?sponsor=DarkGhostHunter&tier_id=195145&preview=false)

[![](sponsors.png)](https://github.com/sponsors/DarkGhostHunter/sponsorships?sponsor=DarkGhostHunter&tier_id=294921)

[Become a Sponsor and get instant access to this package](https://github.com/sponsors/DarkGhostHunter/sponsorships?sponsor=DarkGhostHunter&tier_id=294921).

## Requirements

* Laravel 11 or later.

## Installation

You can install the package via Composer. Open your `composer.json` and point the location of the private repository under the `repositories` key.

```json
{
    // ...
    "repositories": [
        {
            "type": "vcs",
            "name": "laragear/subscriptions",
            "url": "https://github.com/laragear/subscriptions.git"
        }
    ],
}
```

Then call Composer to retrieve the package.

```bash
composer require laragear/subscriptions
```

You will be prompted for a personal access token. If you don't have one, follow the instructions or [create one here](https://github.com/settings/tokens/new?scopes=repo). It takes just seconds.

> [!TIP]
> 
> You can find more information about in [this article](https://darkghosthunter.medium.com/php-use-your-private-repository-in-composer-without-ssh-keys-da9541439f59).

## How this works?

Laragear's Subscriptions package uses the power of Laravel Eloquent Models with some magic to handle a small but flexible Subscription system.

The basic approach is simple: a Plan acts like a blueprint to create Subscriptions, which are marked to renew by certain intervals. A model, like your included User, can have one subscription, or even multiple subscriptions at the same time.

This enables multiple ways to handle subscriptions across your app:

- Upgrade only to a group of Plans.
- Share a subscription between multiple users.
- Lock and unlock Plans...

... and much more.

## Set up

Before creating plans and subscribing users to a Plan, you will need to install the application and setup some  minor models so you're ready to go. Don't worry, it only takes 5 minutes.

### 1. Install the files

Install the migrations tables, the policies, and the plans blueprints file. You can install everything with the convenient `subscriptions::install` Artisan command.

```shell
php artisan subscriptions:install
```

### 2. Set the relationship

> [!TIP]
> 
> If you will be subscribing only your `App\Models\User` model, and access them using `subscribers()`, you can skip this step.

To add your Users as subscribers, and be able to access them easily through the Subscription model, use `Subscription::macroRelation()`. For polymorphic relations, use `Subscription::macroPolymorphicRelation()`. You can do this in your `App\Providers\AppServiceProvider` class, or in your `bootstrap/app.php` file. 

These are methods that use [dynamic relationships](https://laravel.com/docs/11.x/eloquent-relationships#dynamic-relationships) but with a more complete relation callback and `name => class` convenience.

```php
use Laragear\Subscriptions\Models\Subscription;
use Illuminate\Foundation\Application;
use App\Models\Adult;
use App\Models\Kid;

return Application::configure(basePath: dirname(__DIR__))
    ->booted(function () {
        // For a simple monomorphic relationship.
        Subscription::macroRelation('adults', Adult::class);
        
        // For all of your polymorphic relationships.
        Subscription::macroPolymorphicRelations([
            'adults' => Adult::class,
            'kids' => Kids::class,
        ]); 
    })->create();
```

This way you will be able to access subscribers from a Subscription model using the relation name:

```php
use Laragear\Subscriptions\Models\Subscription;

$actors = Subscription::find('bc728326...')->adults;
```

> [!TIP]
> 
> More details in the [polymorphic section](#polymorphic-relations). Don't worry, is just some paragraphs with code.

### 3. Migrate the tables.

If you don't require any further [change to the migrations files](MIGRATIONS.md), you can just migrate your database tables like it was another day in the office:

```shell
php artisan migrate
```

> [!TIP]
> 
> Migrations create 3 tables: `plans`, `subscribable` and `subscriptions`.

### 4. Add the `WithSubscription` trait

Finally, add the `Laragear\Subscriptions\WithSubscriptions` trait to your models, which will enable it to handle subscriptions easily with just a few methods.

```php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laragear\Subscriptions\WithSubscriptions;

class User extends Authenticatable
{
    use WithSubscriptions;

    // ...
}
```

## Plans

First, we need to create the Plans. We can go inside the `plans/plans.php` and create them using the convenient Plan builder. You're free to remove the example Plan.

```php
use Laragear\Subscriptions\Facades\Plan;

// Plan::called('Free')->capability('coolness', 10)->monthly();
```

> [!TIP]
> 
> A Plan works like a blueprint for a Subscription: when a Subscription is created, the Plan data is copied into the Subscription.

For example, you may create two plans for a Delivery app: one for free, and the other paid. We will differentiate them by their names and the number of deliveries per month allowed.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Free')->capability('deliveries', 1)->monthly();

Plan::called('Basic')->capability('deliveries', 8)->monthly();
```

> [!WARNING]
> 
> Plans require an interval. You will get an error if the interval is not set. You can easily make lifetime plans by [renewing them automatically](#renew--extend) in your app.

Once plans are _declared_, we will need to push them into the database. For that, we can use `subscriptions:plans`. The command will read the plans and create them into the database.

```shell
php artisan subscriptions:plans

# Created 2 plans:
# - Free
# - Basic
``` 

That's it, from here you can subscribe entities to it, which can be users, companies, or whatever you want.

Before going into the subscriptions themselves, let's check how we can configure a Plan thoroughly.

### Plan Capabilities

Capabilities are the core of Laragear's Subscriptions. Simple Plans may not need any, but complex ones may need a list of values to allow (or not) a user to do a given action.

The capabilities set in the Plan will get copied over new Subscriptions. This makes the Plan immutable, and make changes only affect new Subscriptions.

Capabilities can be set using `capability()`. For example, we will set the number of deliveries for the Basic Plan:

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Basic')->capability('deliveries', 8)->monthly();
```

You can also set a list of capabilities in one go using an array.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Basic')->capability([
    'deliveries' => 8,
    'delivery' => [
        'priority' => 'normal',
        'groceries' => false,
    ]
])->monthly();
```

Capabilities can be accessed using `dot.notation`. In your Subscription, you can access them later with `capability()`.

```php
$priority = $user->subscription->capability('delivery.priority');
```

### Metadata

Metadata is like a note in the plan itself, which are is visible at serialization by default. This metadata replicated into the subscription itself. To attach metadata to a plan use `withMetadata()`.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Free')->withMetadata([
    'payment' => 'not_allowed',
    'remember' => 'weekly',
])->monthly();
```

Metadata is just a `Collection` instance, so it's easy to retrieve and save values to it later in the Subscription.

```php
echo $subscription->metadata('payment'); // "not_allowed"
```

### Custom columns

Since the migration files are editable with new columns, you can use `withColumns()` to add primitive values to their respective columns.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Basic')->withColumns([
    'price' => 29.90,
])->monthly();
```

Later, these values can be accessed like a normal property from its Subscription, as these values will be copied over them as long the column exists.

```php
echo $subscription->price; // 29.90
```

### Plan groups and levels

Plan groups allows to Subscriptions to be upgraded only to Plans of the same group, and only for "better" plans. This makes a clear path to subscription upgrades, like from "Free" to "Basic" and vice versa.

With `group()` and the name on the group, you can add your Plans from the most basic to the more powerful.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::group('deliveries', [
    0 => Plan::called('Free')->capability('deliveries', 1)->monthly(),
    1 => Plan::called('Basic')->capability('deliveries', 8)->monthly(),
]);
```

With that, users won't be able to downgrade to a lesser plan, or subscribe to two plans of the same group, when checking these actions with the [authorization gates](#authorization).

```php
use Laragear\Subscriptions\Models\Plan;
use Illuminate\Support\Facades\Auth;

$plan = Plan::find('b6954722...');

if (Auth::user()->cannot('upgradeTo', $plan)) {
    return "You cannot upgrade to the $plan->name.";
}

Auth::user()->upgradeTo($plan);
```

Since downgrades are disabled, you can enable them using `downgradeableGroup()` through the `Plan` Facade.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::downgradeableGroup('deliveries', [
    0 => Plan::called('Free')->capability('deliveries', 1)->monthly(),
    1 => Plan::called('Basic')->capability('deliveries', 8)->monthly(),
]);
```

> [!IMPORTANT]
> 
> This doesn't remove the ability for an entity to subscribe to another Plan outside the group.

### Shareable subscriptions

Normally, a subscription would be attached to one entity, like a user or a company, but you can set a subscription to be "shareable" between multiple entities. For example, you can use this to create subscriptions for families or teams.

Use the `sharedUpTo()` to set a number of maximum entities to share the subscription.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Newlyweds special')->monthly()->sharedUpTo(2);
```

An entity will be still able to be attached to another entity subscription, even if it's the same Plan and is marked as _unique_. To disable this, you may want to check if the entity already is attached to Plan before attaching it to another:

```php
use Laragear\Subscriptions\Models\Plan;
use App\Models\User;

if (User::find(1)->hasOneSubscription()) {
    return "The user cannot be attached to two or more active subscriptions.";
}

$plan = Plan::find('b6954722...');

if (User::find(2)->isSubscribedTo($plan)) {
    return "The user is already subscribed to $plan->name.";
}
```

#### Open shared subscriptions

Normally, the shared subscription slots can only be filled by the admin, but you can enable "open" subscriptions where users don't need prior approval to join in. You can easily set this using `openlySharedUpTo()`.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Sunday golf')->monthly()->openlySharedUpTo(4);
```

Later in your app, you can use the [convenient authorization gates](#authorization) to check if a user can be attached to an open or closed subscription:

```php
if ($user->cant('attachTo', [$subscription, $user])) {
    return 'Only the owner of this subscription can invite you.';
}
```

You can programmatically open and close a subscription with `shareOpenly()` and `dontShareOpenly()`, respectively:

```php
$user->subscription->shareOpenly();

$user->subscription->dontShareOpenly();
```

### Renewal cycle

One problem that Laragear Subscriptions fixes is setting by how much a Plan cycle runs before having to be renewed. You have access to a multitude of helpers to set that interval.

| Method            | Description                               |
|-------------------|-------------------------------------------|
| `days($amount)`   | Sets a cycle based on a number of days.   |
| `daily()`         | Sets a daily cycle.                       |
| `weeks($amount)`  | Sets a cycle based on a number of weeks.  |
| `weekly()`        | Sets a weekly cycle.                      |
| `biweekly()`      | Sets a biweekly cycle (two weeks).        |
| `months($amount)` | Sets a cycle based on a number of months. |
| `monthly()`       | Sets a monthly cycle.                     |
| `bimonthly()`     | Sets a bimonthly cycle (two months).      |
| `quarterly()`     | Sets a quarterly cycle (three months).    |
| `biannual()`      | Sets a semester cycle (six months).       |
| `yearly()`        | Sets a yearly cycle.                      |
| `biennial()`      | Sets a two-year cycle.                    |
| `triennial()`     | Sets a three-year cycle.                  |

If none of these cycles adjusts to your liking, you fine tune the interval with `every()`, which accepts an interval.

```php
use Laragear\Subscriptions\Facades\Plan;
use Carbon\CarbonInterval;

Plan::called('Special')->every(CarbonInterval::month()->addDays(5));
```

> [!IMPORTANT]
> 
> Monthly intervals never overflow to the next month. For example, a monthly subscription that starts 31st of January will expire at 28th/29th of February, and continue until the 31st of March.

#### Not renewable plans

Plans are renewable by default, but you may want to create one-time plans with subscriptions that only last one cycle, like a "trial". For that, you can use `notRenewable()`.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Basic')->monthly();

Plan::called('Trial')->weekly()->notRenewable();
```

You can also use the helpers `forOnlyOneDay()`, `forOnlyOneWeek()`, and `forOnlyOneMonth()` to create not-renewable plans for one day, week or month, respectively.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Trial')->forOnlyOneWeek();
```

### Plan subscribers limit

Plans can have a limited number of subscriptions. For example, we may set a "Premium" to not have more than 10 Subscriptions using `limitedTo()`.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Premium')->monthly()->limitedTo(10));
```

By default, a Plan will be always subscribable as long there is slots available. A slot is freed when its Subscription is no longer active. You may disable freeing slots with `permanentlyLimitedTo()`: even if a Subscription is cancelled, nobody will be able to subscribe to it once all slots were taken.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Premium')->monthly()->permanentlyLimitedTo(10);
```

For example, if the 10th subscription is no longer active, no new Subscriptions for the Plan will be able to be created. This makes the `subscribeTo` [authorization gate](#authorization) fail if there are no free slots.

```php
if (Auth::user()->cant('subscribeTo', $plan) && $plan->isFilled()) {
    return 'There is no more available Subscriptions for this Plan.';
}
```

The Plan's Subscription limit can be resized later manually in the Plan Model, from anywhere in your application.

```php
use Laragear\Subscriptions\Models\Plan;

$plan = Plan::find('b6954722...');

// Resize the limit of subscriptions
$plan->resizeLimit(20);

// Or remove the limits altogether.
$plan->removeLimit();
```

Resizing a Plan doesn't unsubscribe users from it, even if the new limit is lower than the active Subscriptions. If you want to forcefully [terminate](#unsubscribe--terminate) subscriptions outside the limit, it's recommended to do it from the oldest.

```php
use Laragear\Subscriptions\Models\Subscription;
use Laragear\Subscriptions\Models\Plan;

$plan = Plan::find('b6954722...');

Subscription::whereBelongsTo($plan)
    ->oldest()
    ->limit(1000)->skip(20) 
    ->each(fn (Subscription $subscription) => $subscription->terminate());
```

### Locking and Unlocking plans

Plans are by default open to all subscribers, but you can manually lock them using the `lock()` method at runtime. This disables new subscriptions when using the [authorization gates](#authorization).

```php
use Laragear\Subscriptions\Models\Plan;
use Illuminate\Support\Facades\Auth;

$plan = Plan::find('b6954722...');

$plan->lock();

if (Auth::user()->cant('subscribeTo', $plan) && $plan->isLocked()) {
    return 'This plan has been locked for new subscribers.';
}
```

> [!IMPORTANT]
> 
> Locking a plan disables new subscriptions, or subscription upgrades to it, but not renewals. You can [disable Plan renewals](#not-renewable-plans).

To revert the operation, use `unlock()`.

```php
use Laragear\Subscriptions\Models\Plan;

Plan::find('b6954722...')->unlock();
```

### Hidden plans

All plans are retrieved and shown publicly when queried. You can hide plans using `hidden()`. This sets a flag on the Plan to not be shown on queries, but still be available to be subscribed normally.

```php
use Laragear\Subscriptions\Facades\Plan;

Plan::called('Unrestricted plan')->monthly()->hidden();
```

To show hidden plans in your query, use the `withHidden()` to include hidden plans.

```php
use Laragear\Subscriptions\Models\Plan;

$plans = Plan::withHidden()->get();
```

> [!TIP]
> 
> It's not necessary to use `withHidden()` when retrieving the Plan from the `Subscription` model instance, the underlying scope is automatically removed so the Plan is always shown.

### Unique plans

By default, an entity can be subscribed to multiple Plans simultaneously. For example, a user can be subscribed to a "Food deliveries" and "Groceries deliveries" at the same time.

Sometimes you will want to not allow a user to subscribe to other Plans. You may set that Plan to be _unique_, meaning, the user won't be able to subscribe to any other plans as long it's subscribed to that unique plan. This can be done with `unique()` on the plan itself.

```php
use Laragear\Subscriptions\Facades\Plan;

$plan = Plan::called('All groceries')->monthly()->unique();
```

When checking for permissions using the [authorization gate](#authorization), the user won't be able to subscribe if the Plan is subscribed to was marked as unique.

```php
use Laragear\Subscriptions\Models\Plan;
use Illuminate\Support\Facades\Auth;

$plan = Plan::find('b6954722...');

if (Auth::user()->cant('subscribeTo', $plan) && Auth::user()->subscription->isUnique()) {
    return 'You cannot be subscribed to any other Plan.';
}
```

Alternatively, you may want to use [groups](#plan-groups-and-levels) to make an entity only be subscribed to one plan of a Plan group.

### Custom plans

While your plans with a set amount of capabilities may work for many subscribers, you may find your app in a position to offer "customizable" plans. You can create these like with any other plans, on demand.

For example, let's say we want to let a company create its own plan through a web interface. It's easy as just building a new plan, and attaching it immediately to the company using `createFor()`.

```php
use App\Models\Company;
use Illuminate\Http\Request;
use Laragear\Subscriptions\Facades\Plan;

public function createPlan(Request $request)
{
    $request->validate([
        // ...
    ]);
    
    $company = Company::find($request->company_id);
    
    $subscription = Plan::called("Custom Plan for $company->name")
        ->capabilities([
            'deliveries' => 50,
            'on_weekdays' => 50,
            'priority' => 1.0,
        ])
        ->monthly()
        ->createFor($company);
    
    return 'Your are now subscribed to a custom plan for you!';
}
```

What `createFor()` does is relatively simple:

- It [hides the plan](#hidden-plans), so it's not shown publicly by default.
- It [locks the plan](#locking-and-unlocking-plans), so nobody else can subscribe to it.

You can also combine the Plan with `unique()` to make the subscriber be only subscribed to that Plan only.

```php
use Laragear\Subscriptions\Facades\Plan;

$subscription = Plan::called('Custom for Company LLC.')
    ->capabilities([
        // ...
    ])
    ->monthly()
    ->unique()
    ->createFor($company);
```

### Custom ID

Plans ID are UUID v4, and are generated automatically when these are saved into the database. This allows to hide the number of existing plans in your application, especially if you create [custom plans](#custom-plans).

You may want to change the UUID generator using the static property `Plans::$useWithUuid`, which accepts a Closure that returns a UUID.

```php
use Laragear\Subscriptions\Models\Plan;
use Ramsey\Uuid\Uuid;

Plan::$useUuid = function (Plan $plan) {
    return Uuid::uuid6();
}
```

If you want to use the [_ordered UUID_](https://laravel.com/docs/11.x/helpers#method-str-ordered-uuid) from Laravel, just call `Plans::useOrderedUuid()`.

```php
use Laragear\Subscriptions\Models\Plan;
use Laragear\Subscriptions\Models\Subscription;

Plan::useOrderedUuid();
Subscription::useOrderedUuid();
```

## Subscriptions

After your plans are set, the next step is to subscribe entities, like a user or a company. It doesn't matter which model you set to be subscribed, as this package handles the subscribers as a polymorphic relationship.

### Understanding Subscriptions

A Subscriptions contains some pieces of additional information to manage temporal calculations, apart from the data copied from the Plan it belongs to:

- `begins_at`: The date from when cycles are calculated from.
- `starts_at`: The current cycle start.
- `ends_at`: The current cycle end.
- `cycles`: The number of cycles since the subscription began. 
- `grace_ends_at`: The additional grace time after the cycle end to consider the Subscription _active_, if any.

You're free to check this data in your Subscription model instance, but don't change it unless you know what you're doing. Instead, use the helpers, which will make modifying a Subscription painless.

### Subscribing to a Plan

The most common task is subscribing to a Plan. You can use `subscribeTo()` with the Plan instance. It returns a persisted `Subscription` model.

```php
use Laragear\Subscriptions\Models\Plan;
use Illuminate\Support\Facades\Auth;

$plan = Plan::find('b6954722...');

if (Auth::user()->cannot('subscribeTo', $plan)) {
    return "You cannot subscribe to the plan $plan->name.";
}

$subscription = Auth::user()->subscribeTo($plan);
```

You can use a second parameter to add or override custom attributes to the subscription. For example, we can add custom metadata for one subscription in particular.

```php
$subscription = $user->subscribeTo($plan, [
    'metadata' => ['is_cool' => 'true']
]);
```

Alternatively, you can use the `Subscribe` facade to attach multiple entities to one subscription, like when doing "Family" or "Team" plans. When doing this, only the first entity will be considered the _admin_.

```php
use App\Models\User;
use Laragear\Subscriptions\Models\Plan;
use Laragear\Subscriptions\Facades\Subscribe;

$plan = Plan::find('b6954722...');

$users = User::where('family', 'Doe Family')->sortBy('age', 'desc')->lazy();

$subscription = Subscribe::to($plan, $users, [
    'metadata' => ['is_cool' => true]
]);
```

#### Attaching data to the pivot

Behind the scenes, Laragear's Subscription package uses the `Laragear\Subscriptions\Models\Subscribable` Pivot Model. This pivot allows to bind one or multiple entities to a Subscription.

When you subscribe a single model to a Plan, change it, or retrieve one from it, you can access the pivot model as `subscribable`.

```php
use Illuminate\Support\Facades\Auth;

Auth::user()->subscription->subscribable->metadata('is_cool'); // "false"
```

You may set any metadata you want for that particular subscriber at subscribing time, or even when changing it. There is no need to add `is_admin` or `metadata`, as the pivot table already comes with these columns, but, for additional columns, you will have to [edit the migration](#3-migrate-the-tables).

```php
use Illuminate\Support\Facades\Auth;
use Laragear\Subscriptions\Models\Plan;

// Subscribe the user to a plan and add metadata for only him.
$subscription = Auth::user()->subscribeTo(Plan::first(), pivotAttributes: [
    'metadata' => ['is_cool' => true]
]);

$subscription->subscribable->metadata('is_cool'); // "true"
```

### Postponed beginning

When a subscription is created, it starts immediately. You may want to defer the starting moment using `postponedTo()`, which moves the subscription starting date forward by an amount of days. You can also use a function that should return the moment when it will begin. Until then, the subscription won't be active.

```php
use Laragear\Subscriptions\Facades\Plan;
use Illuminate\Support\Facades\Auth;

if (Auth::user()->cannot('subscribeTo', $plan)) {
    return "You cannot subscribe to the plan $plan->name.";
}

$subscription = Auth::user()->subscribeTo($plan)->postponedTo(function ($start) {
    return $start->addDay()->setTime(9, 0);
});

$subscription->save();
```

When postponing the subscription, the user will be still subscribed to the Plan, but the subscription won't be considered _active_ until the postponed date.

If you're looking to only change when the cycles should be calculated from, you may want to [push the cycle start forward](#push-cycle-start). 

### Push cycle start

The Subscription begins from the exact moment is created. You can "move" the subscription cycle to accommodate a date (like, from next monday) using `pushedTo()` and the number of days, or a function that returns the date to extend to and the start the cycle.

For example, let's assume a user subscribes on Friday 24th at 12:30 PM. By default, the next monthly cycle will start the same day at the next month. With `pushedTo()`, we can push the cycle start to Monday 27th 00:00, giving him the whole weekend "for free".

```php
use Laragear\Subscriptions\Facades\Plan;
use Illuminate\Support\Facades\Auth;

if (Auth::user()->cannot('subscribeTo', $plan)) {
    return "You cannot subscribe to the plan $plan->name.";
}

$subscription = Auth::user()
    ->subscribeTo($plan)
    ->pushedTo(function ($start) {
        return $start->nextMonday();
    });

$subscription->save();
```

During this _extended_ time, the Subscription will be considered active.

> [!TIP]
> 
> If you need to move the subscription start instead of pushing the start, you can [postpone it](#postponed-beginning). 

### Modifying the Interval

If you need to change the subscription interval to be different from the parent Plan, use `updateInterval()` with the interval you want to force into the Subscription.

```php
use Laragear\Subscriptions\Models\Plan;
use Illuminate\Support\Facades\Auth;
use Carbon\CarbonInterval;

// Subscribe the user to a monthly plan.
$subscription = Auth::user()->subscribeTo(Plan::find('b6954722...'));

// Replace the monthly interval to yearly.
$subscription->updateInterval(CarbonInterval::year());
```

> [!DANGER]
> 
> This only works with freshly created subscriptions. Forcefully changing the interval **after** the first cycle will corrupt the cycle count and start/end dates. 

### Retrieving the subscription

Once a subscription is created, it can be accessed with the `subscription` property, like a normal Eloquent Relationship. This method automatically retrieves **the latest active** subscription for the entity, or returns `null` if there is none.

```php
use App\Models\User;

$user = User::find(1);

$subscription = $user->subscription;
```

For the case of users attached to multiple subscriptions, it always returns the latest subscribed from all active. You can use the ID of a subscription to retrieve it directly, as long it's active. It will also return `null` if there is no active Subscription found.

```php
use App\Models\User;

$user = User::find(1);

$subscription = $user->subscription('bc728326...');
```

#### Retrieving all subscriptions

You can find all subscriptions for an entity using the `subscriptions()` method, just like any Eloquent Relationship. This will actually find all subscriptions, include those expired, cancelled, and future.

```php
use App\Models\User;

$user = User::find(1);

$subscriptions = $user->subscriptions()->get();
```

Since using this method alone can be a little hectic, the relation includes some useful scopes to filter the records to retrieve:

| Method           | Description                                              |
|------------------|----------------------------------------------------------|
| `active()`       | Filters for active (not expired) subscriptions.          |
| `expired()`      | Filters for expired (inactive) subscriptions.            |
| `soon()`         | Filters for subscriptions yet to start.                  |
| `cancelled()`    | Filters for subscriptions that where actively cancelled. |
| `notCancelled()` | Filters for subscriptions that where not cancelled.      |

For example, you can use the scopes to retrieve subscriptions that where cancelled and no longer active.

```php
use Laragear\Subscriptions\Models\Subscription;

$cancelled = Subscription::cancelled()->expired()->get();
```

Also, Subscriptions do have timestamps, so you can use `oldest()` to query your subscriptions like any other Eloquent Model.

```php
use Laragear\Subscriptions\Models\Subscription;

$oldestExpired = Subscription::oldest()->expired()->get();
```

### Renew / Extend

To renew a subscription for another cycle, use the `renewOnce()` method. It will _extend_ the subscription one cycle.

```php
use Illuminate\Support\Facades\Auth;

$user = Auth::user();

echo $user->subscription->ends_at; // "2020-02-28 23:59:59"

echo $user->subscription->renewOnce()->ends_at; // "2020-03-31 23:59:59"
```

You may also extend a subscription for more than one cycle using `renewBy()` along the number of cycles.

```php
echo $user->subscription->renewBy(2)->ends_at; // "2020-04-30 23:59:59"
```

> [!IMPORTANT]
> 
> Since renewing a subscription removes the past [grace period](#grace-period), ensure you call `graceTo()` after renewing or [upgrading/downgrading](#upgrading--downgrading).

### Grace period

When a Subscription is not renewed, the subscription will expire at the end of the cycle, rendering it non-active. This can be relatively problematic on some scenarios, for example, when you don't expect users to renew their subscriptions in your premises on a weekend when the store is closed. To avoid this, you can set a grace period in which the subscription will stay active after the cycle has ended.

You can use `graceTo()` with the amount of days to take as a grace period, or a function that returns the time the grace period should end.

```php
use Illuminate\Support\Facades\Auth;

Auth::user()
    ->subscription
    ->renewOnce()
    ->graceTo(function ($start, $end) {
        return $end->nextFriday()->endOfDay();
    })
    ->save();
```

If a subscription has a grace period or not, you can use `hasGracePeriod()` and `doesntHaveGracePeriod()`, respectively:

```php
if ($subscription->hasGracePeriod()) {
    return 'Your subscription will terminate ' . $subscription->ends_at->diffForHumans();
}
```

You can check if the subscription is on grace period or not using `isOnGracePeriod()` and `isNotOnGracePeriod()`, respectively:

```php
if ($subscription->isOnGracePeriod()) {
    return 'Your subscription already expired. Renew it to not lose access.'
}
```

> [!WARNING] 
> 
> Ensure that you use `graceTo()` after [renewing it](#renew--extend), as a renewal will invalidate the previous grace period.

### Unsubscribe / Terminate

To terminate a plan, use the `terminate()` method over it. This cancels the subscription immediately, and cuts short any grace period it may have to the moment of termination.

```php
use Laragear\Subscriptions\Facades\Plan;
use Laragear\Subscriptions\Models\Subscription;
use Illuminate\Support\Facades\Auth;

$user = Auth::user();

if ($user->cannot('terminate', $user->subscription)) {
    return "You don't have permissions to terminate this subscription.";
}

$user->subscription->terminate();

echo $user->subscription->isActive(); // false
echo $user->subscription->isCancelled(); // true
```

> [!IMPORTANT]
> 
> Terminating a Subscription doesn't detach the admin from it, but it will detach all other subscribers. This can be disabled with `terminateWithoutDetaching()` over the subscription
> 
> ```php
> $user->subscription->terminateWithoutDetaching();
> ```
> 

#### Cancellation mark

Most of the time you will want to let the subscription expire instead of cancelling it immediately. To help to detect when users don't want to automatically renew their subscriptions, use the `cancel()` method, which sets a flag on the Subscription that later you can check with `isCancelled()`.

```php
use Illuminate\Support\Facades\Auth;

$user = Auth::user();

$user->subscription->cancel()->save();

echo $user->subscription->isActive(); // true
echo $user->subscription->isCancelled(); // true
```

This is mostly a helper so later these subscriptions cannot be renewed automatically.

```php
$user = Auth::user();

if ($user->subscription->isCancelled()) {
    return "Your subscription won't be renewed. You will lose access at {$user->subscription->ends_at}."
}
```

### Modifying the subscription

Most of the subscription data is copied from its parent [Plan](#plans). Laragear's Subscriptions package automatically copies over the Plan data into the Subscription model, which allows you to modify a particular Subscription, isolating its data from the parent Plan itself and other similar Subscriptions.

```php
$subscription->capabilities->set('deliveries', 20);

echo $subscription->capability('deliveries'); // 20
echo $subscription->plan->capability('deliveries'); // 10
```

You're free to edit the Subscription as you see fit. Remember calling `save()` to push the changes into the database permanently.

### Moving the cycle start

Moving the subscription cycle start it's usually a big problem, but sometimes you will need to change it for legal reasons or because a particular demand. The way Laragear's Subscription package deals with moving the cycle start is by **extending the current cycle until the new date**.

You only need to use `adjustStartTo()` with a new future date. The subscription will change at the date issued.

```php
use Illuminate\Support\Facades\Auth;

$user = Auth::user();

$user->subscription->adjustStartTo(now()->day(5));

$user->subscription->save();

return 'Your subscriptions is now billed the 5th of each month.';
```

This method returns a `CarbonInterval` instance. You can use that, for example, to calculate how much to charge up-front or at the next billing date, based on the costs of the subscription.

In the following example, the user will be charged up-front for the extended date, before persisting the cycle change.

```php
use Illuminate\Support\Facades\Auth;

// Today is 1st of January. We need the cycle to start from the 5th of each month. 
$extended = Auth::user()->subscription->adjustStartTo(now()->day(5));

// Multiply the days extended by the cost-per-day of the monthly subscription.
$cost = $extended->totalDays * (29.90 / 30); // 3.98, if you're curious.

return "You will be charged $ {$cost} to accommodate the cycle change. Next cycle will be charged at the normal rate.";
```

### Shared subscriptions

Normally, one entity will be tied to one subscription, but you can attach multiple entities to a subscription, likely to create "Family" or "Teams" based Subscriptions. You can use `attach()` over the Subscription with entity to attach. It also accepts an optional array of attributes to set in the pivot table.

This is useful to mix with an [authorization gate](#authorization) to check if the plan allows to be shared, and there are slots free to attach the entities.

```php
use App\Models\User;
use Laragear\Subscriptions\Models\Subscription;
use Illuminate\Support\Facades\Auth;

$admin = Auth::user();
$guests = User::find(4);

if ($admin->cannot('attachTo', [$admin->subscription, $guests])) {
    return 'Only admins can attach users to this subscriptions.';
}

$admin->subscription->attach($guests);
```

To remove an entity from the subscription, use `detach()` over the subscription and the entity.

```php
use App\Models\User;
use Laragear\Subscriptions\Facades\Subscription;
use Illuminate\Support\Facades\Auth;

$admin = Auth::user();
$guest = User::find(4);

$subscription = Subscription::find('bc728326...');

if ($admin->cannot('detachFrom', [$admin->subscription, $guest])) {
    return 'Only admins can attach users to this subscriptions.';
}

$admin->subscription->detach($guest);
```

### Used and unused time

You can easily get how much of the cycle has been used or remains using `used()` and `unused()`, respectively. These return a `CarbonInterval` with the difference in time.

For example, we can use the `unused()` interval to calculate how much an upgrade will cost for other Plan by subtracting the current cost of the unused days.

```php
$user = User::find(1);

$newPlanCost = 49.99

$price = $newPlanCost - $user->subscription()->unused()->days * 29.99;

return "Upgrade now and pay a reduced price of $ $price.";
```

> [!IMPORTANT]
> 
> Intervals for `used()` and `unused()` return empty intervals if the subscription hasn't started or already ended.

### Upgrading / Downgrading

To upgrade or downgrade a subscription, use the `changeTo()` method with the Plan you want to change to, over the entity that's subscribed.

```php
use App\Models\User;
use Laragear\Subscriptions\Models\Plan;

$user = User::find(1);

$plan = Plan::find('b6954722...');

if ($user->cant('upgradeTo', $plan)) {
    return "You cannot upgrade to the plan $plan->name";
}

$upgraded = $user->changeTo($plan);
```

> [!TIP]
> 
> This will automatically migrate all attached entities to the new Subscription, and keep the same _admin_.

You can also change a Subscription to another Plan manually using the `Change` facade.

```php
use Laragear\Subscriptions\Facades\Change;
use Laragear\Subscriptions\Models\Plan;
use Laragear\Subscriptions\Models\Subscription;

$plan = Plan::find('b6954722...');

$new = Change::to(Subscription::find('bc728326...'), $plan);
```

> [!IMPORTANT]
> 
> Changing a Subscription to another Plan creates a new Subscription. It doesn't change the current Subscription. Ensure you [set a grace period](#grace-period) if needed to.

## Subscription Capabilities

Subscriptions and Plans contain capabilities. This is copied directly from the Plan itself, and allows checking if an entity has certain power to do something in your app.

You can check capabilities using `capability()` which returns the value of the capability. It also accepts a default value to return if the capability doesn't exist.

```php
use Illuminate\Support\Facades\Auth;

$user = Auth::user();

$count = $user->deliveries()->forCurrentMonth()->count();

$max = $user->subscription->capability('deliveries', 0);

return "You have made $count deliveries out of $max for this month.";
```

You can use `capabilities` property directly to set a new value. For example, we can make a lottery to upgrade the deliveries amount for the winners.

```php
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Lottery;

$subscription = Auth::user()->subscription;

return Lottery::odds(1, 20)
    ->winner(function () use ($subscription) {
        $subscription->capabilities->set('deliveries', 20)->save();
        $subscription->save();

        return 'You are the lucky winner! You now have 20 deliveries forever!';
    })
    ->loser(fn() => 'Better luck next time!')
    ->choose();
```

You can also rewrite the capabilities entirely by setting the `capabilities` property using an array.

```php
$user->subscription->capabilities = [
    'deliveries' => 20,
    'priority' => 'low'
];

$user->subscription->save();
```

### Checking capabilities

To avoid using complex syntax, you can use convenient methods to check for the capabilities of a subscription. Use `check()` along the capability key in `dot.notation` and one of the self-explanatory methods to check if a condition is true or false.

```php
$user = Auth::user();

if ($user->subscription->hasDisabled('delivery.express')) {
    return 'Your subscription does not support Express Deliveries';
}
```

These are the following methods you can chain to a check

| Method      | Description                                      |
|-------------|--------------------------------------------------|
| hasEnabled  | Checks the capability exists and is truthy.      |
| canUse      | Checks the capability exists and is truthy.      |
| hasDisabled | Checks the capability doesn't exist or is falsy. |
| cannotUse   | Checks the capability doesn't exist or is falsy. |
| cantUse     | Checks the capability doesn't exist or is falsy. |
| isBlank     | Checks the capability value is "blank".          |
| isFilled    | Checks the capability value is "filled".         |

You can also fluently compare values using the `check()` method:

```php
$user = Auth::user();

$count = $user->deliveries()->forCurrentMonth()->count();

if ($user->subscription->check($count)->exceeds('deliveries')) {
    return 'You have depleted all your available deliveries for this month.';
}
```

| Method                              | Description                                                         |
|-------------------------------------|---------------------------------------------------------------------|
| `isGreaterThan($capability)`        | Checks if the value is greater than the named capability.           |
| `exceeds($capability)`              | Checks if the value is greater than the named capability.           |
| `isEqualOrGreaterThan($capability)` | Checks if the value is equal or greater than the named capability.  |
| `is($capability)`                   | Checks if the value is equal to the named capability, strictly.     |
| `isNot($capability)`                | Checks if the value is not equal to the named capability, strictly. |
| `isSameAs($capability)`             | Checks if the value is equal to the named capability.               |
| `isEqualOrLessThan($capability)`    | Checks if the value is equal or less than the named capability.     |
| `doesntExceeds($capability)`        | Checks if the value is equal or less than the named capability.     |
| `isLessThan($capability)`           | Checks if the value is less than the named capability.              |

## Authorization

This package includes a lot of helpers to authorize actions over Plans and Subscriptions. All of those are managed via the `App\Policies\SubscriptionPolicy` and `App\Policies\PlanPolicy` [policy classes](https://laravel.com/docs/11.x/authorization#creating-policies) that should be already [installed](#1-install-the-files) in `app\Policies`.

| Method                               | Description                                                          |
|--------------------------------------|----------------------------------------------------------------------|
| `'subscribe', Plan::class`           | Determines that the user can subscribe to any Plan.                  |
| `'subscribeTo', $plan`               | Determines that the user can subscribe to a given Plan.              |
| `'upgradeTo', $plan`                 | Determines that the user subscription can upgrade to a given Plan.   |
| `'downgradeTo', $plan`               | Determines that the user subscription can downgrade to a given Plan. |
| `'renew', $subscription`             | Determines that the user can renew the Subscription.                 |
| `'cancel', $subscription`            | Determines that the user can cancel the Subscription.                |
| `'terminate', $subscription`         | Determines that the user can terminate the subscription.             |
| `'attachTo', $subscription, $user`   | Determines that the user can attach an user to the subscription.     |
| `'detachFrom', $subscription, $user` | Determines that the user can detach an user from the subscription.   |

This allows you to use authorization gates directly in your application, like inside controllers or views.

```blade
@can('terminate', $subscription)
    <a href="/subscription/terminate">Terminate subscription</a>
@else
    You are not allowed to terminate the subscription.
@endcan
```

You're free to change them as you see fit for your application. For example, you can add a `hasUpgradeDiscount()` gate to check if a subscription upgrade is eligible for a discount.

```php
// app\Policies\SubscriptionPolicy.php
public function hasUpgradeDiscount($user, Subscription $subscription): bool
{
    return $subscription->isActive() 
        && $subscription->isNotOnGracePeriod() 
        && $subscription->isNotFirstCycle();
}
```

Then, you can easily use the newly created gate in your application:

```blade
@can('haveUpgradeDiscount', $user->subscription)
    You're eligible for an upgrade discount of $ {{ $discount }}.
@endcan
```

## Pruning subscribers

When plans or subscription are deleted, the pivot table `subscribables` may contain orphaned records. To prune them, use `subscriptions:prune`.

```shell
php artisan subscriptions:prune

# Removed 10 dangling records pointing to deleted plans or subscriptions.
```

## Events

This package fires the following self-explanatory events:

| Event                    | Description                                                   |
|--------------------------|---------------------------------------------------------------|
| `OnDemandPlanCreated`    | A new on-demand Plan was created.                             |
| `SubscriberAttached`     | An existing subscription was attached to an entity.           |
| `SubscriberDetached`     | An existing subscription was detached to an entity.           |
| `SubscriptionCancelled`  | An existing subscription was marked as cancelled.             |
| `SubscriptionChanged`    | An existing subscription was changed to another Plan.         |
| `SubscriptionCreated`    | A new subscription was created.                               |
| `SubscriptionDowngraded` | An existing subscription was replaced for a new "worse" one.  |
| `SubscriptionRenewed`    | An existing subscription was renewed for a new cycle.         |
| `SubscriptionTerminated` | An existing subscription was terminated.                      |
| `SubscriptionUpgraded`   | An existing subscription was replaced for a new "better" one. |

## Polymorphic relations

To use polymorphic relations, use the `Subscription::addPolymorphicRelations()` to add the models you plan to attach to the Subscription model. You can do this in your `AppServiceProvider` or `bootstrap/app.php` file.

```php
use App\Models\Director;
use App\Models\Producer;
use App\Models\Actor;
use Illuminate\Foundation\Application;
use Laragear\Subscriptions\Models\Subscription;

return Application::configure(basePath: dirname(__DIR__))
    ->booted(function () {
        Subscription::macroPolymorphicRelations([
            'producers' => Producer::class,
            'actors' => Actor::class,
            'director' => Director::class,
        ])
    )
    ->create();
```

Once it's done, you can easily access these polymorphic relations like you would normally do using Eloquent Models. These new methods you register will even support [eager-loading](https://laravel.com/docs/11.x/eloquent-relationships#eager-loading).

```php
use Laragear\Subscriptions\Models\Subscription;

$subscription = Subscription::with('actors')->find('bc728326...');

foreach ($subscription->actors as $actor) {
    echo $actor->name;
};
```

If you like to have manual control on the dynamic relations, you can also use an array with both the model class name and a callback to use as a relation builder.

```php
use Laragear\Subscriptions\Models\Subscription;
use App\Models\Producer;
use App\Models\Actor;

public function boot()
{
    Subscription::macroPolymorphicRelations([
        'actors' => Actor::class,
        'producers' => [
            Producer::class, 
            function($subscription)  {
                return $subscription
                    ->morphedByMany(Producer::class, 'subscribable')
                    ->credited();
            }
        ];
    ]);
    
    // ...
}
```

While you can use [dynamic relationships](https://laravel.com/docs/11.x/eloquent-relationships#dynamic-relationships) directly,these methods add more data to the relationship, and makes the Subscription model aware of the relationship nature (monomorphic or polymorphic).

### Retrieving polymorphic records manually

If you're using polymorphic subscribers, the default model is `App\Models\User`, so the `subscribers` property will retrieve all models for that polymorphic relation, unless you change it to other model.

```php
use Laragear\Subscriptions\Models\Subscription;

$subscription = Subscription::find('bc728326...');

$users = $subscription->subscribers;
```

When not using the default model, you will need to use the `subscribers()` with the model name or instance to retrieve the different types of subscribers.

```php
use Laragear\Subscriptions\Models\Subscription;
use App\Models\Cameraman;
use App\Models\Photographer;

$subscription = Subscription::find('bc728326...');

$cameramen = $subscription->subscribers(Cameraman::class)->get();
$photographers = $subscription->subscribers(Photographer::class)->get();
```

## Testing

This package includes a `PlanFactory` and `SubscriptionFactory`, you can easily access from their respective models.

```php
use Laragear\Subscriptions\Models\Plan;
use App\Models\User;

$plan = Plan::factory()->called('test')->monthly()->createOne();
```

For the case of the Subscription Factory, you may want to attach entities after creating them using the `attach()` method.

```php
use Laragear\Subscriptions\Models\Subscription;
use App\Models\Company;

$company = Company::find(1);

$subscription = Subscription::factory()
    ->called('test')
    ->monthly()
    ->createOne();

$subscription->attach($company);
```

## Laravel Octane compatibility

- The only singleton registered is the Plan Bag. On-demand plans removes themselves from the bag once created.
- There are no singletons using a stale config instance.
- There are no singletons using a stale request instance.
- There are no static properties written during a request.

There should be no problems using this package with Laravel Octane.

## Security

If you discover any security related issues, please email darkghosthunter@gmail.com instead of using the issue tracker.

# License

This specific package version is licensed under the terms of the [MIT License](LICENSE.md), at time of publishing.

[Laravel](https://laravel.com) is a Trademark of [Taylor Otwell](https://github.com/TaylorOtwell/). Copyright © 2011-2025 Laravel LLC.
