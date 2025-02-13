# Keeper - 引导程序 (Bootloaders)

Keeper 包含以下引导程序:

- `Spiral\Keeper\Bootloader\KeeperBootloader` 是所有其他引导程序的入口点
- `Spiral\Keeper\Bootloader\GuestBootloader` 授予访客用户完全访问权限 (仅用于测试)
- `Spiral\Keeper\Bootloader\AnnotatedBootloader` 读取 `Controller` 和 `Action` 注解 (参见 [路由](../keeper/routing.md))
- `Spiral\Keeper\Bootloader\SitemapBootloader` 读取站点地图注解，也可以用于通过代码定义 `Sitemap` (参见 [站点地图](../keeper/sitemap.md))
- `Spiral\Keeper\Bootloader\UIBootloader` 注册 keeper 视图 - 布局、侧边栏、面包屑、网格等 (参见 [站点地图](../keeper/sitemap.md) 和 [视图](../keeper/views.md))

## 使用方法

`Spiral\Keeper\Bootloader\KeeperBootloader` 是抽象类。首先，创建你自己的继承的引导程序。如果需要，所有其他的 keeper 引导程序都应该在 `LOAD` 常量中注册。

```php
use App\Bootloader\Keeper;
use Spiral\DataGrid\Interceptor\GridInterceptor;
use Spiral\Domain\CycleInterceptor;
use Spiral\Domain\FilterInterceptor;
use Spiral\Domain\GuardInterceptor;
use Spiral\Keeper\Bootloader;
use Spiral\Keeper\Middleware;

class KeeperBootloader extends Bootloader\KeeperBootloader
{
    protected const NAMESPACE          = 'keeper';
    protected const PREFIX             = 'keeper/';
    protected const DEFAULT_CONTROLLER = 'dashboard';
    protected const CONFIG_NAME        = '';

    protected const LOAD = [
        Bootloader\SitemapBootloader::class,
        Bootloader\AnnotatedBootloader::class,
    ];

    protected const MIDDLEWARE = [
        Middleware\LoginMiddleware::class
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        GuardInterceptor::class,
        FilterInterceptor::class,
        GridInterceptor::class,
    ];
}
```

### 命名空间 (Namespace)

您可以使用 `NAMESPACE` 常量定义当前的 keeper 命名空间。 默认情况下是 `keeper`。

### 前缀 (Prefix)

您可以使用 `PREFIX` 常量定义当前路由前缀。 默认情况下是 `keeper/`。

### 默认控制器 (Default Controller)

您可以使用 `DEFAULT_CONTROLLER` 常量定义默认路由控制器。 默认的 action 只能通过配置文件声明 (或者，如果已设置，通过 `defaultAction` 注解属性或 `index` 方法)。

### 配置名称 (Config name)

您可以使用 `CONFIG_NAME` 常量定义当前 keeper 命名空间的配置名称。 如果省略，将使用 `NAMESPACE` 常量的值。

### 中间件 (Middlewares)

您可以在 `MIDDLEWARE` 常量中列出自定义中间件。 额外的中间件可以在路由中单独应用到每个路由，两个集合都将被应用。

### 拦截器 (Interceptors)

您可以在 `INTERCEPTORS` 常量中列出自定义拦截器。

## 配置 (Config)

Keeper 配置可以在配置文件中完全声明。
> 配置文件应该存储在应用的 `config` 目录中。

```php
return [
   'routePrefix'   => '',
   'routeDefaults' => ['controller' => '', 'action' => ''],
   'loginView'     => 'keeper:login',
   'middleware'    => [],
   'modules'       => [],
   'interceptors'  => [],
];
```

前缀、默认控制器、中间件列表、模块的引导程序和拦截器从常量中作为配置默认值使用。
