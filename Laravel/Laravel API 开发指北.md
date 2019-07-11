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
return UserResource::collection($users);
```

