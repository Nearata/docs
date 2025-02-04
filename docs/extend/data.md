# Working with Data

Data is the foundation of any forum, so you're going to need to play nice with it if you want your extension to do anything useful. This document runs through how data flows in Flarum, from the database to the JSON-API to the frontend, and all the way back again.

Flarum makes use of [Laravel's Database component](https://laravel.com/docs/database). You should familiarize yourself with it before proceeding, as it is assumed as prior knowledge in the following documentation.


## API Request Lifecycle

Before we go into detail about how to extend Flarum's data API, it's worth thinking about the lifecycle of a typical data request:

1. An HTTP request is sent to Flarum's API. Typically, this will come from the Flarum frontend, but external programs can also interact with the API. Flarum's API mostly follows the [JSON:API](https://jsonapi.org/) specification, so accordingly, requests should follow [said specification](https://jsonapi.org/format/#fetching).
2. The request is run through [middleware](middleware.md), and routed to the proper controller. You can learn more about controllers as a whole on our [routes and content documentation](routes.md). Assuming the request is to the API (which is the case for this section), the controller that handles the request will be a subclass of `Flarum\Api\AbstractSerializeController`.
3. Any modifications done by extensions to the controller via the [`ApiController` extender](#extending-api-controllers) are applied. This could entail changing sort, adding includes, changing the serializer, etc.
4. The `$this->data()` method of the controller is called, yielding some raw data that should be returned to the client. Typically, this data will take the form of a Laravel Eloquent model collection or instance, which has been retrieved from the database. That being said, the data could be anything as long as the controller's serializer can process it. Each controller is responsible for implementing its own `data` method. Note that for `PATCH`, `POST`, and `DELETE` requests, `data` will perform the operation in question, and return the modified model instance.
5. That data is run through any pre-serialization callbacks that extensions register via the [`ApiController` extender](#extending-api-controllers).
6. The data is passed through a [serializer](#serializers), which converts it from the backend, database-friendly format to the JSON:API format expected by the frontend. It also attaches any related objects, which are run through their own serializers. As we'll explain below, extensions can [add / override relationships and attributes](#attributes-and-relationships) at the serialization level.
7. The serialized data is returned as a JSON response to the frontend.
8. If the request originated via the Flarum frontend's `Store`, the returned data (including any related objects) will be stored as [frontend models](#frontend-models) in the frontend store.

## Migrations

If we want to use a custom model, or add attributes to an existing one, we will need to modify the database to add tables / columns. We do this via migrations.

Migrations are like version control for your database, allowing you to easily modify Flarum's database schema in a safe way. Flarum's migrations are very similar to [Laravel's](https://laravel.com/docs/migrations), although there are some differences.

Migrations live inside a folder suitably named `migrations` in your extension's  directory. Migrations should be named in the format `YYYY_MM_DD_HHMMSS_snake_case_description` so that they are listed and run in order of creation.

### Migration Structure

In Flarum, migration files should **return an array** with two functions: `up` and `down`. The `up` function is used to add new tables, columns, or indexes to your database, while the `down` function should reverse these operations. These functions receive an instance of the [Laravel schema builder](https://laravel.com/docs/6.x/migrations#creating-tables) which you can use to alter the database schema:

```php
<?php

use Illuminate\Database\Schema\Builder;

return [
    'up' => function (Builder $schema) {
        // up migration
    },
    'down' => function (Builder $schema) {
        // down migration
    }
];
```

For common tasks like creating a table, or adding columns to an existing table, Flarum provides some helpers which construct this array for you, and take care of writing the `down` migration logic while they're at it. These are available as static methods on the `Flarum\Database\Migration` class.

### Migration Lifecycle

Migrations are applied when the extension is enabled for the first time or when it's enabled and there are some outstanding migrations. The executed migrations are logged in the database, and when some are found in the migrations folder of an extension that aren't logged as completed yet, they will be executed. 

Migrations can also be manually applied with `php flarum migrate` which is also needed to update the migrations of an already enabled extension. To undo the changes applied by migrations, you need to click "Uninstall" next to an extension in the Admin UI, or you need to use the `php flarum migrate:reset` command. Nothing can break by running `php flarum migrate` again if you've already migrated - executed migrations will not run again.

There are currently no composer-level hooks for managing migrations at all (i.e. updating an extension with `composer update` will not run its outstanding migrations).

### Creating Tables

To create a table, use the `Migration::createTable` helper. The `createTable` helper accepts two arguments. The first is the name of the table, while the second is a `Closure` which receives a `Blueprint` object that may be used to define the new table:

```php
use Flarum\Database\Migration;
use Illuminate\Database\Schema\Blueprint;

return Migration::createTable('users', function (Blueprint $table) {
    $table->increments('id');
});
```

When creating the table, you may use any of the schema builder's [column methods](https://laravel.com/docs/6.x/migrations#creating-columns) to define the table's columns.

### Renaming Tables

To rename an existing database table, use the `Migration::renameTable` helper:

```php
return Migration::renameTable($from, $to);
```

### Creating/Dropping Columns

To add columns to an existing table, use the `Migration::addColumns` helper. The `addColumns` helper accepts two arguments. The first is the name of the table. The second is an array of column definitions, with the key being the column name. The value of each item is an array with the column definitions, as understood by Laravel's `Illuminate\Database\Schema\Blueprint::addColumn()` method. The first value is the column type, and any other keyed values are passed through to `addColumn`.

```php
return Migration::addColumns('users', [
    'email' => ['string', 'length' => 255, 'nullable' => true],
    'discussion_count' => ['integer', 'unsigned' => true]
]);
```

To drop columns from an existing table, use the `Migration::dropColumns` helper, which accepts the same arguments as the `addColumns` helper. Just like when dropping tables, you should specify the full column definitions so that the migration can be rolled back cleanly.

### Renaming Columns

To rename columns, use the `Migration::renameColumns` helper. The `renameColumns` helper accepts two arguments. The first is the name of the table, while the second is an array of column names to rename:

```php
return Migration::renameColumns('users', ['from' => 'to']);
```

### Data Migrations (Advanced)

A migration doesn't have to change database structure: you could use a migration to insert, update, or delete rows in a table. For instance, you could use migrations to assign [custom permissions](permissions.md) to groups other than Admin, or provide some initial data for a custom Eloquent model. Since you have access to the [Eloquent Schema Builder](https://laravel.com/docs/6.x/migrations#creating-tables), anything is possible (although of course, you should be extremely cautious and test your extension extensively).

Data migrations are the recommended way to specify default settings and permissions.

## Backend Models

With all your snazzy new database tables and columns, you're going to want a way to access the data in both the backend and the frontend. On the backend it's pretty straightforward – you just need to be familiar with [Eloquent](https://laravel.com/docs/6.x/eloquent).

### Adding New Models

If you've added a new table, you'll need to set up a new model for it. Rather than extending the Eloquent `Model` class directly, you should extend `Flarum\Database\AbstractModel` which provides a bit of extra functionality to allow your models to be extended by other extensions.

<!--
### Extending Models

If you've added columns to existing tables, they will be accessible on existing models. For example, you can grab data from the `users` table via the `Flarum\User\User` model.

If you need to define any attribute [accessors](https://laravel.com/docs/6.x/eloquent-mutators#defining-an-accessor), [mutators](https://laravel.com/docs/6.x/eloquent-mutators#defining-a-mutator), [dates](https://laravel.com/docs/6.x/eloquent-mutators#date-mutators), [casts](https://laravel.com/docs/6.x/eloquent-mutators#attribute-casting), or [default values](https://laravel.com/docs/6.x/eloquent#default-attribute-values) on an existing model, you can use the `Model` extender:

```php
use Flarum\Extend;
use Flarum\User\User;

return [
    new Extend\Model(User::class)
        ->defaultValue('is_alive', true)
        ->accessor('first_name', function ($value) {
            return ucfirst($value)
        })
        ->mutator('first_name', function ($value) {
            return strtolower($value);
        })
        ->date('suspended_until')
        ->cast('is_admin', 'boolean')
];
```
-->

### Relationships

You can also add [relationships](https://laravel.com/docs/6.x/eloquent-relationships) to existing models using the `hasOne`, `belongsTo`, `hasMany`,  `belongsToMany`and `relationship` methods on the `Model` extender. The first argument is the relationship name; the rest of the arguments are passed into the equivalent method on the model, so you can specify the related model name and optionally override table and key names:

```php
    new Extend\Model(User::class)
        ->hasOne('phone', 'App\Phone', 'foreign_key', 'local_key')
        ->belongsTo('country', 'App\Country', 'foreign_key', 'other_key')
        ->hasMany('comment', 'App\Comment', 'foreign_key', 'local_key')
        ->belongsToMany('role', 'App\Role', 'role_user', 'user_id', 'role_id')
```

Those 4 should cover the majority of relations, but sometimes, finer-grained customization is needed (e.g. `morphMany`, `morphToMany`, and `morphedByMany`). ANY valid Eloquent relationship is supported by the `relationship` method:

```php
    new Extend\Model(User::class)
        ->relationship('mobile', 'App\Phone', function ($user) {
            // Return any Eloquent relationship here.
            return $user->belongsToMany(Discussion::class, 'recipients')
                ->withTimestamps()
                ->wherePivot('removed_at', null);
        })
```


## Serializers

The next step is to expose your new data in Flarum's JSON:API so that it can be consumed by the frontend. You should become familiar with the [JSON:API specification](https://jsonapi.org/format/). Flarum's JSON:API layer is powered by the [tobscure/json-api](https://github.com/tobscure/json-api) library.

JSON:API resources are defined by **serializers**. To define a new resource type, create a new serializer class extending `Flarum\Api\Serializer\AbstractSerializer`. You must specify a resource `$type` and implement the `getDefaultAttributes` method which accepts the model instance as its only argument:

```php
use Flarum\Api\Serializer\AbstractSerializer;
use Flarum\Api\Serializer\UserSerializer;

class DiscussionSerializer extends AbstractSerializer
{
    protected $type = 'discussions';

    protected function getDefaultAttributes($discussion)
    {
        return [
            'title' => $discussion->title,
        ];
    }
}
```

### Attributes and Relationships

You can also specify relationships for your resource. Simply create a new method with the same name as the relation on your model, and return a call to `hasOne` or `hasMany` depending on the nature of the relationship. You must pass in the model instance and the name of the serializer to use for the related resources.

```php
    protected function user($discussion)
    {
        return $this->hasOne($discussion, UserSerializer::class);
    }
```

To add **attributes** and **relationships** to an existing resource type, use the `ApiSerializer` extender:

```php
use Flarum\Api\Serializer\UserSerializer;

return [
    (new Extend\ApiSerializer(UserSerializer::class))
        // One attribute at a time
        ->attribute('firstName', function ($serializer, $user, $attributes) {
                return $user->first_name
        })
        // Multiple modifications at once, more complex logic
        ->mutate(function($serializer, $user, $attributes) {
            $attributes['someAttribute'] = $user->someAttribute;
            if ($serializer->getActor()->can('administrate')) {
                $attributes['someDate'] = $serializer->formatDate($user->some_date);
            }

            return $attributes;
        })
        // API relationships
        ->hasOne('phone', PhoneSerializer::class)
        ->hasMany('comments', CommentSerializer::class),
]
```

## API Endpoints

Once you have defined your resources in serializers, you will need to expose them as API endpoints by adding routes and controllers.

Following JSON-API conventions, you can add five standard routes for your resource type using the `Routes` extender:

```php
    (new Extend\Routes('api'))
        ->get('/tags', 'tags.index', ListTagsController::class)
        ->get('/tags/{id}', 'tags.show', ShowTagController::class)
        ->post('/tags', 'tags.create', CreateTagController::class)
        ->patch('/tags/{id}', 'tags.update', UpdateTagController::class)
        ->delete('/tags/{id}', 'tags.delete', DeleteTagController::class)
```

The `Flarum\Api\Controller` namespace contains a number of abstract controller classes that you can extend to easily implement your JSON-API resources.

### Listing Resources

For the controller that lists your resource, extend the `Flarum\Api\Controller\AbstractListController` class. At a minimum, you need to specify the `$serializer` you want to use to serialize your models, and implement a `data` method to return a collection of models. The `data` method accepts the `Request` object and the tobscure/json-api `Document`.

```php
use Flarum\Api\Controller\AbstractListController;
use Psr\Http\Message\ServerRequestInterface as Request;
use Tobscure\JsonApi\Document;

class ListTagsController extends AbstractListController
{
    public $serializer = TagSerializer::class;
    
    protected function data(Request $request, Document $document)
    {
        return Tag::all();
    }
}
```

### Showing a Resource

For the controller that shows a single resource, extend the `Flarum\Api\Controller\AbstractShowController` class. Like for the list controller, you need to specify the `$serializer` you want to use to serialize your models, and implement a `data` method to return a single model:

```php
use Flarum\Api\Controller\AbstractShowController;
use Illuminate\Support\Arr;
use Psr\Http\Message\ServerRequestInterface as Request;
use Tobscure\JsonApi\Document;

class ShowTagController extends AbstractShowController
{
    public $serializer = TagSerializer::class;
    
    protected function data(Request $request, Document $document)
    {
        $id = Arr::get($request->getQueryParams(), 'id');
        
        return Tag::findOrFail($id);
    }
}
```

### Creating a Resource

For the controller that creates a resource, extend the `Flarum\Api\Controller\AbstractCreateController` class. This is the same as the show controller, except the response status code will automatically be set to `201 Created`. You can access the incoming JSON:API document body via `$request->getParsedBody()`:

```php
use Flarum\Api\Controller\AbstractCreateController;
use Illuminate\Support\Arr;
use Psr\Http\Message\ServerRequestInterface as Request;
use Tobscure\JsonApi\Document;

class CreateTagController extends AbstractCreateController
{
    public $serializer = TagSerializer::class;
    
    protected function data(Request $request, Document $document)
    {
        $attributes = Arr::get($request->getParsedBody(), 'data.attributes');
        
        return Tag::create([
            'name' => Arr::get($attributes, 'name')
        ]);
    }
}
```

### Updating a Resource

For the controller that updates a resource, extend the `Flarum\Api\Controller\AbstractShowController` class. Like for the create controller, you can access the incoming JSON:API document body via `$request->getParsedBody()`.

### Deleting a Resource

For the controller that deletes a resource, extend the `Flarum\Api\Controller\AbstractDeleteController` class. You only need to implement a `delete` method which enacts the deletion. The controller will automatically return an empty `204 No Content` response.

```php
use Flarum\Api\Controller\AbstractDeleteController;
use Illuminate\Support\Arr;
use Psr\Http\Message\ServerRequestInterface as Request;

class DeleteTagController extends AbstractDeleteController
{    
    protected function delete(Request $request)
    {
        $id = Arr::get($request->getQueryParams(), 'id');
        
        Tag::findOrFail($id)->delete();
    }
}
```

### Including Relationships

To include relationships when **listing**, **showing**, or **creating** your resource, specify them in the `$include` and `$optionalInclude` properties on your controller:

```php
    // The relationships that are included by default.
    public $include = ['user'];
    
    // Other relationships that are available to be included.
    public $optionalInclude = ['discussions'];
```

You can then get a list of included relationships using the `extractInclude` method. This can be used to eager-load the relationships on your models before they are serialized:

```php
$relations = $this->extractInclude($request);

return Tag::all()->load($relations);
```

### Pagination

You can allow the number of resources being **listed** to be customized by specifying the `limit` and `maxLimit` properties on your controller:

```php
    // The number of records included by default.
    public $limit = 20;
    
    // The maximum number of records that can be requested.
    public $maxLimit = 50;
```

You can then extract pagination information from the request using the `extractLimit` and `extractOffset` methods:

```php
$limit = $this->extractLimit($request);
$offset = $this->extractOffset($request);

return Tag::skip($offset)->take($limit);
```

To add pagination links to the JSON:API document, use the [`Document::addPaginationLinks` method](https://github.com/tobscure/json-api#meta--links).

### Sorting

You can allow the sort order of resources being **listed** to be customized by specifying the `sort` and `sortField` properties on your controller:

```php
    // The default sort field and order to use.
    public $sort = ['name' => 'asc'];
    
    // The fields that are available to be sorted by.
    public $sortFields = ['firstName', 'lastName'];
```

You can then extract sorting information from the request using the `extractSort` method. This will return an array of sort criteria which you can apply to your query:

```php
$sort = $this->extractSort($request);
$query = Tag::query();

foreach ($sort as $field => $order) {
    $query->orderBy(snake_case($field), $order);
}

return $query->get();
```

### Extending API Controllers

It is possible to customize all of these options on _existing_ API controllers too via the `ApiController` extender

```php
use Flarum\Api\Event\WillGetData;
use Flarum\Api\Controller\ListDiscussionsController;
use Illuminate\Contracts\Events\Dispatcher;

return [
    (new Extend\ApiController(ListDiscussionsController::class))
        ->setSerializer(MyDiscussionSerializer::class)
        ->addInclude('user')
        ->addOptionalInclude('posts')
        ->setLimit(20)
        ->setMaxLimit(50)
        ->setSort(['name' => 'asc'])
        ->addSortField('firstName')
        ->prepareDataQuery(function ($controller) {
            // Add custom logic here to modify the controller
            // before data queries are executed.
        })
]
```

The `ApiController` extendercan also be used to adjust data before serialization

```php
use Flarum\Api\Event\WillSerializeData;
use Flarum\Api\Controller\ListDiscussionsController;
use Illuminate\Contracts\Events\Dispatcher;

return [
    (new Extend\ApiController(ListDiscussionsController::class))
        ->prepareDataForSerialization(function ($controller, $data, $request, $document) {
            $data->load('myCustomRelation');
        }),
]
```

## Frontend Models

Now that you have exposed your data in Flarum's JSON:API, it's finally time to bring it to life and consume it on the frontend.

### Fetching Data

Flarum's frontend contains a local data `store` which provides an interface to interact with the JSON:API. You can retrieve resource(s) from the API using the `find` method, which always returns a promise:

```js
// GET /api/discussions?sort=createdAt
app.store.find('discussions', {sort: 'createdAt'}).then(console.log);

// GET /api/discussions/123
app.store.find('discussions', 123).then(console.log);
```

Once resources have been loaded, they will be cached in the store so you can access them again without hitting the API using the `all` and `getById` methods:

```js
const discussions = app.store.all('discussions');
const discussion = app.store.getById('discussions', 123);
```

The store wraps the raw API resource data in model objects which make it a bit easier to work with. Attributes and relationships can be accessed via pre-defined instance methods:

```js
const id = discussion.id();
const title = discussion.title();
const posts = discussion.posts(); // array of Post models
```

You can learn more about the store in our [API documentation](https://api.docs.flarum.org/js/master/class/src/common/store.js~store).

### Adding New Models

If you have added a new resource type, you will need to define a new model for it. Models must extend the `Model` class and re-define the resource attributes and relationships:

```js
import Model from 'flarum/common/Model';

export default class Tag extends Model {
  title = Model.attribute('title');
  createdAt = Model.attribute('createdAt', Model.transformDate);
  parent = Model.hasOne('parent');
  discussions = Model.hasMany('discussions');
}
```

You must then register your new model with the store:

```js
app.store.models.tags = Tag;
```

<!-- You must then register your new model with the store using the `Model` extender:

```js
export const extend = [
  new Extend.Model('tags', Tag)
];
``` -->

### Extending Models

To add attributes and relationships to existing models, modify the model class prototype:

```js
Discussion.prototype.user = Model.hasOne('user');
Discussion.prototype.posts = Model.hasMany('posts');
Discussion.prototype.slug = Model.attribute('slug');
```

<!-- To add attributes and relationships to existing models, use the `Model` extender:

```js
  new Extend.Model('discussions')
    .attribute('slug')
    .hasOne('user')
    .hasMany('posts')
``` -->

### Saving Resources

To send data back through the API, call the `save` method on a model instance. This method returns a Promise which resolves with the same model instance:

```js
discussion.save({ title: 'Hello, world!' }).then(console.log);
```

You can also save relationships by passing them in a `relationships` key. For has-one relationships, pass a single model instance. For has-many relationships, pass an array of model instances.

```js
user.save({
  relationships: {
    groups: [
      store.getById('groups', 1),
      store.getById('groups', 2)
    ]
  }
})
```

### Creating New Resources

To create a new resource, create a new model instance for the resource type using the store's `createRecord` method, then `save` it:

```js
const discussion = app.store.createRecord('discussions');

discussion.save({ title: 'Hello, world!' }).then(console.log);
```

### Deleting Resources

To delete a resource, call the `delete` method on a model instance. This method returns a Promise:

```js
discussion.delete().then(done);
```
