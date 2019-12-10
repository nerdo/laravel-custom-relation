# Laravel Custom Relation

A custom relation for when stock relations aren't enough.

## Use this if...

* None of the stock Relations fit the bill. (BelongsToManyThrough, etc)

## Installation

The recommended way to install is with [composer](http://getcomposer.org/):

```shell
composer require nerdo/laravel-custom-relation
```

## Example

Let's say we have 3 models:

- `User`
- `Role`
- `Permission`

Let's also say `User` has a many-to-many relation with `Role`, and `Role` has a many-to-many relation with `Permission`.

So their models might look something like this. (I kept them brief on purpose.)

```php
class User
{
    public function roles() {
        return $this->belongsToMany(Role::class);
    }
}
```
```php
class Role
{
    public function users() {
        return $this->belongsToMany(User::class);
    }

    public function permissions() {
        return $this->belongsToMany(Permission::class);
    }
}
```
```php
class Permission
{
    public function roles() {
        return $this->belongsToMany(Role::class);
    }
}
```

**What if you wanted to get all the `Permission`s for a `User`, or all the `User`s with a particular `Permission`?** There no stock Relation in Laravel to descibe this. What we need is a `BelongsToManyThrough` but no such thing exists in stock Laravel.

## Solution

First, make sure your models are using the `HasCustomRelations` trait. Then, define custom relations like this.

```php
use Nerdo\LaravelCustomRelation\CustomRelations;

class User
{
    use CustomRelations;

    /**
     * Get the related permissions
     *
     * @return Illuminate\Database\Eloquent\Relations\Relation
     */
    public function permissions()
    {
        return $this->customRelation(
            Permission::class,

            // add constraints
            function ($relation) {
                $relation->getQuery()
                    // join the pivot table for permission and roles
                    ->join('permission_role', 'permission_role.permission_id', '=', 'permissions.id')
                    // join the pivot table for users and roles
                    ->join('role_user', 'role_user.role_id', '=', 'permission_role.role_id')
                    // for this user
                    ->where('role_user.user_id', $this->id);
            },

            // add eager constraints
            function ($relation, $models) {
                $relation->getQuery()->whereIn('role_user.user_id', $relation->getKeys($models));
            }
        );
    }
}
```

```php
use Nerdo\LaravelCustomRelation\CustomRelations;

class Permission
{
    use CustomRelations;

    /**
     * Get the related users
     *
     * @return Illuminate\Database\Eloquent\Relations\Relation
     */
    public function users()
    {
        return $this->customRelation(
            User::class,

            // constraints
            function ($relation) {
                $relation->getQuery()
                    // join the pivot table for users and roles
                    ->join('role_user', 'role_user.user_id', '=', 'users.id')
                    // join the pivot table for permission and roles
                    ->join('permission_role', 'permission_role.role_id', '=', 'role_user.role_id')
                    // for this permission
                    ->where('permission_role.permission_id', $this->id);
            },

            // eager constraints
            function ($relation, $models) {
                $relation->getQuery()->whereIn('permission_role.permission_id', $relation->getKeys($models));
            }
        );
    }
}
```

You could now do all the normal stuff for relations without having to query in-between relations first.

## A more complex example with eager loading.

In this scenario there is a Client model with notes, but the notes may be attributed to the client using the `client_id` or an external `crm_id`, so we use the custom relation to define the relationship.

```php
use Nerdo\LaravelCustomRelation\CustomRelations;

class Client
{
    use CustomRelations;

    public function notes()
    {
        $notesTable = (new Note)->getTable();
        $clientsTable = $this->getTable();

        return $this->customRelation(
            Note::class,

            // constraints
            function ($relation) use ($notesTable, $clientsTable) {
                $relation
                    ->getQuery()
                    ->select($notesTable . '.*')
                    ->join(
                        $clientsTable,
                        function ($join) use ($clientsTable, $notesTable) {
                            $join
                                ->on($clientsTable . '.client_id', '=', $notesTable . '.client_id')
                                ->orOn($clientsTable . '.crm_id', '=', $notesTable . '.crm_id');
                            if ($this->id) {
                                $join->where($clientsTable . '.id', '=', $this->id);
                            }
                        }
                    );
            },

            // eager constraints
            function ($relation, $models) use ($clientsTable) {
                $clients = collect($models);
                $relation
                    ->getQuery()
                    ->whereIn(
                        $clientsTable . '.client_id',
                        $clients
                            ->map(function ($client) {
                                return $client->client_id;
                            })
                            ->filter()
                            ->values()
                            ->all()
                    )
                    ->orWhereIn(
                        $clientsTable . '.crm_id',
                        $clients
                            ->map(function ($client) {
                                return $client->crm_id;
                            })
                            ->filter()
                            ->values()
                            ->all()
                    );
            },

            // eager matcher
            function (array $models, \Illuminate\Support\Collection $results, $relation, $customRelation) {
                $buildDictionary = function (\Illuminate\Support\Collection $results) {
                    return $results
                        ->reduce(
                            function ($dictionary, $current) {
                                if (!($current instanceof Note)) {
                                    return $dictionary;
                                }
                                if ($current->client_id) {
                                    $key = 'client_id_' . $current->client_id;
                                    $dictionary[$key] = $dictionary[$key] ?? [];
                                    $dictionary[$key][] = $current;
                                }
                                if ($current->crm_id) {
                                    $key = 'crm_id_' . $current->crm_id;
                                    $dictionary[$key] = $dictionary[$key] ?? [];
                                    $dictionary[$key][] = $current;
                                }
                                return $dictionary;
                            },
                            []
                        );
                };

                $dictionary = collect($buildDictionary($results));
                $related = $customRelation->getRelated();

                // Once we have the dictionary we can simply spin through the parent models to
                // link them up with their children using the keyed dictionary to make the
                // matching very convenient and easy work. Then we'll just return them.
                foreach ($models as $model) {
                    $cKey = 'client_id_' . $model->getAttribute('client_id');
                    $crmKey = 'crm_id_' . $model->getAttribute('crm_id');

                    $matches = $dictionary
                        ->filter(function ($value, $key) use ($cKey, $crmKey) {
                            return $key === $cKey || $key === $crmKey;
                        })
                        ->values()
                        ->flatten()
                        ->unique()
                        ->all();

                    if ($matches) {
                        $model->setRelation($relation, $related->newCollection($matches));
                    }
                }

                return $models;
            }
        );
    }
}
```
