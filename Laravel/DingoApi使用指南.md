# 安装
```shell
$ composer require dingo/api
```
# 配置
```shell
$ php artisan vendor:publish --provider="Dingo\Api\Provider\LaravelServiceProvider"

.
.
.
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
### 路由管理
### Transform




