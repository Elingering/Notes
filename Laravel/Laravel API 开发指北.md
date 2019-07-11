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

### 修改配置文件
