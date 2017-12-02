### What is global scope

I am just copying original description of global scope from the Laravel's official site. Which is

> Global scopes allow you to add constraints to all queries for a given model.

Yes, it's very handy tool and can be effective in many cases. For example, you have the notification system in your app and you want that whenever you fetch notifications from the database, you only get notification of currently logged-in user or you want to fetch only unread notifications without specifying same constraint every time.

In short, global scope is a where condition that you want to apply on every query of the model.

As Laravel's own soft delete functionality uses the global scope, I have briefly explained it in my previous article [How Soft Delete works in Laravel 5.3 or 5.4\.](http://www.linkedin.com/pulse/how-softdelete-works-laravel-53-taukeer-liaqat)

## How Global Scope works

If you want to add global in your model, you should implement "_Illuminate\Database\Eloquent\Scope.php" interface._

_File: Illuminate\Database\Eloquent\Scope.php_


```php
interface Scope {
    public function apply(Builder $builder, Model $model);
}
```

In this apply method, you can add any condition to your model's query. For example, in the case of fetching only unread notifications, it can be


```php
public class NotificationScope implements Scope {

  ﻿public function apply(Builder $builder, Model $model) {
      $builder->where('is_read', false);
  }

}
```


Let's see how does Laravel call this function.

Whenever you want to apply global scope to your model, you add the following line in the _boot_ method of the model.

```php
protected static function boot()
{
    parent::boot();

    static::addGlobalScope(new NotificationScope);
}
```

File: _Illuminate\Database\Eloquent\Model.php_

```php
public static function addGlobalScope($scope, Closure $implementation = null) {
    if (is_string($scope) && ! is_null($implementation)) {
        return static::$globalScopes[static::class][$scope] = $implementation;
    }

    if ($scope instanceof Closure) {
        return static::$globalScopes[static::class][spl_object_hash($scope)] = $scope;
    }

    if ($scope instanceof Scope) {
        return static::$globalScopes[static::class][get_class($scope)] = $scope;
    }

    throw new InvalidArgumentException('Global scope must be an instance of Closure or Scope.');
}
```

What global scope method say is, you can add global scope to your model in multiple ways. But in every case, it will add a new entry to the static property _$globalScopes as a key-value pair, where the key is model's class name and the value is global scope's name._

### When does Laravel apply global scopes

Laravel applies global scopes in the _newQuery_ method.

This method is called whenever we try to retrieve a new record from the database.

Basically it is called from eloquent's get method which itself is called by model's data retrieving methods like first, find, all, or by the user himself after any _where_ clause.

File: _Illuminate\Database\Eloquent\Builder.php_


```php
public function get($columns = ['*']){

  $builder = $this->applyScopes();

  $models = $builder->getModels($columns);

  if (count($models) > 0) {
    $models = $builder->eagerLoadRelations($models);
  }

  return $builder->getModel()->newCollection($models);

}
```

This method calls _applyScopes_

File: _Illuminate\Database\Eloquent\Builder.php_

```php
public function applyScopes(){

  if (! $this->scopes) {
    return $this;
  }

  $builder = clone $this;

  foreach ($this->scopes as $scope) {

    $builder->callScope(function (Builder $builder) use ($scope) {
      if ($scope instanceof Closure) {
        $scope($builder);
      } elseif ($scope instanceof Scope) {
        $scope->apply($builder, $this->getModel());
      }

    });
  }
  return $builder;
}
```


There are multiple things to notice here.

### 1) $this->scopes

First things worth to be noticed here is that whenever you apply global scope to your model, _**addGlobalScope method**_ (explained above) add entries to the $globalScopes property of the model. And here we are trying to get our global scopes from builder's _**scope**_ property.

What going on here is, Whenever we build the query for any model, function _newQuery()_ is called.

File: _Illuminate\Database\Eloquent\Model.php_

```
public function newQuery(){
  $builder = $this->newQueryWithoutScopes();

  foreach ($this->getGlobalScopes() as $identifier => $scope) {

    $builder->withGlobalScope($identifier, $scope);

  }

  return $builder;
}
```

The method of our interest is _**withGlobalScope**_

File: _Illuminate\Database\Eloquent\Builder.php_


```php
public function withGlobalScope($identifier, $scope) {
    $this->scopes[$identifier] = $scope;

    if (method_exists($scope, 'extend')) {
        $scope->extend($this);
    }

    return $this;
}
```

Here we are setting scope property of builder which is same as model's _**globalScopes**_ with the key-value pair.

**2) $builder->callScope:** In short, this method executes the closure passed to it.

**3) $scope->apply:** this method calls _apply_ method of the global scope where we have added the constraint to be applied to model's query.

### Anonymous Global Scopes

If you have applied global scope through closure, _**$scope($builder)** _will be called which is the closure you have registered as global scope.

Happy coding :)
