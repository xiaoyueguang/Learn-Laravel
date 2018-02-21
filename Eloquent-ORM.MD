# Eloquent ORM

> `Laravel`内置了一个`Eloquent-ORM`, 对数据库进行查询, 插入, 更新或删除等操作.  

一个传统的WEB框架基本都为MVC, 即模型, 视图, 控制器.
* `Model`模型是负责应用程序数据逻辑的部分
* `View`视图是应用中处理数据显示的部分
* `Controller`是应用程序中处理用户交互的部分.

`Laravel`的模型基于他自身的`Eloquent`, 描述这个模型与表的关系.

### 创建一个模型文件
可通过 `php artisan make:model User` 自动生成一个模型文件, 像这句会默认创建一个 `User.php`文件, 且数据表会指向对应的复数形式的表, 即 `Users`, 也可以手动指定类中的`$table`属性, 来指定对应的表.

`$primaryKey`可用来更改表的主键.

表中得有`updated_at`, `created_at`, 表示修改和创建时间, `Laravel`会自动更新数据, 可通过`$timestamps`属性来禁用.

### 查询
```php
// 获取所有记录的 集合
User::all();
// 获取主键为 1的单条记录. find获取不到时 不会有任何返回
User::find(1);
// 与find不同的时, 以下两个方法 获取不到数据会返回404
// 获取主键为 3的单条记录.
User::findOrFail(3);
// 获取主键大于3 的首个记录
User::where('id', '>', 3)->firstOrFail();
```

### 批量赋值
`Eloquent`允许我们对表进行批量赋值, 因为批量操作, 对表的修改影响很大, 因此需要两个属性来确定批量赋值属性, 即白名单(`fillable`)和黑名单(`guarded`).
```php
// 支持批量赋值的属性
protected $fillable = [];
// 阻止批量赋值的属性
protected $guarded = [];
```

### 新增
```php
$user = new User;
$user->name = 'test';
// 调用 save 后会新增一条记录
$user->save();
// 创建一个新用户
User::create(['name' => 'Test']);
// 先按照属性查找, 找不到就创建一条数据并返回实例
User::firstOrCreate(['name' => 'Test']);
// 先按照属性查找, 找不到就返回实例
User::firstOrNew(['name' => 'Test']);
```
### 更新
```php
// 先取出来
$user = User::find(1);
$user->name = 'test';
// 保存
$user->save();
```

### 删除
```php
$user = User::find(1);
// 取出来后删除
$user->delete();
// 按照主键删除
User::destroy(1);
User::destroy([1, 2, 3]);
User::destroy(1, 2, 3);
```

### 软删除
数据不会被真实删掉. 软删除后, 会设定默认的 deleted_at.
```php
use Illuminate\Database\Eloquent\SoftDeletes;

{
  use SoftDeletes;
  // 默认为 deleted_at
  protected $dates = ['deleted_at'];
}

// 查询被软删除的数据
User::withTrashed()
->withTrashed()

// 只查询被软删除的数据
::onlyTrashed();
// 恢复被软删除的数据
$user->restore();

// 强制性删除
$user->forceDelete();
```

### 范围查询
在模型里定义有`scope`前缀的方法, 定义查询的范围, 即可达到复用.

```php
// Model
// $id 为传入的值
public function scopeTest ($query, $id) {
  return $query->where('id', '>', $id);
}
// Controller
$user = User::test(1)->get();

```

## 关联
数据表和表之间可能会有关系, `Eloquent`能很好的管理这些关系.
* 一对一
* 一对多
* 多对多
* 远层一对多关联
* 多态关联
* 多态的多对多关联

下面以博客为例子, 来解释这些关系.

### 一对一
博客里发布的作者和手机是一比一, 一个作者只有一个手机号, 一个手机号 只属于一个作者.

users
|字段|说明|
|:--|:--|
|id|用户表主键|
|name|用户名|

phones
|字段|说明|
|:--|:--|
|id|手机表主键|
|phone|手机号码|
|user_id|用户ID|

```php
// User.php
public function phone () {
    // 参数依次为指定模型, 外键, 主键
    return $this->hasOne('App\Phone', 'user_id', 'id');
}

// Controller
User::find(2)->phone;
```
这里将会执行两条SQL
```php
select * from `users` where `users`.`id` = ? limit 1;
select * from `phones` where `phones`.`user_id` = ? and `phones`.`user_id` is not null limit 1
```
相对的, phone 也会属于user中的一个
```php
// Phone.php
public function user() {
    return $this->belongsTo('App\User', 'user_id', 'id');
}

Phone::find(2)->user;
```

### 一对多
一个人可能会发布很多文章. 这就属于 一(作者)对多(文章)的关系

users
|字段|说明|
|:--|:--|
|id|用户表主键|
|name|用户名|

articles
|字段|说明|
|:--|:--|
|id|主键|
|title|标题|
|content|内容|
|user_id|用户ID|

```php
// User.php
public function article () {
    // 参数依次为 管理的模型, 外键, 主键
    return $this->hasMany('App\Article', 'user_id', 'id');
}

// Controller
User::find(1)->article;
// 在上面的基础上再进行条件查询
User::find(1)->article()->where('id', '>', '4')->get();
```

### 多对多
一个用户, 可能有 管理员或客人两种身份.  
一个管理员身份, 也可能被多个用户所有用.  
这样就形成了多对多的关联, 需要三个表来实现这种需求.

users
|字段|说明|
|:--|:--|
|id|主键|
|name|用户名|

roles
|字段|说明|
|:--|:--|
|id|主键|
|name|权限名称|

role_user
|字段|说明|
|:--|:--|
|id|主键|
|user_id|用户ID|
|role_id|权限ID|

```php
// User.php
public function roles () {
    // 参数分别为. 多对多关联的表, 默认的中间枢纽表, 枢纽表中的两个关联字段名称
    return $this->belongsToMany('App\Role', 'role_user', 'user_id', 'role_id');
}
```
`Eloquent`会先从第一个表(User)中筛选出对应的主键, 用这个主键从枢纽表中筛选出需要的数据, 并内连接第二个表(Role) 返回结果.
```php
select `roles`.*, `role_user`.`user_id` as `pivot_user_id`, `role_user`.`role_id` as `pivot_role_id` from `roles` inner join `role_user` on `roles`.`id` = `role_user`.`role_id` where `role_user`.`user_id` = ?
```

// 远层一对多关联
如果要查看某个作者发布的文章下的所有评论, 则需要先从User表中获取对应的文章, 然后再从评论中再查找属于这篇文章的评论. 这就是远程一对多关联.

users
|字段|说明|
|:--|:--|
|id|主键|
|name|用户名|

articles
|字段|说明|
|:--|:--|
|id|主键|
|title|标题|
|content|内容|
|user_id|用户ID|

comments
|字段|说明|
|:--|:--|
|id|主键|
|article_id|文章ID|
|content|内容|

```php
// User.php
public function comments () {
    // 参数依次为 远程模型, 中间模型, 中间模型的外键, 远程模型的外键
    return $this->hasManyThrough('App\Comment', 'App\Article', 'user_id', 'article_id');
}
```
`Eloquent`会先从第一个表(User)中筛选出对应的主键, 用这个主键去中间表里找出对应的记录, 内连接第三个表, 返回结果.

### 多态关联
多态关联, 是指 有三个表.  

photos 图片表
|字段|说明|
|:--|:--|
|id|主键|
|path|路径|
|imageable_id|关联的ID|
|imageable_type|关联的模型类型. 本例中为 App\Staff或App\Orders|

staff 员工表
|字段|说明|
|:--|:--|
|id|主键|
|name|员工名|
orders 订单表
|字段|说明|
|:--|:--|
|id|主键|
|price|订单价格|

```php
// Photo.php
public function imageable () {
    // 获取对应的模型
    return $this->morphTo();
}

// Staff.php
protected $table = 'staff';
public function photos () {
    // 获取员工照片
    return $this->morphMany('App\Photo', 'imageable');
}

// Order.php
public function photos () {
    // 获取订单图片
    return $this->morphMany('App\Photo', 'imageable');
}
```

多态的神奇在于, 可以从 `Staff`或`Order`中获取对应的图片, 也可以从图片中获取该图片所属的拥有者
```php
// 获取员工ID为1的照片
Staff::find(1)->photos;

// 获取图片ID为1的拥有者
Photo::find(1)->imageable;
```

### 多态的多对多关联
设计一个标签系统, 文章和视频都可以打标签. 这里就需要用到多态的多对多关联.

articles
|字段|说明|
|:--|:--|
|id|主键|
|title|标题|
|content|内容|
|user_id|用户ID|

videos
|字段|说明|
|:--|:--|
|id|主键|
|title|标题|

tags
|字段|说明|
|:--|:--|
|id|主键|
|name|标签名|

taggables
|字段|说明|
|:--|:--|
|id|主键|
|tag_id|标签ID|
|taggable_id|关联的ID|
|taggable_type|关联的类型|

```php
// Tag.php
public function articles () {
    // 返回对应的拥有者
    return $this->morphedByMany('App\Article', 'taggable');
}
public function videos () {
    // 返回对应的拥有者
    return $this->morphedByMany('App\Video', 'taggable');
}

// Article.php Video.php
public function tags () {
    // 查找对应的标签
    return $this->morphToMany('App\Tag', 'taggable');
}

// Controller
// 获取 文章ID为1的标签
Article::find(1)->tags;
// 获取 标签ID为1的所有视频
Tag::find(1)->videos
```

## 关联查询
关联条件查询
```php
// 查找至少有一篇文章的用户
User::has('article')->get();
// 查找有四篇文章以上的用户
User::has('article', '>', 4)->get();
// 查找文章中有标签的用户
User::has('article.tags')->get();
```

## 动态属性
```php
// 只需要获取 article 属性, 内部会自己执行方法, 将获取的值传到该属性中
User::find(1)->article;
```

## 预载入
```php
// SQL执行一次
$users = User::all();
// SQL执行N次
foreach ($users as $user) {
    echo $user;
}
```

**这样会造成极大的浪费. 需要预载入**
```php
// SQL执行两次
$users = User::with('article')->get();
// SQL执行0次
foreach ($users as $user) {
    echo $user;
}
// 也可以这样载入
User::with('article.tags')->get();
```


### 预载入条件限制
预载入时加个查询限制
```
User::with(['article' => function ($query) {
    $query->where('id', '>', 3);
    $query->orderBy('id', 'desc');
}])->get();
```

### 延迟预载入
```php
$user = User::all();
$user->load('article');

$user->load([
    'article' => function ($query) {
        $query->where('id', '>', 3);
    }
]);
```

### 新增关联模型

#### 一对一关联
```php
$phone = new Phone([
    'phone' => '110'
]);
$user = User::find(5);
// $phone会自动设置user_id 为5
$data = $user->phone()->save($phone);
```

#### 一对多关联
```php
$articles = [
    new Article([
        'title' => '111',
        'content' => '111'
    ]),
    new Article([
        'title' => '222',
        'content' => '222'
    ]),
    new Article([
        'title' => '333',
        'content' => '333'
    ])
];
$user = User::find(5);
// 通过 saveMany 来保存多个数据
$user->article()->saveMany($articles);
```

#### 从属关联
```php
$user = User::find(4);
$phone = Phone::find(4);
// 通过 associate 更新从属关联. 最后一步需要 调用 save 方法
$phone->user()->associate($user);
$phone->save();
```

#### 多对多关联
```php
$user = User::find(2);
// 给ID为2的用户, 添加 身份ID为3的身份
$user->roles()->attach(3);
// 给ID为2的用户, 删除 身份ID为3的身份
$user->roles()->detach(3);
```
`attach`和`detach`也能接受一个数组, 批量新增或删除.
`sync`会将参数包含的ID或ID集合添加到枢纽表中, 将不在参数内的移除掉.
```php
// 先获取用户对应的身份ID, 删除或添加对应的身份ID.
$user->roles()->sync(3);
```
添加身份的时候, 可通过 save 方法新增身份, 且自动在枢纽表中添加关联
```php
$user = User::find(5);
$role = new Role([
    'name' => '来宾'
]);
$data = $user->roles()->save($role);
```

### 更新
#### 从属关联(更新上层时间戳)
当我们修改手机号码的时, 需要自动更新 User 表中的 updated_at.
```php
// Phone.php
// 指定自动更新上层时间戳.
protected $touches = ['user'];


// Controller
$phone = Phone::find(1);
$phone->phone = 110;
$phone->save();
// 保存后 User对应的updated_at 也就变了
```

#### 多对多关联(使用枢纽表)
`User::find(1)->roles;` 只能获取到`user_id`和`role_id`. 若需要获取其他属性, 则在模型里添加`withPivot`  
维护枢纽表的时间戳则采用 `withTimestamps`  
删除枢纽表中某个用户ID的所有关联数据, 直接调用 `detach`, 参数为空.
```php
// User.php
return $this->belongsToMany('App\Role')->withPivot('id');
return $this->belongsToMany('App\Role')->withTimestamps();
// 删除所有关联数据
User::find(1)->roles()->detach();
```

更新枢纽表的数据
```php
User::find(1)->roles()->updateExistingPivot(1, [
    'status' => true
]);
```
这将会更新枢纽表上 user_id为1 且 role_id为1的数据.

## 获取器和修改器
`Eloquent`可指定模型中获取或修改之前处理数据.
```php
// 获取状态. status 为 true 时, 返回字符 启用. 否则禁用.
public function getStatusAttribute($value) {
    if ($value) {
        return '启用';
    } else {
      return '禁用';
    }
}
// 根据值, 来设置对应的值.
public function setStatusAttribute($value) {
    if ($value == '启用') {
        $this->attributes['status'] = true;
    } else {
        $this->attributes['status'] = false;
    }
}
```

## 属性转换
将值获取之前转换成另一种属性
```php
protected $casts = [
    'is_admin' => 'boolean',
];
```

## 模型事件 模型观察者
`Eloquent`在操作数据库时, 会触发许多生命钩子. 模型观察者会监听这些生命钩子.