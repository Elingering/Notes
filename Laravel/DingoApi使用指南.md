# 安装
```php
$ composer require dingo/api
```

# 配置
```php
$ php artisan vendor:publish --provider="Dingo\Api\Provider\LaravelServiceProvider"

.env
API_STANDARDS_TREE=prs
API_SUBTYPE=larabbs
API_PREFIX=api
API_VERSION=v1
API_DEBUG=true
```

# API版本
使用 Accept 头来指定我们需要访问的 API 版
```html
Accept: application/<API_STANDARDS_TREE>.<API_SUBTYPE>.v1+json
例：Accept: application/prs.larabbs.v1+json
```

# 开始使用

## 控制器

### 新建控制器基类
```php
$ php artisan make:controller Api/Controller
使用trait use Dingo\Api\Routing\Helpers;
```

## 表单验证基类
```php
$ php artisan make:request Api/FormRequest
```

## 返回信息
```php
响应一个数组
return $this->response->array($user->toArray());
响应一个元素
return $this->response->item($user, new UserTransformer);
响应一个元素集合
return $this->response->collection($users, new UserTransformer);
分页响应
$users = User::paginate(25);
return $this->response->paginator($users, new UserTransformer);
无内容响应
return $this->response->noContent();
创建了资源的响应
return $this->response->created();
return $this->response->created($location);

错误响应
// 一个自定义消息和状态码的普通错误。
return $this->response->error('This is an error.', 404);
// 一个没有找到资源的错误，第一个参数可以传递自定义消息。
return $this->response->errorNotFound();
// 一个 bad request 错误，第一个参数可以传递自定义消息。
return $this->response->errorBadRequest();
// 一个服务器拒绝错误，第一个参数可以传递自定义消息。
return $this->response->errorForbidden();
// 一个内部错误，第一个参数可以传递自定义消息。
return $this->response->errorInternal();
// 一个未认证错误，第一个参数可以传递自定义消息。
return $this->response->errorUnauthorized();

添加额外的头信息
return $this->response->item($user, new UserTransformer)->withHeader('X-Foo', 'Bar');
添加Meta信息
return $this->response->item($user, new UserTransformer)->addMeta('foo', 'bar');
$meta = [
    'foo' => 'bar',
    'css' => 'less',
];
return $this->response->item($user, new UserTransformer)->setMeta($meta);

设置响应状态码
return $this->response->item($user, new UserTransformer)->setStatusCode(200);
```

## 路由管理
```php
$api = app('Dingo\Api\Routing\Router');
$api->version('v1', [
    'namespace' => 'App\Http\Controllers\Api',
    'middleware' => ['serializer:array', 'bindings', 'change-locale']
], function ($api) {
    $api->group([
        'middleware' => 'api.throttle',
        'limit' => config('api.rate_limits.sign.limit'),
        'expires' => config('api.rate_limits.sign.expires'),
    ], function ($api) {
        // 用户注册
        $api->post('users', 'UsersController@store')->name('api.users.store');
    });
    $api->group([
        'middleware' => 'api.throttle',
        'limit' => config('api.rate_limits.access.limit'),
        'expires' => config('api.rate_limits.access.expires'),
    ], function ($api) {
        // 游客可以访问的接口
        // 话题列表
        $api->get('topics', 'TopicsController@index')->name('api.topics.index');
        // 需要 token 验证的接口
        $api->group(['middleware' => 'api.auth'], function($api) {
            // 当前登录用户信息
            $api->get('user', 'UsersController@me')->name('api.user.show');
        });
    });
});
```

## JWT

### 安装jwt-auth及使用
```php
$ composer require tymon/jwt-auth:1.0.0-rc.4.1
$ php artisan jwt:secret

修改 config/auth.php，将 api guard 的 driver 改为 jwt。

修改 config/api.php，auth 中增加 JWT 相关的配置：
.
'auth' => [
    'jwt' => 'Dingo\Api\Auth\Provider\JWT',
],
.

user 模型需要继承 Tymon\JWTAuth\Contracts\JWTSubject 接口，并实现接口的两个方法 getJWTIdentifier () 和 getJWTCustomClaims ()。getJWTIdentifier 返回了 User 的 id，getJWTCustomClaims 是我们需要额外再 JWT 载荷中增加的自定义内容：

use Tymon\JWTAuth\Contracts\JWTSubject;
class User extends Authenticatable implements MustVerifyEmailContract, JWTSubject
{
    public function getJWTIdentifier()
    {
        return $this->getKey();
    }

    public function getJWTCustomClaims()
    {
        return [];
    }
}
```

### 登录获取token
```php
账号密码登录：
$credentials = [
    'email' => 'lzqiang@qq.com',
//    'phone' => '18611606666',
    'password' => 'secret',
];
$token = \Auth::guard('api')->attempt($credentials);
第三方登录：
$token = \Auth::guard('api')->fromUser($user);
token类型：
$token_type = 'Bearer';
token过期时间：
$expires_in = \Auth::guard('api')->factory()->getTTL() * 60;
```

### 刷新、删除token
```php
Auth::guard('api')->refresh();
Auth::guard('api')->logout();
```

## Transform（Fractal）

### 数据结构
DataArraySerializer
ArraySerializer //会减少数据嵌套层级
JsonApiSerializer

### 创建 UserTransformer
```php
$ touch app/Transformers/UserTransformer.php

<?php

namespace App\Transformers;

use App\Models\User;
use League\Fractal\TransformerAbstract;

class UserTransformer extends TransformerAbstract
{
    public function transform(User $user)
    {
        return [
            'id' => $user->id,
            'name' => $user->name,
            'email' => $user->email,
            'avatar' => $user->avatar,
            'introduction' => $user->introduction,
            'bound_phone' => $user->phone ? true : false,
            'bound_wechat' => ($user->weixin_unionid || $user->weixin_openid) ? true : false,
            'last_actived_at' => $user->last_actived_at->toDateTimeString(),
            'created_at' => (string) $user->created_at,
            'updated_at' => (string) $user->updated_at,
        ];
    }
}
```

### 响应结构保持统一
```php
少一层嵌套的 ArraySerializer：
$ composer require liyu/dingo-serializer-switch
路由文件加入中间件：
'middleware' => 'serializer:array'
```

## 路由模型绑定出了问题
因为路由交给 DingoApi 来处理了，所以模型绑定的中间件并没有注册上。手动增加 bindings 中间件。
```php
$api->version('v1', [
    'namespace' => 'App\Http\Controllers\Api',
    'middleware' => ['serializer:array', 'bindings']
], function ($api) {
```

## API的异常处理
```php
手动处理异常 app/Providers/AppServiceProvider.php

public function register()
{
    \API::error(function (\Illuminate\Database\Eloquent\ModelNotFoundException $exception) {
        abort(404);
    });

    \API::error(function (\Illuminate\Auth\Access\AuthorizationException $exception) {
        abort(403, $exception->getMessage());
    });
}
```

## Include机制
```php
<?php

namespace App\Transformers;

use App\Models\Topic;
use League\Fractal\TransformerAbstract;

class TopicTransformer extends TransformerAbstract
{
    protected $availableIncludes = ['user', 'category'];

    public function transform(Topic $topic)
    {
        return [
            'id' => $topic->id,
            'title' => $topic->title,
            'body' => $topic->body,
            'user_id' => $topic->user_id,
            'category_id' => $topic->category_id,
            'reply_count' => $topic->reply_count,
            'view_count' => $topic->view_count,
            'last_reply_user_id' => $topic->last_reply_user_id,
            'excerpt' => $topic->excerpt,
            'slug' => $topic->slug,
            'created_at' => $topic->created_at->toDateTimeString(),
            'updated_at' => $topic->updated_at->toDateTimeString(),
        ];
    }

    public function includeUser(Topic $topic)
    {
        return $this->item($topic->user, new UserTransformer());
    }

    public function includeCategory(Topic $topic)
    {
        return $this->item($topic->category, new CategoryTransformer());
    }
}
首先设置了 protected $availableIncludes = ['user', 'category']，可以理解为可以嵌套的额外资源有 user 和 category。
availableIncludes 中的每一个参数都对应一个具体的方法，方法命名规则为 include + user 、 include + category 驼峰命名。
由客户端提交的 include 参数指定，什么时候引入额外的资源，多个参数通过逗号分隔。例：http://local/api/topics?include=topic,topic.category
```

## N+1 问题以及查询日志
```php
日志查询组件
$ composer require overtrue/laravel-query-logger --dev
查看
$ tail -f ./storage/logs/laravel.log
```
通常不会出现N+1问题。如果遇到复杂的嵌套及关系加载，可以 app(\Dingo\Api\Transformer\Factory::class)->disableEagerLoading(); 临时关闭 DingoApi 的预加载，手动处理：
```php
public function index(Topic $topic, Request $request)
{
    app(\Dingo\Api\Transformer\Factory::class)->disableEagerLoading();

    $replies = $topic->replies()->paginate(20);

    if ($request->include) {
        $replies->load(explode(',', $request->include));
    }

    return $this->response->paginator($replies, new ReplyTransformer());
}
```

## 本地化
接口根据客户端语言切换错误信息：
- Accept-Language zh-CN —— 简体中文
- Accept-Language en —— 英文

### 增加middleware
```php
$ php artisan make:middleware ChangeLocale

<?php

namespace App\Http\Middleware;

use Closure;

class ChangeLocale
{
    public function handle($request, Closure $next)
    {
        $language = $request->header('accept-language');
        if ($language) {
            \App::setLocale($language);
        }

        return $next($request);
    }
}
```

### 注册middleware
```php
app/Http/Kernel.php
protected $routeMiddleware = [
// 接口语言设置
'change-locale' => \App\Http\Middleware\ChangeLocale::class,
];
routes/api.php
$api->version('v1', [
    'namespace' => 'App\Http\Controllers\Api',
    'middleware' => ['serializer:array', 'bindings', 'change-locale']
], function ($api) {
```

## 极光推送









