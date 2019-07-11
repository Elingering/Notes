# 环境
```php
PHP > 7.1
MySQL > 5.5
Redis > 2.8
Laravel 5.7
```
- 为了模拟AJAX请求，请将 header头 设置X-Requested-With 为 XMLHttpRequest

# 初始化数据

## Model
划分一个Models

## 控制器
```php
php artisan make:controller Api/UserController
```

## 路由
```php
<?php
use Illuminate\Http\Request;

Route::namespace('Api')->prefix('v1')->group(function () {
    Route::get('/users','UserController@index')->name('users.index');
});
```

## 验证器
```php
基类验证器
php artisan make:request Api/FormRequest
普通验证器
php artisan make:request Api/UserRequest
```

## 使用name登录（默认是email）
```php
app/Http/Controllers/auth/LoginController.php 加入：
public function username()
{
    return 'name';
}
```

# 构造

## 跨域问题
```php
- 安装：composer require medz/cors

- 配置：php artisan vendor:publish --provider="Medz\Cors\Laravel\Providers\LaravelServiceProvider" --force

- 修改配置文件config/cors.php：
return [
    ......
    'expose-headers'     => ['Authorization'],
    ......
];

- 增加中间件app/Http/Kernel.php：
protected $routeMiddleware = [
        ...... //前面的中间件
        'cors'=> \Medz\Cors\Laravel\Middleware\ShouldGroup::class,
];

- 路由增加中间件
```

## 统一 Response 响应处理

### 封装返回的统一消息
```php
app/Api/Helpers/ApiResponse.php
<?php
namespace App\Api\Helpers;
use Symfony\Component\HttpFoundation\Response as FoundationResponse;
use Response;

trait ApiResponse
{
    /**
     * @var int
     */
    protected $statusCode = FoundationResponse::HTTP_OK;

    /**
     * @return mixed
     */
    public function getStatusCode()
    {
        return $this->statusCode;
    }

    /**
     * @param $statusCode
     * @return $this
     */
    public function setStatusCode($statusCode,$httpCode=null)
    {
        $httpCode = $httpCode ?? $statusCode;
        $this->statusCode = $statusCode;
        return $this;
    }

    /**
     * @param $data
     * @param array $header
     * @return mixed
     */
    public function respond($data, $header = [])
    {

        return Response::json($data,$this->getStatusCode(),$header);
    }

    /**
     * @param $status
     * @param array $data
     * @param null $code
     * @return mixed
     */
    public function status($status, array $data, $code = null){

        if ($code){
            $this->setStatusCode($code);
        }
        $status = [
            'status' => $status,
            'code' => $this->statusCode
        ];

        $data = array_merge($status,$data);
        return $this->respond($data);

    }

    /**
     * @param $message
     * @param int $code
     * @param string $status
     * @return mixed
     */
    /*
     * 格式
     * data:
     *  code:422
     *  message:xxx
     *  status:'error'
     */
    public function failed($message, $code = FoundationResponse::HTTP_BAD_REQUEST,$status = 'error'){

        return $this->setStatusCode($code)->message($message,$status);
    }

    /**
     * @param $message
     * @param string $status
     * @return mixed
     */
    public function message($message, $status = "success"){

        return $this->status($status,[
            'message' => $message
        ]);
    }

    /**
     * @param string $message
     * @return mixed
     */
    public function internalError($message = "Internal Error!"){

        return $this->failed($message,FoundationResponse::HTTP_INTERNAL_SERVER_ERROR);
    }

    /**
     * @param string $message
     * @return mixed
     */
    public function created($message = "created")
    {
        return $this->setStatusCode(FoundationResponse::HTTP_CREATED)
            ->message($message);

    }

    /**
     * @param $data
     * @param string $status
     * @return mixed
     */
    public function success($data, $status = "success"){

        return $this->status($status,compact('data'));
    }

    /**
     * @param string $message
     * @return mixed
     */
    public function notFond($message = 'Not Fond!')
    {
        return $this->failed($message,Foundationresponse::HTTP_NOT_FOUND);
    }
}
```

### 使用响应
- 新建Api控制器基类，引入ApiResponse Trait；
- 继承基类控制器

```php
//返回正确消息
return $this->success('用户登录成功...');
//返回正确资源消息
return $this->success($user);
//返回自定义 http 状态码的正确信息
return $this->setStatusCode(201)->success('用户登录成功...');
//返回错误信息
return $this->failed('用户注册失败');
//返回自定义 http 状态码的错误信息
return $this->failed('用户登录失败',401);
//返回自定义 http 状态码的错误信息，同时也想返回自己内部定义的错误码
return $this->failed('用户登录失败',401,10001);
```

## Api-Resource 资源

### 创建资源
```php
php artisan make:resource Api/UserResource
app/Http/Resources/Api/UserResource.php
<?php

namespace App\Http\Resources\Api;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * Transform the resource into an array.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array
     */
    public function toArray($request)
    {
        switch ($this->status){
            case -1:
                $this->status = '已删除';
                break;
            case 0:
                $this->status = '正常';
                break;
            case 1:
                $this->status = '冻结';
                break;
        }
        return [
            'id'=>$this->id,
            'name' => $this->name,
            'status' => $this->status,
            'created_at'=>(string)$this->created_at,
            'updated_at'=>(string)$this->updated_at
        ];
    }
}
```

### 使用资源
```php
//返回单一的资源
return $this->success(new UserResource($user));
//返回资源列表
$users = User::paginate(3);
return UserResource::collection($users);
```

## Enum 枚举
- 自己模拟app/Models/Enum/UserEnum.php
- 使用第三方扩展（）//TODO

## 异常处理

### 创建自定义异常处理
app/Api/Helpers/ExceptionReport.php
```php
<?php

namespace App\Api\Helpers;

use Exception;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Auth\AuthenticationException;
use Illuminate\Database\Eloquent\ModelNotFoundException;
use Illuminate\Http\Request;
use Illuminate\Validation\ValidationException;
use Symfony\Component\HttpKernel\Exception\MethodNotAllowedHttpException;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException;
use Tymon\JWTAuth\Exceptions\TokenInvalidException;

class ExceptionReport
{
    use ApiResponse;

    /**
     * @var Exception
     */
    public $exception;
    /**
     * @var Request
     */
    public $request;

    /**
     * @var
     */
    protected $report;

    /**
     * ExceptionReport constructor.
     * @param Request $request
     * @param Exception $exception
     */
    function __construct(Request $request, Exception $exception)
    {
        $this->request = $request;
        $this->exception = $exception;
    }

    /**
     * @var array
     */
    //当抛出这些异常时，可以使用我们定义的错误信息与HTTP状态码
    //可以把常见异常放在这里
    public $doReport = [
        AuthenticationException::class => ['未授权',401],
        ModelNotFoundException::class => ['该模型未找到',404],
        AuthorizationException::class => ['没有此权限',403],
        ValidationException::class => [],
        UnauthorizedHttpException::class=>['未登录或登录状态失效',422],
        TokenInvalidException::class=>['token不正确',400],
        NotFoundHttpException::class=>['没有找到该页面',404],
        MethodNotAllowedHttpException::class=>['访问方式不正确',405],
        QueryException::class=>['参数错误',401],
    ];

    public function register($className,callable $callback){

        $this->doReport[$className] = $callback;
    }

    /**
     * @return bool
     */
    public function shouldReturn(){
    //只有请求包含是json或者ajax请求时才有效
//        if (! ($this->request->wantsJson() || $this->request->ajax())){
//
//            return false;
//        }
        foreach (array_keys($this->doReport) as $report){
            if ($this->exception instanceof $report){
                $this->report = $report;
                return true;
            }
        }

        return false;

    }

    /**
     * @param Exception $e
     * @return static
     */
    public static function make(Exception $e){

        return new static(\request(),$e);
    }

    /**
     * @return mixed
     */
    public function report(){
        if ($this->exception instanceof ValidationException){
            $error = array_first($this->exception->errors());
            return $this->failed(array_first($error),$this->exception->status);
        }
        $message = $this->doReport[$this->report];
        return $this->failed($message[0],$message[1]);
    }
    public function prodReport(){
        return $this->failed('服务器错误','500');
    }
}
```

### 捕捉异常
app/Exceptions/Handler.php
```php
<?php

namespace App\Exceptions;
use App\Api\Helpers\ExceptionReport;
use Exception;
use Illuminate\Foundation\Exceptions\Handler as ExceptionHandler;

class Handler extends ExceptionHandler
{

    public function render($request, Exception $exception)
    {
        //ajax请求我们才捕捉异常
        if ($request->ajax()){
            // 将方法拦截到自己的ExceptionReport
            $reporter = ExceptionReport::make($exception);
            if ($reporter->shouldReturn()){
                return $reporter->report();
            }
            if(env('APP_DEBUG')){
                //开发环境，则显示详细错误信息
                return parent::render($request, $exception);
            }else{
                //线上环境,未知错误，则显示500
                return $reporter->prodReport();
            }
        }
        return parent::render($request, $exception);
    }
}
```

## jwt-auth

### 安装
```php
composer require tymon/jwt-auth 1.0.0-rc.3
```

### 配置
```php
- 添加服务提供商 config/app.php
'providers' => [
    ...
    Tymon\JWTAuth\Providers\LaravelServiceProvider::class,
]；
- 发布配置文件
php artisan vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"
- 生成秘钥
php artisan jwt:secret
- 配置 Auth guard config/auth.php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
       'driver' => 'jwt',
       'provider' => 'users',
    ],
],
- 更改 Model
<?php

namespace App\Models;

use Illuminate\Notifications\Notifiable;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Tymon\JWTAuth\Contracts\JWTSubject;

class User extends Authenticatable implements JWTSubject
{
    use Notifiable;

    public function getJWTIdentifier()
    {
        return $this->getKey();
    }

    public function getJWTCustomClaims()
    {
        return [];
    }
    ......
```

### 使用
```php
登录获取token：
$token=Auth::guard('api')->attempt(['name'=>$request->name,'password'=>$request->password]);
['token' => 'bearer ' . $token]
返回当前登录用户信息：
$user = Auth::guard('api')->user();
退出：
Auth::guard('api')->logout();
```

### 请求
在 Postman 的 Header 头部分再加一个 key 为 Authorization，value 为登陆成功后返回的 token 值，然后再次进行请求，可以看到成功返回当前登陆用户的信息。

## 自动刷新用户认证

### 自定义认证中间件
```php
php artisan make:middleware Api/RefreshTokenMiddleware
<?php

namespace App\Http\Middleware\Api;

use Auth;
use Closure;
use Tymon\JWTAuth\Exceptions\JWTException;
use Tymon\JWTAuth\Facades\JWTAuth;
use Tymon\JWTAuth\Http\Middleware\BaseMiddleware;
use Tymon\JWTAuth\Exceptions\TokenExpiredException;
use Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException;

// 注意，我们要继承的是 jwt 的 BaseMiddleware
class RefreshTokenMiddleware extends BaseMiddleware
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request $request
     * @param  \Closure $next
     *
     * @throws \Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException
     *
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        // 检查此次请求中是否带有 token，如果没有则抛出异常。
        $this->checkForToken($request);
//         使用 try 包裹，以捕捉 token 过期所抛出的 TokenExpiredException  异常
        try {
            // 检测用户的登录状态，如果正常则通过
            if ($this->auth->parseToken()->authenticate()) {
                return $next($request);
            }
            throw new UnauthorizedHttpException('jwt-auth', '未登录');
        } catch (TokenExpiredException $exception) {
            // 此处捕获到了 token 过期所抛出的 TokenExpiredException 异常，我们在这里需要做的是刷新该用户的 token 并将它添加到响应头中
            try {
                // 刷新用户的 token
                $token = $this->auth->refresh();
                // 使用一次性登录以保证此次请求的成功
                Auth::guard('api')->onceUsingId($this->auth->manager()->getPayloadFactory()->buildClaimsCollection()->toPlainArray()['sub']);
            } catch (JWTException $exception) {
                // 如果捕获到此异常，即代表 refresh 也过期了，用户无法刷新令牌，需要重新登录。
                throw new UnauthorizedHttpException('jwt-auth', $exception->getMessage());
            }
        }

        // 在响应头中返回新的 token
        return $this->setAuthenticationHeader($next($request), $token);
    }
}
```

### 注册中间件
app/Http/Kernel.php
```php
protected $routeMiddleware = [
    ......
    'api.refresh'=>\App\Http\Middleware\Api\RefreshTokenMiddleware::class,
];
```
- 在路由中使用
- 与前端商定一个token失效的状态码，用于重新登录

## 多角色认证

### 用户认证文件
config/auth.php
```php
'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],

        'api' => [
            'driver' => 'jwt',
            'provider' => 'users',
        ],

        'admin' => [
            'driver' => 'jwt',
            'provider' => 'admins',
        ],
],
'providers' => [
        'users' => [
            'driver' => 'eloquent',
            'model' => App\Models\User::class,
        ],
        'admins' => [
            'driver' => 'eloquent',
            'model' => App\Models\Admin::class,
        ],
        // 'users' => [
        //     'driver' => 'database',
        //     'table' => 'users',
        // ],
    ],
```
同User一样，copy一份Admin：
- Admin 用户表
- 框架文件：model、controller、request、resource...
- copy一份认证中间件
- 开始使用 （Auth::guard('admin')）

### 自动区分 guard

#### 新建中间件
```php
php artisan make:middleware Api/AdminGuardMiddleware
<?php

namespace App\Http\Middleware\Api;
use Closure;
class AdminGuardMiddleware
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request $request
     * @param  \Closure $next
     *
     * @throws \Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException
     *
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        config(['auth.defaults.guard'=>'admin']);
        return $next($request);
    }
}
```
- 注册中间件
- 在后台路由组中使用
- 前台的同理copy一份

