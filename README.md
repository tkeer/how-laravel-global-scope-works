### What is global scope

I am just copying original description of global scope from the Laravel's official site. Which is

> Global scopes allow you to add constraints to all queries for a given model.

Yes, it's very handy tool and can be effective in many cases. For example, you have the notification system in your app and you want that whenever you fetch notifications from the database, you only get notification of currently logged-in user or you want to fetch only unread notifications without specifying same constraint every time.

In short, global scope is a where condition that you want to apply on every query of the model.

As Laravel's own soft delete functionality uses the global scope, I have briefly explained it in my previous article [How Soft Delete works in Laravel 5.3 or 5.4\.](http://www.linkedin.com/pulse/how-softdelete-works-laravel-53-taukeer-liaqat)

## How Global Scope works

If you want to add global in your model, you should implement "_Illuminate\Database\Eloquent\Scope.php" interface._

_File: Illuminate\Database\Eloquent\Scope.php_

<pre spellcheck="false"><span class="hljs-class"><span class="hljs-keyword">interface</span> <span class="hljs-title">Scope</span></span> {
    <span class="hljs-keyword">public</span> <span class="hljs-function"><span class="hljs-keyword">function</span> <span class="hljs-title">apply</span><span class="hljs-params">(Builder $builder, Model $model)</span></span>;
}
</pre>

In this apply method, you can add any condition to your model's query. For example, in the case of fetching only unread notifications, it can be

<pre spellcheck="false"><span class="hljs-keyword">public</span> <span class="hljs-class"><span class="hljs-keyword">class</span> <span class="hljs-title">NotificationScope</span> <span class="hljs-keyword">implements</span> <span class="hljs-title">Scope</span></span> {

  ﻿<span class="hljs-keyword">public</span> <span class="hljs-function"><span class="hljs-keyword">function</span> <span class="hljs-title">apply</span><span class="hljs-params">(Builder $builder, Model $model)</span></span> {
      $builder->where(<span class="hljs-string">'is_read'</span>, <span class="hljs-keyword">false</span>);
  }

}
</pre>

Let's see how does Laravel call this function.

Whenever you want to apply global scope to your model, you add the following line in the _boot_ method of the model.

<pre spellcheck="false">protected static function boot()
{
    <span class="hljs-attribute">parent</span>::<span class="hljs-built_in">boot</span>();

    <span class="hljs-attribute">static</span>::<span class="hljs-built_in">addGlobalScope</span>(new NotificationScope);
}
</pre>

File: _Illuminate\Database\Eloquent\Model.php_

<pre spellcheck="false"><span class="hljs-keyword">public</span> <span class="hljs-keyword">static</span> <span class="hljs-function"><span class="hljs-keyword">function</span> <span class="hljs-title">addGlobalScope</span><span class="hljs-params">($scope, Closure $implementation = null)</span></span> {
    <span class="hljs-keyword">if</span> (is_string($scope) && ! is_null($implementation)) {
        <span class="hljs-keyword">return</span> <span class="hljs-keyword">static</span>::$globalScopes[<span class="hljs-keyword">static</span>::class][$scope] = $implementation;
    }

    <span class="hljs-keyword">if</span> ($scope <span class="hljs-keyword">instanceof</span> Closure) {
        <span class="hljs-keyword">return</span> <span class="hljs-keyword">static</span>::$globalScopes[<span class="hljs-keyword">static</span>::class][spl_object_hash($scope)] = $scope;
    }

    <span class="hljs-keyword">if</span> ($scope <span class="hljs-keyword">instanceof</span> Scope) {
        <span class="hljs-keyword">return</span> <span class="hljs-keyword">static</span>::$globalScopes[<span class="hljs-keyword">static</span>::class][get_class($scope)] = $scope;
    }

    <span class="hljs-keyword">throw</span> <span class="hljs-keyword">new</span> InvalidArgumentException(<span class="hljs-string">'Global scope must be an instance of Closure or Scope.'</span>);
}
</pre>

What global scope method say is, you can add global scope to your model in multiple ways. But in every case, it will add a new entry to the static property _$globalScopes as a key-value pair, where the key is model's class name and the value is global scope's name._

### When does Laravel apply global scopes

Laravel applies global scopes in the _newQuery_ method.

This method is called whenever we try to retrieve a new record from the database.

Basically it is called from eloquent's get method which itself is called by model's data retrieving methods like first, find, all, or by the user himself after any _where_ clause.

File: _Illuminate\Database\Eloquent\Builder.php_

<pre spellcheck="false"><span class="hljs-keyword">public</span> <span class="hljs-function"><span class="hljs-keyword">function</span> <span class="hljs-title">get</span><span class="hljs-params">($columns = [<span class="hljs-string">'*'</span>])</span></span>{

  $builder = <span class="hljs-keyword">$this</span>->applyScopes();

  $models = $builder->getModels($columns);

  <span class="hljs-keyword">if</span> (count($models) > <span class="hljs-number">0</span>) {
    $models = $builder->eagerLoadRelations($models);
  }

  <span class="hljs-keyword">return</span> $builder->getModel()->newCollection($models);

}
</pre>

This method calls _applyScopes_

File: _Illuminate\Database\Eloquent\Builder.php_

<pre spellcheck="false"><span class="hljs-keyword">public</span> <span class="hljs-function"><span class="hljs-keyword">function</span> <span class="hljs-title">applyScopes</span><span class="hljs-params">()</span></span>{

  <span class="hljs-keyword">if</span> (! <span class="hljs-keyword">$this</span>->scopes) {
    <span class="hljs-keyword">return</span> <span class="hljs-keyword">$this</span>;
  }

  $builder = <span class="hljs-keyword">clone</span> <span class="hljs-keyword">$this</span>;

  <span class="hljs-keyword">foreach</span> (<span class="hljs-keyword">$this</span>->scopes as $scope) {

    $builder->callScope(<span class="hljs-function"><span class="hljs-keyword">function</span> <span class="hljs-params">(Builder $builder)</span> <span class="hljs-title">use</span> <span class="hljs-params">($scope)</span></span> {
      <span class="hljs-keyword">if</span> ($scope instanceof Closure) {
        $scope($builder);
      } <span class="hljs-keyword">elseif</span> ($scope instanceof Scope) {
        $scope->apply($builder, <span class="hljs-keyword">$this</span>->getModel());
      }

    });
  }
  <span class="hljs-keyword">return</span> $builder;
}
</pre>

There are multiple things to notice here.

### 1) $this->scopes

First things worth to be noticed here is that whenever you apply global scope to your model, _**addGlobalScope method**_ (explained above) add entries to the $globalScopes property of the model. And here we are trying to get our global scopes from builder's _**scope**_ property.

What going on here is, Whenever we build the query for any model, function _newQuery()_ is called.

File: _Illuminate\Database\Eloquent\Model.php_

<pre spellcheck="false"><span class="hljs-keyword">public</span> <span class="hljs-function"><span class="hljs-keyword">function</span> <span class="hljs-title">newQuery</span><span class="hljs-params">()</span></span>{
  $builder = <span class="hljs-keyword">$this</span>->newQueryWithoutScopes();

  <span class="hljs-keyword">foreach</span> (<span class="hljs-keyword">$this</span>->getGlobalScopes() <span class="hljs-keyword">as</span> $identifier => $scope) {

    $builder->withGlobalScope($identifier, $scope);

  }

  <span class="hljs-keyword">return</span> $builder;
}
</pre>

The method of our interest is _**withGlobalScope**_

File: _Illuminate\Database\Eloquent\Builder.php_

<pre spellcheck="false"><span class="hljs-keyword">public</span> <span class="hljs-function"><span class="hljs-keyword">function</span> <span class="hljs-title">withGlobalScope</span><span class="hljs-params">($identifier, $scope)</span></span> {
    <span class="hljs-keyword">$this</span>->scopes[$identifier] = $scope;

    <span class="hljs-keyword">if</span> (method_exists($scope, <span class="hljs-string">'extend'</span>)) {
        $scope->extend(<span class="hljs-keyword">$this</span>);
    }

    <span class="hljs-keyword">return</span> <span class="hljs-keyword">$this</span>;
}
</pre>

Here we are setting scope property of builder which is same as model's _**globalScopes**_ with the key-value pair.

**2) $builder->callScope:** In short, this method executes the closure passed to it.

**3) $scope->apply:** this method calls _apply_ method of the global scope where we have added the constraint to be applied to model's query.

### Anonymous Global Scopes

If you have applied global scope through closure, _**$scope($builder)** _will be called which is the closure you have registered as global scope.

Happy coding :)
