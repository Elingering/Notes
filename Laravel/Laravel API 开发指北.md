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
新建Api基类