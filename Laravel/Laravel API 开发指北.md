# 环境
```php
PHP > 7.1
MySQL > 5.5
Redis > 2.8
Laravel 5.7
```

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

