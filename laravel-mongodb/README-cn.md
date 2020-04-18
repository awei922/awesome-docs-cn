# Laravel MongoDB 中文文档

> GitHub：https://github.com/jenssegers/laravel-mongodb

一个支持MongoDB的Eloquent模型和Query构建器，使用原始的Laravel API。*该库扩展了原始的Laravel类，因此它使用完全相同的方法。*

安装（Installation）
----------------

确保安装了MongoDB PHP驱动程序。您可以在[http://php.net/manual/en/mongodb.installation.php上](http://php.net/manual/en/mongodb.installation.php)找到安装说明

**警告**：版本> = 3.0时不再支持旧的mongo PHP驱动程序。

使用composer安装：

```
composer require jenssegers/mongodb
```

### Laravel版兼容性
 Laravel  | Package
:---------|:----------
 4.2.x    | 2.0.x
 5.0.x    | 2.1.x
 5.1.x    | 2.2.x or 3.0.x
 5.2.x    | 2.3.x or 3.0.x
 5.3.x    | 3.1.x or 3.2.x
 5.4.x    | 3.2.x
 5.5.x    | 3.3.x
 5.6.x    | 3.4.x
 5.7.x    | 3.4.x
 5.8.x    | 3.5.x
 6.x      | 3.6.x

并在`config/app.php`添加服务提供商：

```
JenssegersMongodbMongodbServiceProvider::class,
```

与[Lumen一起](http://lumen.laravel.com/)使用，请在`bootstrap/app.php`添加服务提供商。在此文件中，您还需要启用Eloquent。不过，你必须确保你调用的`$app->withEloquent();`是在您注册MongodbServiceProvider的位置**下面**：

```
$app->register(JenssegersMongodbMongodbServiceProvider::class);

$app->withEloquent();
```

服务提供者将使用原始数据库管理器注册mongodb数据库扩展。无需注册其他facades或objects。使用mongodb连接时，Laravel会自动为您提供相应的mongodb对象。

在Laravel框架外使用，请查看[Capsule管理器](https://github.com/illuminate/database/blob/master/README.md)并添加：

```
$capsule->getDatabaseManager()->extend('mongodb', function($config)
{
    return new JenssegersMongodbConnection($config);
});
```

升级（Upgrading）
-------------

#### 从版本2升级到3

在支持新MangoDB PHP扩展的新的主要版本中，我们还移动了Model类的位置，并用一个Trait替换了MySQL模型类。

请在顶级模型文件或注册别名的所有引用`JenssegersMongodbModel` 改成`JenssegersMongodbEloquentModel`。

```
use JenssegersMongodbEloquentModel as Eloquent;

class User extends Eloquent {}
```

如果您正在使用混合关系，那么您的MySQL类现在要做的应该是扩展原始的Eloquent模型类`IlluminateDatabaseEloquentModel`而不是删除`JenssegersEloquentModel`。可以使用新的`JenssegersMongodbEloquentHybridRelations` trait来替代。这样使事情更清楚，因为只有一个单一的模型类。

```
use JenssegersMongodbEloquentHybridRelations;

class User extends Eloquent {

    use HybridRelations;

    protected $connection = 'mysql';

}
```

嵌套关系后，现在返回的是`IlluminateDatabaseEloquentCollection`而不是自定义Collection类。如果您使用的是一种可用的特殊方法，请将它们转换为Collection操作。

```
$books = $user->books()->sortBy('title');
```

测试（Testing）
-----------

运行此程序包的测试，请运行：

```
docker-compose up
```

配置（Configuration）
-----------------

在`config/database.php`更改您的默认数据库连接名称：

```
'default' => env('DB_CONNECTION', 'mongodb'),
```

添加一个新的mongodb连接：

```
'mongodb' => [
    'driver'   => 'mongodb',
    'host'     => env('DB_HOST', 'localhost'),
    'port'     => env('DB_PORT', 27017),
    'database' => env('DB_DATABASE'),
    'username' => env('DB_USERNAME'),
    'password' => env('DB_PASSWORD'),
    'options'  => [
        'database' => 'admin' // sets the authentication database required by mongo 3
    ]
],
```

您可以使用以下配置连接到多个服务器或集群：

```
'mongodb' => [
    'driver'   => 'mongodb',
    'host'     => ['server1', 'server2'],
    'port'     => env('DB_PORT', 27017),
    'database' => env('DB_DATABASE'),
    'username' => env('DB_USERNAME'),
    'password' => env('DB_PASSWORD'),
    'options'  => [
        'replicaSet' => 'replicaSetName'
    ]
],
```

或者，您可以使用MongoDB连接字符串：

```
'mongodb' => [
    'driver'   => 'mongodb',
    'dsn' => env('DB_DSN'),
    'database' => env('DB_DATABASE'),
],
```

有关其URI格式，请参阅MongoDB官方文档：[https](https://docs.mongodb.com/manual/reference/connection-string/)：[//docs.mongodb.com/manual/reference/connection-string/](https://docs.mongodb.com/manual/reference/connection-string/)

Eloquent
--------

该扩展包包括了一个启用MongoDB的Eloquent类，您可以使用它来定义相应集合的模型。

```
use JenssegersMongodbEloquentModel as Eloquent;

class User extends Eloquent {}
```

请注意，我们没有告诉Eloquent使用哪个集合用于`User`模型。就像原始的Eloquent一样，除非明确指定其他名称，否则该类的小写复数名称将用作集合名称。您可以通过`collection`在模型上定义属性来指定自定义集合（表的别名）：

```
use JenssegersMongodbEloquentModel as Eloquent;

class User extends Eloquent {

    protected $collection = 'users_collection';

}
```

**注意：** Eloquent还假定每个集合都有一个名为id的主键列。您可以定义一个`primaryKey`属性来覆盖此约定。同样地，您可以定义一个`connection`属性来覆盖使用模型时应该使用的数据库连接的名称。

```
use JenssegersMongodbEloquentModel as Eloquent;

class MyModel extends Eloquent {

    protected $connection = 'mongodb';

}
```

其他所有（应该）都像原始的Eloquent模型一样工作。在[http://laravel.com/docs/eloquent](http://laravel.com/docs/eloquent)上阅读有关Eloquent的更多信息

可选：别名（Optional: Alias）
----------------------

您还可以通过将以下内容添加到`config/app.php`别名数组中来注册MongoDB模型的别名：

```
'Moloquent'       => JenssegersMongodbEloquentModel::class,
```

这将允许您使用注册的别名，如：

```
class MyModel extends Moloquent {}
```

查询生成器（Query Builder）
--------------------

数据库驱动程序直接插入原始查询构建器。 使用mongodb连接时，您将能够构建流畅的查询来执行数据库操作。 为方便起见，提供了一个表的集合别名以及一些额外的mongodb特定运算符/操作。

```
$users = DB::collection('users')->get();

$user = DB::collection('users')->where('name', 'John')->first();
```

如果您未更改默认数据库连接，则需要在查询时指定它。

```
$user = DB::connection('mongodb')->collection('users')->get();
```

在[http://laravel.com/docs/queries](http://laravel.com/docs/queries)上阅读有关查询构建器的更多信息

架构（Schema）
----------

数据库驱动程序还具有（有限的）schema构建器支持。您可以轻松地操作集合并设置索引：

```
Schema::create('users', function($collection)
{
    $collection->index('name');

    $collection->unique('email');
});
```

支持的操作有：

*   create和drop
*   collection
*   hasCollection
*   index和dropIndex (也支持复合索引)
*   unique
*   background, sparse, expire, geospatial (MongoDB特定)

所有其他（不支持的）操作都是通过虚拟传递方法实现的，因为MongoDB不使用预定义的模式。在[http://laravel.com/docs/schema](http://laravel.com/docs/schema)上阅读有关架构构建器的更多信息

### 地理空间索引（geospatial ）

地理空间索引可用于查询基于位置的文档。它们有两种形式：`2d`和`2dsphere`。使用schema构建器将这些添加到集合中。

添加`2d`索引：

```
Schema::create('users', function($collection)
{
    $collection->geospatial('name', '2d');
});
```

添加`2dsphere`索引：

```
Schema::create('users', function($collection)
{
    $collection->geospatial('name', '2dsphere');
});
```

扩展（Extensions）
--------------

### 验证

如果您想使用Laravel的自带的Auth功能，请注册此包含的服务提供商：

```
'JenssegersMongodbAuthPasswordResetServiceProvider',
```

此服务提供程序将稍微修改内部DatabaseReminderRepository以添加对基于MongoDB的密码提醒的支持。如果您不使用密码提醒，则无需注册此服务提供商，其他一切都应该正常工作。

### 队列

如果要将MongoDB用作数据库后端，请在`config/queue.php`更改以下驱动程序：

```
'connections' => [
    'database' => [
        'driver' => 'mongodb',
        'table'  => 'jobs',
        'queue'  => 'default',
        'expire' => 60,
    ],
```

如果要使用MongoDB处理失败的作业，请在`config/queue.php`以下位置更改数据库：

```
'failed' => [
    'database' => 'mongodb',
    'table'    => 'failed_jobs',
    ],
```

并在`config/app.php`添加服务提供商：

```
JenssegersMongodbMongodbQueueServiceProvider::class,
```

### Sentry

如果您想将此库与[Sentry](https://cartalyst.com/manual/sentry)一起使用，请查看[https://github.com/jenssegers/Laravel-MongoDB-Sentry](https://github.com/jenssegers/Laravel-MongoDB-Sentry)

### Sessions

MongoDB会话驱动程序在单独的包中提供，请查看[https://github.com/jenssegers/Laravel-MongoDB-Session](https://github.com/jenssegers/Laravel-MongoDB-Session)

例子（Examples）
------------

### 基本用法

**检索所有模型**

```
$users = User::all();
```

**通过主键\_id检索记录**

```
$user = User::find('517c43667db388101e00000f');
```

**Wheres**

```
$users = User::where('votes', '>', 100)->take(10)->get();
```

**Or**

```
$users = User::where('votes', '>', 100)->orWhere('name', 'John')->get();
```

**And**

```
$users = User::where('votes', '>', 100)->where('name', '=', 'John')->get();
```

**WhereIn**

```
$users = User::whereIn('age', [16, 18, 20])->get();
```

使用`whereNotIn`时，如果字段不存在，将返回使用对象。可以结合`whereNotNull('age')`来过滤这些文件。

**Between**

```
$users = User::whereBetween('votes', [1, 100])->get();
```

**Null**

```
$users = User::whereNull('updated_at')->get();
```

**Order By**

```
$users = User::orderBy('name', 'desc')->get();
```

**Offset & Limit**

```
$users = User::skip(10)->take(5)->get();
```

**Distinct**

Distinct需要一个字段，返回不重复的值。

```
$users = User::distinct()->get(['name']);
// or
$users = User::distinct('name')->get();
```

可以与**where**结合使用：

```
$users = User::where('active', true)->distinct('name')->get();
```

**复杂的Wheres**

```
$users = User::where('name', '=', 'John')->orWhere(function($query)
    {
        $query->where('votes', '>', 100)
              ->where('title', '<>', 'Admin');
    })
    ->get();
```

**Group By**

未分组的选定列将与$last函数聚合在一起。

```
$users = Users::groupBy('title')->get(['title', 'name']);
```

**聚合(Aggregation)**

_聚合仅适用于大于2.2的MongoDB版本。_

```
$total = Order::count();
$price = Order::max('price');
$price = Order::min('price');
$price = Order::avg('price');
$total = Order::sum('price');
```

聚合可以结合**where**：

```
$sold = Orders::where('sold', true)->sum('price');
```

聚合也可以用于子文档：

```
$total = Order::max('suborder.price');
...
```

**注意**：此aggreagtion仅适用于单个子文档（如embedsOne）而非子文档数组（如embedsMany）

**Like**

```
$user = Comment::where('body', 'like', '%spam%')->get();
```

**递增或递减**

对指定的属性执行增量或减量（默认值1）：

```
User::where('name', 'John Doe')->increment('age');
User::where('name', 'Jaques')->decrement('weight', 50);
```

返回更新的对象数：

```
$count = User->increment('age');
```

您还可以指定要更新的其他列：

```
User::where('age', '29')->increment('age', 1, ['group' => 'thirty something']);
User::where('bmi', 30)->decrement('bmi', 1, ['category' => 'overweight']);
```

**软删除**

软删除模型时，实际上并未从数据库中删除它。而是在记录上设置deleted\_at时间戳。要为模型启用软删除，请将SoftDeletingTrait应用于模型：

```
use JenssegersMongodbEloquentSoftDeletes;

class User extends Eloquent {

    use SoftDeletes;

    protected $dates = ['deleted_at'];

}
```

有关更多信息，请访问[http://laravel.com/docs/eloquent#soft-deleting](http://laravel.com/docs/eloquent#soft-deleting)

### MongoDB特定的操作

**Exists**

匹配具有指定字段的文档。

```
User::where('age', 'exists', true)->get();
```

**All**

匹配包含查询中指定的所有元素的数组。

```
User::where('roles', 'all', ['moderator', 'author'])->get();
```

**Size**

如果数组字段是指定大小，则选择文档。

```
User::where('tags', 'size', 3)->get();
```

**正则表达式**

选择值与指定正则表达式匹配的文档。

```
User::where('name', 'regex', new MongoDBBSONRegex("/.*doe/i"))->get();
```

**注意：**您还可以使用Laravel regexp操作。这些更灵活，会自动将正则表达式字符串转换为 MongoDBBSONRegex 对象。

```
User::where('name', 'regexp', '/.*doe/i'))->get();
```

反之：

```
User::where('name', 'not regexp', '/.*doe/i'))->get();
```

**Type**

如果字段是指定类型，则选择文档。有关更多信息，请访问：[http](http://docs.mongodb.org/manual/reference/operator/query/type/#op._S_type)：[//docs.mongodb.org/manual/reference/operator/query/type/#op.\_S\_type](http://docs.mongodb.org/manual/reference/operator/query/type/#op._S_type)

```
User::where('age', 'type', 2)->get();
```

**Mod**

对字段的值执行模运算，并选择具有指定结果的文档。

```
User::where('age', 'mod', [10, 0])->get();
```

**Near**

**注意：**按此顺序指定坐标：`longitude, latitude`。

```
$users = User::where('location', 'near', [
    '$geometry' => [
        'type' => 'Point',
        'coordinates' => [
            -0.1367563,
            51.5100913,
        ],
    ],
    '$maxDistance' => 50,
]);
```

**GeoWithin**

```
$users = User::where('location', 'geoWithin', [
    '$geometry' => [
        'type' => 'Polygon',
        'coordinates' => [[
            [
                -0.1450383,
                51.5069158,
            ],       
            [
                -0.1367563,
                51.5100913,
            ],       
            [
                -0.1270247,
                51.5013233,
            ],  
            [
                -0.1450383,
                51.5069158,
            ],
        ]],
    ],
]);
```

**GeoIntersects**

```
$locations = Location::where('location', 'geoIntersects', [
    '$geometry' => [
        'type' => 'LineString',
        'coordinates' => [
            [
                -0.144044,
                51.515215,
            ],
            [
                -0.129545,
                51.507864,
            ],
        ],
    ],
]);
```

**Where**

匹配满足JavaScript表达式的文档。有关更多信息，请访问[http://docs.mongodb.org/manual/reference/operator/query/where/#op.\_S\_where](http://docs.mongodb.org/manual/reference/operator/query/where/#op._S_where)

### Inserts, updates and deletes

插入，更新和删除记录就像原始的Eloquent一样。

**保存新模型**

```
$user = new User;
$user->name = 'John';
$user->save();
```

您还可以使用简单一行create方法将新模型保存：

```
User::create(['name' => 'John']);
```

**更新模型**

要更新模型，您可以检索它，更改属性并使用save方法。

```
$user = User::first();
$user->email = 'john@foo.com';
$user->save();
```

_还有对upsert操作的支持，请查看[https://github.com/jenssegers/laravel-mongodb#mongodb-specific-operations](https://github.com/jenssegers/laravel-mongodb#mongodb-specific-operations)_

**删除模型**

要删除模型，只需在实例上调用delete方法：

```
$user = User::first();
$user->delete();
```

或者通过\_id值删除模型：

```
User::destroy('517c43667db388101e00000f');
```

有关模型操作的更多信息，请查看[http://laravel.com/docs/eloquent#insert-update-delete](http://laravel.com/docs/eloquent#insert-update-delete)

### 日期

Eloquent允许您使用Carbon/DateTime对象而不是MongoDate对象。在内部，这些日期在保存到数据库时将转换为MongoDate对象。如果您希望在非默认日期字段上使用此功能，则需要按照此处所述手动指定它们：[http](http://laravel.com/docs/eloquent#date-mutators)：[//laravel.com/docs/eloquent#date-mutators](http://laravel.com/docs/eloquent#date-mutators)

例：

```
use JenssegersMongodbEloquentModel as Eloquent;

class User extends Eloquent {

    protected $dates = ['birthday'];

}
```

这允许您执行以下查询：

```
$users = User::where('birthday', '>', new DateTime('-18 years'))->get();
```

### 关系

支持的关系有：

*   hasOne
*   hasMany
*   belongsTo
*   belongsToMany
*   embedsOne
*   embedsMany

例：

```
use JenssegersMongodbEloquentModel as Eloquent;

class User extends Eloquent {

    public function items()
    {
        return $this->hasMany('Item');
    }

}
```

和反比关系：

```
use JenssegersMongodbEloquentModel as Eloquent;

class Item extends Eloquent {

    public function user()
    {
        return $this->belongsTo('User');
    }

}
```

belongsToMany关系不会使用中间关系“table”（表），而是将id推送到related\_ids属性。这使belongsToMany方法的第二个参数无用。如果要为关系定义自定义键，请将其设置为`null`：

```
use JenssegersMongodbEloquentModel as Eloquent;

class User extends Eloquent {

    public function groups()
    {
        return $this->belongsToMany('Group', null, 'user_ids', 'group_ids');
    }

}
```

其他关系尚未得到支持，但可能会在未来添加。有关这些关系的更多信息，请访问[http://laravel.com/docs/eloquent#relationships](http://laravel.com/docs/eloquent#relationships)

### EmbedsMany Relations

如果要嵌入模型而不是引用模型，可以使用`embedsMany`关系。此关系类似于`hasMany`关系，但将模型嵌入父对象中。

**记住**：这些关系返回El​​oquent集合，它们不返回查询构建器对象！

```
use JenssegersMongodbEloquentModel as Eloquent;

class User extends Eloquent {

    public function books()
    {
        return $this->embedsMany('Book');
    }

}
```

您可以通过动态属性访问嵌入式模型：

```
$books = User::first()->books;
```

反向关系是自动_神奇_可用的，您不需要定义此反向关系。

```
$user = $book->user;
```

插入和更新嵌入式模型的工作方式类似于以下`hasMany`关系：

```
$book = new Book(['title' => 'A Game of Thrones']);

$user = User::first();

$book = $user->books()->save($book);
// or
$book = $user->books()->create(['title' => 'A Game of Thrones'])
```

您可以使用他们的`save`方法更新嵌入式模型（从2.0.0版开始提供）：

```
$book = $user->books()->first();

$book->title = 'A Game of Thrones';

$book->save();
```

您可以使用`destroy`关系上的`delete`方法或模型上的方法（从2.0.0版开始提供）来删除嵌入式模型：

```
$book = $user->books()->first();

$book->delete();
// or
$user->books()->destroy($book);
```

如果要在不触及数据库的情况下添加或删除嵌入式模型，可以使用`associate`和`dissociate`方法。要最终将更改写入数据库，请保存父对象：

```
$user->books()->associate($book);

$user->save();
```

与其他关系一样，embedsMany根据模型名称假定关系的本地键。您可以通过将第二个参数传递给embedsMany方法来覆盖默认本地键：

```
return $this->embedsMany('Book', 'local_key');
```

嵌入式关系将返回嵌入项的集合，而不是查询构建器。查看可用的操作：[https](https://laravel.com/docs/master/collections)：[//laravel.com/docs/master/collections](https://laravel.com/docs/master/collections)

### EmbedsOne Relations

embedsOne关系类似于EmbedsMany关系，但只嵌入单个模型。

```
use JenssegersMongodbEloquentModel as Eloquent;

class Book extends Eloquent {

    public function author()
    {
        return $this->embedsOne('Author');
    }

}
```

您可以通过动态属性访问嵌入式模型：

```
$author = Book::first()->author;
```

插入和更新嵌入式模型的工作方式类似于以下`hasOne`关系：

```
$author = new Author(['name' => 'John Doe']);

$book = Books::first();

$author = $book->author()->save($author);
// or
$author = $book->author()->create(['name' => 'John Doe']);
```

您可以使用该`save`方法更新嵌入式模型（从2.0.0版开始提供）：

```
$author = $book->author;

$author->name = 'Jane Doe';
$author->save();
```

您可以使用以下新模型替换嵌入式模型：

```
$newAuthor = new Author(['name' => 'Jane Doe']);
$book->author()->save($newAuthor);
```

### MySQL关系

如果你正在使用混合的MongoDB和SQL设置，那么你很幸运！模型将根据相关模型的类型自动返回MongoDB或SQL关系。当然，如果您希望此功能同时工作，那么您的SQL模型将需要使用该`JenssegersMongodbEloquentHybridRelations`特征。请注意，此功能仅适用于hasOne，hasMany和belongsTo关系。

示例基于SQL的用户模型：

```
use JenssegersMongodbEloquentHybridRelations;

class User extends Eloquent {

    use HybridRelations;

    protected $connection = 'mysql';

    public function messages()
    {
        return $this->hasMany('Message');
    }

}
```

而基于Mongodb的Message模型：

```
use JenssegersMongodbEloquentModel as Eloquent;

class Message extends Eloquent {

    protected $connection = 'mongodb';

    public function user()
    {
        return $this->belongsTo('User');
    }

}
```

### 原始表达

这些表达式将直接注入查询中。

```
User::whereRaw(['age' => array('$gt' => 30, '$lt' => 40)])->get();
```

您还可以在内部MongoCollection对象上执行原始表达式。如果在模型类上执行此操作，它将返回一组模型。如果在查询构建器上执行此操作，它将返回原始响应。

```
// Returns a collection of User models.
$models = User::raw(function($collection)
{
    return $collection->find();
});

// Returns the original MongoCursor.
$cursor = DB::collection('users')->raw(function($collection)
{
    return $collection->find();
});
```

可选：如果未将闭包传递给raw方法，则可以访问内部MongoCollection对象：

```
$model = User::raw()->findOne(['age' => array('$lt' => 18)]);
```

可以像这样访问内部MongoClient和MongoDB对象：

```
$client = DB::getMongoClient();
$db = DB::getMongoDB();
```

### MongoDB的具体操作

**光标超时（Cursor timeout）**

要防止MongoCursorTimeout异常，您可以手动设置将应用于游标的超时值：

```
DB::collection('users')->timeout(-1)->get();
```

**Upsert**

更新或插入文档。update方法的其他选项将直接传递给本机更新方法。

```
DB::collection('users')->where('name', 'John')
                       ->update($data, ['upsert' => true]);
```

**Projections**

您可以使用该`project`方法将投影应用于查询。

```
DB::collection('items')->project(['tags' => ['$slice' => 1]])->get();
DB::collection('items')->project(['tags' => ['$slice' => [3, 7]]])->get();
```

**分页（Pagination）**

```
$limit = 25;
$projections = ['id', 'name'];
DB::collection('items')->paginate($limit, $projections);
```

**Push**

将项添加到数组。

```
DB::collection('users')->where('name', 'John')->push('items', 'boots');
DB::collection('users')->where('name', 'John')->push('messages', ['from' => 'Jane Doe', 'message' => 'Hi John']);
```

如果您不想要重复项，请将第三个参数设置为`true`：

```
DB::collection('users')->where('name', 'John')->push('items', 'boots', true);
```

**Pull**

从数组中删除项目。

```
DB::collection('users')->where('name', 'John')->pull('items', 'boots');
DB::collection('users')->where('name', 'John')->pull('messages', ['from' => 'Jane Doe', 'message' => 'Hi John']);
```

**Unset**

从文档中删除一个或多个字段。

```
DB::collection('users')->where('name', 'John')->unset('note');
```

您还可以在模型上执行删除。

```
$user = User::where('name', 'John')->first();
$user->unset('note');
```

### 查询缓存(Query Caching)

您可以使用remember方法轻松缓存查询结果：

```
$users = User::remember(10)->get();
```

_来自：[http](http://laravel.com/docs/queries#caching-queries)：[//laravel.com/docs/queries#caching-queries](http://laravel.com/docs/queries#caching-queries)_

### 查询记录(Query Logging)

默认情况下，Laravel会记录已为当前请求运行的所有查询的内存。但是，在某些情况下，例如插入大量行时，这可能会导致应用程序使用过多的内存。要禁用日志，您可以使用以下`disableQueryLog`方法：

```
DB::connection()->disableQueryLog();
```

_来自：[http](http://laravel.com/docs/database#query-logging)：[//laravel.com/docs/database#query-logging](http://laravel.com/docs/database#query-logging)_