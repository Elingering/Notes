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

## OAuth 认证 Passport

### OAuth2 授权码模式
授权码模式，主要应用在平台给第三方的应用进行用户授权，简单梳理一下流程，假设我们有 Larabbs 开放平台，第三方的应用 Larabbs-Blog，希望用户可以直接通过 Larabbs 中的账户密码登录，并且获取用户在 Larabbs 中发布的话题展示出来。

1. Larabbs 为 Larabbs-Blog 创建客户端应用，并且分配 client_id 和 client_secret 给 Larabbs-Blog；
2. 用户在 Larabbs-Blog 的登录界面，点击 使用 Larabbs 登录；
3. Larabbs-Blog 跳转到 Larabbs 的用户授权页面，用户在输入用户名密码登录，授权 Larabbs-Blog 可以获取 Larabbs 中的信息；
4. Larabbs 跳转回 Larabbs-Blog，并返回授权码；
5. Larabbs-Blog 通过 client_secret 以及授权码获取到 access_token，然后通过 access_token 获取 Larabbs 中的用户以及话题信息。

上面是一个完整的 OAuth2 授权流程，可以看到使用授权码模式，Larabbs-Blog 没有任何机会获取到用户的密码，而且只有用户在 Larabbs 中同意授权以后，Larabbs-Blog 才能获取用户的信息，保证了用户数据的安全。

授权码模式是完整基础的 OAuth2 流程，我们通常说的第三方登录都是指授权码模式。

### OAuth2 密码模式
但是对于我们自己的客户端，比如 Larabbs 的 IOS 客户端，中间的授权码流程就显得有些多余，这时 OAuth2 的另一个模式 —— 密码模式，就很好的解决了这个问题。

对于我们自己的客户端，用户应该直接在客户端中输入用户名和密码，客户端直接通过用户数据的用户名和密码获取 access_token 即可。

密码模式流程如下：
- 用户在客户端输入用户名和密码；
- 客户端提交用户名，密码，client_id 和 client_secret 到服务器；
- 服务器直接返回 access_token；

可以看到密码模式的流程非常简洁，我们可以方便的向自己的客户端发出访问令牌，而不需要遍历整个 OAuth2 授权代码重定向流程。

### Passport

#### 安装
```php
$ composer require laravel/passport
$ php artisan passport:keys
$ php artisan passport:client --password --name='larabbs-ios'
```
![title](https://raw.githubusercontent.com/Elingering/note-images/master/note-images/2019/07/11/1562829422574-1562829422577.png?token=AFRM333WHG2ALM25YQPM34K5E3RKY)

#### 注册路由
```php
app/Providers/AuthServiceProvider.php

use Laravel\Passport\Passport;

public function boot()
{
    // Passport 的路由
    Passport::routes();
    // access_token 过期时间
    Passport::tokensExpireIn(Carbon::now()->addDays(15));
    // refreshTokens 过期时间
    Passport::refreshTokensExpireIn(Carbon::now()->addDays(30));
}
```

#### 获取令牌
密码模式我们通过 larabbs.test/oauth/token 这个路由获取访问令牌。提交的参数如下：
- grant_type —— 密码模式固定为 password；
- client_id —— 通过 passport:client 创建的客户端 id；
- client_secret —— 通过 passport:client 创建的客户端 secret；
- username —— 登录的用户名，数据库中任意用户邮箱；
- password —— 用户密码；
- scope —— 作用域，可填写 * 或者为空；

返回数据：
- token_type —— 令牌类型；
- expires_in—— 多长时间后过期；
- access_token —— 访问令牌；
- refresh_token —— 刷新令牌；

#### 刷新访问令牌
刷新访问令牌 接口与 获取访问令牌 接口一样，只是参数不同。
- grant_type —— 刷新令牌固定为 refresh_token；
- client_id —— 通过 passport:client 创建的客户端 id；
- client_secret —— 通过 passport:client 创建的客户端 secret；
- refresh_token —— 刷新令牌；
- scope —— 作用域，可填写 * 或者为空；

#### 登录
Passport 提供的默认路由为 http://larabbs.test/oauth/token ，而我们的现在接口统一都有 /api 的前缀，所以我们不使用 Passport 默认的路由，依然使用 /api/authorizations。先来修改登录接口：
```php
use Psr\Http\Message\ServerRequestInterface;
use League\OAuth2\Server\AuthorizationServer;
use Zend\Diactoros\Response as Psr7Response;
use League\OAuth2\Server\Exception\OAuthServerException;
.
public function store(AuthorizationRequest $originRequest, AuthorizationServer $server, ServerRequestInterface $serverRequest)
{
    try {
       return $server->respondToAccessTokenRequest($serverRequest, new Psr7Response)->withStatus(201);
    } catch(OAuthServerException $e) {
        return $this->response->errorUnauthorized($e->getMessage());
    }
}
```
逻辑很简单，我们注入了 AuthorizationServer 和 ServerRequestInterface ，调用 AuthorizationServer 的 respondToAccessTokenRequest 方法并直接返回。

respondToAccessTokenRequest 会依次处理：
- 检测 client 参数是否正确；
- 检测 scope 参数是否正确；
- 通过用户名查找用户；
- 验证用户密码是否正确；
- 生成 Response 并返回；

#### 支持手机登录
Passport 会通过用户的邮箱查找用户，要支持手机登录，我们可以在用户模型定义了 findForPassport 方法，Passport 会先检测用户模型是否存在 findForPassport 方法，如果存在就通过 findForPassport 查找用户，而不是使用默认的邮箱。
```php
app/Models/User.php
public function findForPassport($username)
{
    filter_var($username, FILTER_VALIDATE_EMAIL) ?
        $credentials['email'] = $username :
        $credentials['phone'] = $username;

    return self::where($credentials)->first();
}
```

#### 刷新token
```php
app/Http/Controllers/Api/AuthorizationsController.php
public function update(AuthorizationServer $server, ServerRequestInterface $serverRequest)
    {
        try {
           return $server->respondToAccessTokenRequest($serverRequest, new Psr7Response);
        } catch(OAuthServerException $e) {
            return $this->response->errorUnauthorized($e->getMessage());
        }
    }
```

#### 获取登录用户信息
- 将 Laravel\Passport\HasApiTokens Trait 添加到 App\Models\User 模型中，这个 Trait 会给你的模型提供一些辅助函数，用于检查已认证用户的令牌和使用范围。
- 修改 auth 配置，将 api guard 的 driver 由 jwt 修改为 passport。
- 增加了一个 PassportDingoProvider，因为 DingoApi 没有做 Passport 的适配，所以需要手动处理一下：
```php
$ php artisan make:provider PassportDingoProvider

<?php
namespace App\Providers;

use Dingo\Api\Routing\Route;
use Illuminate\Http\Request;
use Illuminate\Auth\AuthManager;
use Dingo\Api\Auth\Provider\Authorization;
use Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException;

class PassportDingoProvider extends Authorization
{
    protected $auth;

    protected $guard = 'api';

    public function __construct(AuthManager $auth)
    {
        $this->auth = $auth->guard($this->guard);
    }

    public function authenticate(Request $request, Route $route)
    {
        if (! $user = $this->auth->user()) {
            throw new UnauthorizedHttpException(
                get_class($this),
                'Unable to authenticate with invalid API key and token.'
            );
        }

        return $user;
    }

    public function getAuthorizationMethod()
    {
        return 'Bearer';
    }
}

将原来的 jwt 替换为 oauth，值为刚才创建的 Provider，这样在 Controller 中我们使用 $this->user() 就能获取到令牌对应的用户模型了。

config/api.php
'auth' => [
    //'jwt' => 'Dingo\Api\Auth\Provider\JWT',
    'oauth' => \App\Providers\PassportDingoProvider::class,
],
```

#### 删除token
```php
$this->user()->token()->revoke();
```

#### 处理第三方登录——直接生成访问令牌
```php
创建一个Trait
$ mkdir app/Traits
$ touch app/Traits/PassportToken.php

<?php

namespace App\Traits;

use App\Models\User;
use DateTime;
use GuzzleHttp\Psr7\Response;
use Illuminate\Events\Dispatcher;
use Laravel\Passport\Bridge\AccessToken;
use Laravel\Passport\Bridge\AccessTokenRepository;
use Laravel\Passport\Bridge\Client;
use Laravel\Passport\Bridge\RefreshTokenRepository;
use Laravel\Passport\Passport;
use Laravel\Passport\TokenRepository;
use League\OAuth2\Server\CryptKey;
use League\OAuth2\Server\Entities\AccessTokenEntityInterface;
use League\OAuth2\Server\Exception\OAuthServerException;
use League\OAuth2\Server\Exception\UniqueTokenIdentifierConstraintViolationException;
use League\OAuth2\Server\ResponseTypes\BearerTokenResponse;

# https://github.com/laravel/passport/issues/71

trait PassportToken
{
    private function generateUniqueIdentifier($length = 40)
    {
        try {
            return bin2hex(random_bytes($length));
            // @codeCoverageIgnoreStart
        } catch (\TypeError $e) {
            throw OAuthServerException::serverError('An unexpected error has occurred');
        } catch (\Error $e) {
            throw OAuthServerException::serverError('An unexpected error has occurred');
        } catch (\Exception $e) {
            // If you get this message, the CSPRNG failed hard.
            throw OAuthServerException::serverError('Could not generate a random string');
        }
        // @codeCoverageIgnoreEnd
    }

    private function issueRefreshToken(AccessTokenEntityInterface $accessToken)
    {
        $maxGenerationAttempts = 10;
        $refreshTokenRepository = app(RefreshTokenRepository::class);

        $refreshToken = $refreshTokenRepository->getNewRefreshToken();
        $refreshToken->setExpiryDateTime((new \DateTime())->add(Passport::refreshTokensExpireIn()));
        $refreshToken->setAccessToken($accessToken);

        while ($maxGenerationAttempts-- > 0) {
            $refreshToken->setIdentifier($this->generateUniqueIdentifier());
            try {
                $refreshTokenRepository->persistNewRefreshToken($refreshToken);

                return $refreshToken;
            } catch (UniqueTokenIdentifierConstraintViolationException $e) {
                if ($maxGenerationAttempts === 0) {
                    throw $e;
                }
            }
        }
    }

    protected function createPassportTokenByUser(User $user, $clientId)
    {
        $accessToken = new AccessToken($user->id);
        $accessToken->setIdentifier($this->generateUniqueIdentifier());
        $accessToken->setClient(new Client($clientId, null, null));
        $accessToken->setExpiryDateTime((new DateTime())->add(Passport::tokensExpireIn()));

        $accessTokenRepository = new AccessTokenRepository(new TokenRepository(), new Dispatcher());
        $accessTokenRepository->persistNewAccessToken($accessToken);
        $refreshToken = $this->issueRefreshToken($accessToken);

        return [
            'access_token' => $accessToken,
            'refresh_token' => $refreshToken,
        ];
    }

    protected function sendBearerTokenResponse($accessToken, $refreshToken)
    {
        $response = new BearerTokenResponse();
        $response->setAccessToken($accessToken);
        $response->setRefreshToken($refreshToken);

        $privateKey = new CryptKey('file://'.Passport::keyPath('oauth-private.key'), null, false);

        $response->setPrivateKey($privateKey);
        $response->setEncryptionKey(app('encrypter')->getKey());

        return $response->generateHttpResponse(new Response);
    }

    protected function getBearerTokenByUser(User $user, $clientId, $output = true)
    {
        $passportToken = $this->createPassportTokenByUser($user, $clientId);
        $bearerToken = $this->sendBearerTokenResponse($passportToken['access_token'], $passportToken['refresh_token']);

        if (! $output) {
            $bearerToken = json_decode($bearerToken->getBody()->__toString(), true);
        }

        return $bearerToken;
    }
}

第三方登录中引入Trait，并使用
$result = $this->getBearerTokenByUser($user, '1', false);
```





