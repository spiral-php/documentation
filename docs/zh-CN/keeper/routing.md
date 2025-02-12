# Keeper — 路由

Keeper 的路由可以通过全局的 `Spiral\Router\RouterInterface` 访问。您可以在路由中直接注册它们，或者通过 Keeper 的引导程序和注解注册。后一种选择允许您将所有路由隔离在给定的命名空间中。

## 权限

默认情况下，GuardInterceptor 被添加到 KeeperCore。使用 `@GuardNamespace` 和 `@Guarded` 来创建权限。如果缺少 `@Guarded` 注解，KeeperCore 将使用 `namespace.controller.method` 权限作为后备来保护一个方法。

## 通过引导程序或 Router 注册路由

新的路由应该在 `parent::boot()` 调用之后声明。控制器必须在路由声明之前声明：

```php
use Spiral\Boot\BootloadManager;
use Spiral\Keeper\Bootloader;
use Spiral\Router\RouterInterface;

class AdminBootloader extends Bootloader\KeeperBootloader
{
    protected const NAMESPACE = 'admin';
    protected const PREFIX    = '/admin';

    public function boot(BootloadManager $bootloadManager, RouterInterface $appRouter): void
    {
        parent::boot($bootloadManager, $appRouter);

        // 控制器通过别名使用
        $this->addController('user', 'App\Admin\Controller\User');
        $this->addRoute('/users/new', 'user', 'new', ['GET'], 'createUser');
    }
}
```

将创建以下路由（参见 `php app.php route:list` 输出）：

```
+-------------------+--------+------------------+---------------------------------- +
| Name:             | Verbs: | Pattern:         | Target:                           |
+-------------------+--------+------------------+---------------------------------- +
| admin[createUser] | POST   | /admin/users/new | App\Admin\Controller\User->create |
| admin[user.new]   | GET    | /admin/users/new | user->new                         |
+-------------------+--------+------------------+---------------------------------- +
```

> **注意**
> 路由将被生成的名称（如 `controller.method`）复制。

## 通过注解注册路由

属性是一种更方便的方式，因为使用注解的路由允许您使用 [sitemaps](../keeper/sitemap.md) - 另一个用于构建菜单导航和面包屑的强大子模块。

`\Spiral\Keeper\Annotation\Action` 和 `\Spiral\Keeper\Annotation\Controller` 属性可用。它们应该一起使用。

`Controller` 定义：

- 当前 `namespace` (可选，默认为 `keeper`)
- 内部 `name`/别名 (必填)
- `prefix` (可选)，用于控制器中的所有 action
- 默认 action `defaultAction` (可选)

`Action` 的工作方式与框架注解路由中的基本 `Route` 注解非常相似。它定义：

- `route` 模式 (必填)
- `name` (可选，默认为 `controller.action`)
- `methods` (可选)
- `defaults` (可选)
- `group` (目前未使用)
- `middleware` (可选)

示例：

```php
use Spiral\Keeper\Annotation\Action;
use Spiral\Keeper\Annotation\Controller;
use Spiral\Views\ViewsInterface;

#[Controller(namespace: "admin", name: "user", prefix: "/users")]
class User
{

     #[Action(route: "/create", name: "createUser", methods: "GET")]
    public function create(ViewsInterface $views): string
    {
        return $views->render('admin:users/create');
    }
}
```

将创建以下路由（参见 `php app.php route:list` 输出）：

```
+--------------------+--------+---------------------+--------------+
| Name:              | Verbs: | Pattern:            | Target:      |
+--------------------+--------+---------------------+--------------+
| admin[createUser]  | GET    | /admin/users/create | user->create |
| admin[user.create] | GET    | /admin/users/create | user->create |
+--------------------+--------+---------------------+--------------+
```

> **注意**
> 路由将被生成的名称（如 `controller.method`）复制。

## 命名空间

您可以通过其名称直接调用路由，或使用 `RouteBuilder`。

```php
/**
 * @var \Spiral\Router\RouterInterface     $router 
 * @var \Spiral\Keeper\Helper\RouteBuilder $routeBuilder
 */
 
$router->uri('admin[user.create]');
$routeBuilder->uri('admin', 'user.create');
```

输出将是相同的: `/admin/users/new`

> **注意**
> `RouteBuilder` 负责命名空间隔离模式。

## 默认值

要启用默认控制器路由，它应该显式地添加到配置中。通过 `KeeperBootloader::DEFAULT_CONTROLLER` 值或通过配置文件：

```php
return [
     'routeDefaults' => ['controller' => 'App\Admin\Controller'],
];
```

也可以在配置中定义默认的控制器 action：

```php
return [
     'routeDefaults' => ['controller' => 'App\Admin\Controller', 'action' => 'list'],
];
```

或者通过 `@Controller` 注解中的 `defaultAction` 属性。

> **注意**
> 对于默认控制器，如果未在默认控制器注解中提供 `defaultAction`，将使用 `index` 方法作为后备。
