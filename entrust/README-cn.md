# Laravel Entrust中文说明

> GitHub：https://github.com/Zizaco/entrust

Entrust为我们在Laravel中实现基于角色的权限管理（RBAC）提供了简洁灵活的方式。

如果您正在寻找Laravel 4版本，请查看[Branch 1.0](https://github.com/Zizaco/entrust/tree/1.0)。它包含Laravel 4的最新版本。

安装
--

在Laravel 5中安装Entrust，只需将以下内容添加到composer.json即可。然后运行`composer update`：

```
"zizaco/entrust": "5.2.x-dev"
```

打开`config/app.php`并将以下内容添加到`providers`数组中：

```
ZizacoEntrustEntrustServiceProvider::class,
```

同样的在`config/app.php`添加以下内容`aliases`：

```
'Entrust'   => ZizacoEntrustEntrustFacade::class,
```

运行以下命令以发布程序包的配置文件`config/entrust.php`：

```
php artisan vendor:publish
```

打开`config/auth.php`并添加以下内容：

```
'providers' => [
    'users' => [
        'driver' => 'eloquent',
        'model' => NamespaceOfYourUserModelUser::class,
        'table' => 'users',
    ],
],
```

如果想要使用[中间件](https://wpfaq.cn/archives/docs/20510#middleware)（需要Laravel 5.1或更高版本），还需要添加以下内容：

```
'role' => izacoEntrustMiddlewareEntrustRole::class,
'permission' => izacoEntrustMiddlewareEntrustPermission::class,
'ability' => izacoEntrustMiddlewareEntrustAbility::class,
```

到`app/Http/Kernel.php`的`routeMiddleware`数组中。

配置
--

在配置文件`config/auth.php`中设置合适的值，Entrust会使用这些配置值来选择相应的用户表和模型类：

要进一步自定义表名和模型类命名空间，请编辑`config/entrust.php`。

### 用户角色权限表

现在生成Entrust迁移文件：

```
php artisan entrust:migration 
```

它将生成类似`<timestamp>_entrust_setup_tables.php`迁移文件。您现在可以使用artisan migrate命令运行它：

```
php artisan migrate 
```

执行迁移后，将生产四个新表：

*   `roles` – 存储角色
*   `permissions` – 存储权限
*   `role_user`– 存储角色和用户之间[的多对多](http://laravel.com/docs/4.2/eloquent#many-to-many)关系
*   `permission_role`– 存储角色和权限之间[的多对多](http://laravel.com/docs/4.2/eloquent#many-to-many)关系

### 模型类

#### 角色

在`app/models/Role.php`里面创建Role模型，并使用以下示例：

```
<?php namespace App;

use ZizacoEntrustEntrustRole;

class Role extends EntrustRole
{
} 
```

该`Role`模型有三个主要属性：

*   `name` – 角色的唯一名称，用于在应用程序层中查找角色信息。例如：“admin”，“owner”，“employee”。
*   `display_name` – 人类可读的角色名称。非唯一的且是可选的。例如：“用户管理员”，“项目所有者”，“Widget Co.员工”。
*   `description` – 更详细地说明角色的作用。也是可选的。

`display_name`和`description`是可选的; 他们的字段在数据库中可以为空。

#### 权限

在`app/models/Permission.php`创建Permission模型，并使用以下示例里面：

```
<?php namespace App;

use ZizacoEntrustEntrustPermission;

class Permission extends EntrustPermission
{
} 
```

该`Permission`模型具有与`Role`模型相同的三个属性：

*   `name` – 权限的唯一名称，用于在应用程序层中查找权限信息。例如：“create-post”，“edit-user”，“post-payment”，“mailing-list-subscribe”。
*   `display_name` – 人类可读的权限名称。非唯一的且是可选的。例如“创建帖子”，“编辑用户”，“邮寄付款”，“订阅邮件列表”。
*   `description` – 对权限的更详细说明。

一般来说，可以使用后两个属性来组成句子帮助说明权限，比如：“该权限`display_name`允许用户`description`。”

#### 用户

接下来，在现有`User`模型中使用`EntrustUserTrait` trait。例如：

```
<?php

use ZizacoEntrustTraitsEntrustUserTrait;

class User extends Eloquent
{
    use EntrustUserTrait; //添加EntrustUserTrait到User模型 

    ...
} 
```

这将与`Role`模型建立关系，并`User`模型添加`roles()`，`hasRole($name)`，`withRole($name)`，`can($permission)`，和`ability($roles, $permissions, $options)`方法。

使用composer autoload更新自动加载

```
composer dump-autoload 
```

#### 软删除

使用Entrust提供的迁移命令生成的关联关系表中默认使用了`onDelete('cascade')`以便父级记录被删除后移除其对应的关联关系。如果你由于某种原因不能在数据库中使用级联删除，那么可以在`EntrustRole`、`EntrustPermission`类以及`HasRole` trait提供的事件监听器中手动删除关联表中的记录。如果模型使用了软删除，那么当不小心误删除数据时，事件监听器将不会删除关联表数据。不过，由于Laravel事件监听器的局限性，所以暂时无法区分是调用`delete()`还是`forceDelete()`，基于这个原因，在你删除一个模型之前，必须手动删除所有关联数据（除非你的数据表使用了级联删除）：

```
$role = Role::findOrFail(1); // 获取给定权限

// 正常删除
$role->delete(); // This will work no matter what

// 强制删除
$role->users()->sync([]); // 删除关联数据
$role->perms()->sync([]); // 删除关联数据

$role->forceDelete(); // 不管透视表是否有级联删除都会生效
```

使用
--

### 创建角色/权限

让我们从创建`Role`s和`Permission`s开始：

```
$owner = new Role();
$owner->name         = 'owner';
$owner->display_name = 'Project Owner'; // 可选
$owner->description  = 'User is the owner of a given project'; // 可选
$owner->save();

$admin = new Role();
$admin->name         = 'admin';
$admin->display_name = 'User Administrator'; // 可选
$admin->description  = 'User is allowed to manage and edit other users'; // 可选
$admin->save(); 
```

创建两个角色后，接下来让我们将它们分配给用户。由于有`HasRole` trait，这将很简单：

```
$user = User::where('username', '=', 'michele')->first();

// 调用EntrustUserTrait提供的attachRole方法
$user->attachRole($admin); // 参数可以是Role对象，数组或id

// 或者也可以使用Eloquent原生的方法
$user->roles()->attach($admin->id); // 只需传递id即可 
```

现在我们只需要为这些角色添加权限：

```
$createPost = new Permission();
$createPost->name         = 'create-post';
$createPost->display_name = 'Create Posts'; // 可选
// 允许用户...
$createPost->description  = 'create new blog posts'; // 可选
$createPost->save();

$editUser = new Permission();
$editUser->name         = 'edit-user';
$editUser->display_name = 'Edit Users'; // 可选
// 允许用户...
$editUser->description  = 'edit existing users'; // 可选
$editUser->save();

$admin->attachPermission($createPost);
// 等价于  $admin->perms()->sync(array($createPost->id));

$owner->attachPermissions(array($createPost, $editUser));
// 等价于  $owner->perms()->sync(array($createPost->id, $editUser->id)); 
```

#### 检查角色&权限

现在我们可以通过以下方式检查角色和权限：

```
$user->hasRole('owner');   // false
$user->hasRole('admin');   // true
$user->can('edit-user');   // false
$user->can('create-post'); // true 
```

`hasRole()`并`can()`都可以接收数组形式的角色和权限进行检查：

```
$user->hasRole(['owner', 'admin']);       // true
$user->can(['edit-user', 'create-post']); // true 
```

默认情况下，如果用户存在任何一个角色或权限，该方法都将返回true。如果需要检查**所有**权限和角色，可以把true作为第二个参数传递相应方法：

```
$user->hasRole(['owner', 'admin']);             // true
$user->hasRole(['owner', 'admin'], true);       // false, user does not have admin role
$user->can(['edit-user', 'create-post']);       // true
$user->can(['edit-user', 'create-post'], true); // false, user does not have edit-user permission 
```

`User`可以拥有任意数量的`Role`s，反之亦然。

`Entrust`类提供了快捷方法`can()`和`hasRole()`来检查当前登录用户是否拥有指定角色和权限：

```
Entrust::hasRole('role-name');
Entrust::can('permission-name');

// 等价于

Auth::user()->hasRole('role-name');
Auth::user()->can('permission-name'); 
```

还可以通过通配符的方式来检查用户权限：

```
// 匹配admin的任何权限
$user->can("admin.*"); // true

// 匹配任何相关的users权限
$user->can("*_users"); // true 
```

要根据特定角色过滤用户，可以使用withRole() 查询作用域，例如检索所有管理员：

```
$admins = User::withRole('admin')->get();
// 也可以是关联关系
$company->users()->withRole('admin')->get(); 
```

#### ability方法

更高级的检查可以使用更好的`ability`方法。它包含三个参数（角色，权限，选项）：

*   `roles` 是一组要检查的角色。
*   `permissions` 是一组要检查的权限。

角色或权限变量可以是逗号分隔的字符串也可以是数组：

```
$user->ability(array('admin', 'owner'), array('create-post', 'edit-user'));

// 或者

$user->ability('admin,owner', 'create-post,edit-user'); 
```

这将会检查用户是否拥有任意提供的角色和权限。在本例中将返回true，因为该用户属于`admin`角色并拥有`create-post`权限。

第三个参数是一个选项数组：

```
$options = array(
    'validate_all' => true | false (Default: false),
    'return_type'  => boolean | array | both (Default: boolean)
); 
```

*   `validate_all` 是一个布尔值，用于设置是否检查所有值，或者至少匹配一个角色或权限则返回true。
*   `return_type` 用于指定返回布尔值、检查值数组还是两者都有。

这是一个示例输出：

```
$options = array(
    'validate_all' => true,
    'return_type' => 'both'
);

list($validate, $allValidations) = $user->ability(
    array('admin', 'owner'),
    array('create-post', 'edit-user'),
    $options
);

var_dump($validate);
// bool(false)

var_dump($allValidations);
// array(4) {
//     ['role'] => bool(true)
//     ['role_2'] => bool(false)
//     ['create-post'] => bool(true)
//     ['edit-user'] => bool(false)
// } 
```

`Entrust`类有一个快捷方法`ability()`来检查当前登录用户：

```
Entrust::ability('admin,owner', 'create-post,edit-user');

// 等价于

Auth::user()->ability('admin,owner', 'create-post,edit-user'); 
```

### Blade模板

可以在Blade模板中使用上述对应的三个方法。直接把指令参数传递给相应的`Entrust`函数。

```
@role('admin')
    <p>This is visible to users with the admin role. Gets translated to 
    Entrust::role('admin')</p>
@endrole

@permission('manage-admins')
    <p>This is visible to users with the given permissions. Gets translated to 
    Entrust::can('manage-admins'). The @can directive is already taken by core 
    laravel authorization package, hence the @permission directive instead.</p>
@endpermission

@ability('admin,owner', 'create-post,edit-user')
    <p>This is visible to users with the given abilities. Gets translated to 
    Entrust::ability('admin,owner', 'create-post,edit-user')</p>
@endability 
```

### 中间件

可以使用中间件按权限或角色过滤路由和路由组

```
Route::group(['prefix' => 'admin', 'middleware' => ['role:admin']], function() {
    Route::get('/', 'AdminController@welcome');
    Route::get('/manage', ['middleware' => ['permission:manage-admins'], 'uses' => 'AdminController@manageAdmins']);
}); 
```

可以使用管道符号作为_OR_逻辑运算符：

```
'middleware' => ['role:admin|root'] 
```

要实现_AND_逻辑功能，只需使用多个中间件实例

```
'middleware' => ['role:owner', 'role:writer'] 
```

对于更复杂的情况，可以使用`ability`中间件接受3个参数：roles，permissions，validate\_all

```
'middleware' => ['ability:admin|owner,create-post|edit-user,true'] 
```

### 路由过滤器的快捷方式

要按权限或角色过滤路由，可以在`app/Http/routes.php`中调用以下内容：

```
// 只有拥有'create-post'权限对应角色的用户才能访问admin/post*
Entrust::routeNeedsPermission('admin/post*', 'create-post');

// 只有所有者才能访问admin/advanced*
Entrust::routeNeedsRole('admin/advanced*', 'owner');

// 第二个参数可以是数组，这样用户必须满足所有条件才能访问对应路由
Entrust::routeNeedsPermission('admin/post*', array('create-post', 'edit-comment'));
Entrust::routeNeedsRole('admin/advanced*', array('owner','writer')); 
```

这两种方法都可以接受第三个参数。如果第三个参数为null，则返回禁止访问默认调用`App::abort(403)`，否则返回第三个参数。所以这样可以使用它：

```
Entrust::routeNeedsRole('admin/advanced*', 'owner', Redirect::to('/home')); 
```

此外，这两种方法还接受第四个参数。它默认为true并检查给定的所有角色/权限。如果将其设置为false，则只有当该用户的所有角色/权限都不满足时，该方法才会失败。对于要管理允许多个组访问的应用程序很有用。

```
// 如果用户拥有'create-post'、'edit-comment'其中之一或者两个权限都具备则可以访问
Entrust::routeNeedsPermission('admin/post*', array('create-post', 'edit-comment'), null, false);

// 如果用户是所有者、作者或者都具备则可以访问
Entrust::routeNeedsRole('admin/advanced*', array('owner','writer'), null, false);

// 如果用户是‘owner’，‘writer’或两者的成员，或者用户具有‘create-post’，‘edit-comment’，他们将有权访问
// 如果第4个参数为true，那么用户必须是角色的成员，必须拥有相应的权限 
Entrust::routeNeedsRoleOrPermission(
    'admin/advanced*',
    array('owner', 'writer'),
    array('create-post', 'edit-comment'),
    null,
    false
); 
```

### 路由过滤器

只需使用Facade中的`can`和`hasRole`方法，就可以在过滤器中使用Entrust角色/权限：

```
Route::filter('manage_posts', function()
{
    // check the current user
    if (!Entrust::can('create-post')) {
        return Redirect::to('admin');
    }
});

//只有用户对应角色拥有 'manage_posts' 权限才能访问任意 admin/post 路由
Route::when('admin/post*', 'manage_posts'); 
```

使用过滤器检查角色：

```
Route::filter('owner_role', function()
{
    // check the current user
    if (!Entrust::hasRole('Owner')) {
        App::abort(403);
    }
});

// 只有所有者才能访问 admin/advanced* 路由
Route::when('admin/advanced*', 'owner_role'); 
```

可以看到`Entrust::hasRole()`和`Entrust::can()`检查用户是否已登录，然后他/她是否具有该角色或权限。如果用户未登录，则返回也将是`false`。

实例
--

### 角色的增删改查

创建

```
//关联新角色权限关系
Role::create(request()->all());
if (request()->permission) {
    Role::attachPermissions(request()->permission);
} 
```

修改

```
//更新角色权限关系
Role::update(request()->all());
if (request()->permission) {
    Role::savePermissions(request()->permission);
} 
```

删除

```
//由于存在外键约束，会自动同步删除角色权限关系
Role::destroy(request()->id); 
```

查询

```
//查询角色拥有的权限
Role::with('perms')->find(request()->id); 
```

### 用户的增删改查

创建

```
//关联角色
User::create(request()->all());
if (request()->role) {
    User::roles()->sync(request()->role);
} 
```

修改

```
//更新角色关系
User::update(request()->all());
if (request()->role) {
    User::roles()->sync($request->role);
} 
```

删除

```
//由于存在外键约束，会自动同步删除用户角色关系
User::destroy(request()->id); 
```

查询

```
//查询用户拥有的角色和权限
User::with('roles.perms')->find(request()->id); 
```

以上是参考，请根据具体业务逻辑发挥

FAQ
---

如果在执行迁移时遇到错误，如下所示：

```
SQLSTATE[HY000]: General error: 1005 Can't create table 'laravelbootstrapstarter.#sql-42c_f8' (errno: 150)
    (SQL: alter table `role_user` add constraint role_user_user_id_foreign foreign key (`user_id`)
    references `users` (`id`)) (Bindings: array ()) 
```

很可能是user表中的`id`列不匹配`role_user`表`user_id`列。确保两者都是`INT(10)`。

当尝试使用EntrustUserTrait方法时，您可能会遇到类似的错误

```
Class name must be a valid object or a string
```

那么很可能没有发布过Entrust资源或做了错误的操作。首先检查`config`目录中是否有`entrust.php`该文件。如果没有，请尝试执行`php artisan vendor:publish`，如果依然没有，请手动复制`/vendor/zizaco/entrust/src/config/config.php`到config目录中并重命名`entrust.php`。

如果使用自定义命名空间，那么需要修改`permission`和`role`模型的位置，可以通过编辑配置文件`config/entrust.php`来执行此操作

```
'role' => 'CustomNamespaceRole' 
'permission' => 'CustomNamespacepermission'
```