# HTTP — 路由

## 安装

默认情况下，路由组件已安装在 Spiral 中，但如果您想在自定义构建中使用它，请使用 composer 安装路由组件。

```terminal
composer require spiral/router
```

## 基于属性的路由

最简单的方式是直接在您的控制器方法中使用属性来定义路由。这可以是一种设置路由的便捷方式，并且具有一些关键优势。首先，它可以使您的代码更简洁易读。它还可以帮助改善应用程序中的关注点分离，并使其他开发人员更容易发现和理解可用的路由。因此，如果您正在寻找一种更有组织和可维护的方式来设置路由，基于属性的路由可能值得考虑！

> **注意**
> 我们使用 Tokenizer 组件进行 [静态分析](../advanced/tokenizer.md)，以识别路由属性。 要指定应该搜索控制器的目录，请参阅 Tokenizer 文档中关于 [自定义搜索目录.](../advanced/tokenizer.md#customizing-search-directories) 的部分。

> **警告**
> 请小心，因为 Tokenizer 将忽略控制器中包含 `include` 或 `require` 语句的任何文件。

只需在您的应用程序中激活 Bootloader `Spiral\Router\Bootloader\AnnotatedRoutesBootloader`：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Router\Bootloader\AnnotatedRoutesBootloader::class,
        // ...
    ];
}
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 部分了解更多关于 Bootloaders 的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Router\Bootloader\AnnotatedRoutesBootloader::class,
    // ...
];
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 部分了解更多关于 Bootloaders 的信息。
:::

::::

就这样！现在您可以使用该组件了。

### 定义路由

`Spiral\Router\Annotation\Route` 属性使您可以通过设置各种属性在控制器方法中建立路由：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Router\Annotation\Route;

class HomeController
{
    #[Route(route: '/', name: 'index', methods: 'GET')] 
    public function index(): string
    {
        return 'hello world';
    }
}
```

以下是每个属性的简要描述：

| 属性         | 类型         | 描述                                                                                                                                                 |
|------------|--------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| route      | string       | 路由模式，定义了路由将匹配的 URL 模式。[Router](/http/routing.md)。**必需**                                        |
| name       | string       | 路由名称。**可选**                                                                                                                                    |
| methods    | array/string | 路由将匹配的 HTTP 方法（例如 `GET`，`POST`，`PUT` 等）。默认为所有方法。                                                      |
| defaults   | array        | 路由参数的默认值数组。                                                                                                            |
| group      | string       | 路由所属的路由组。默认为 `default`。                                                                                         |
| middleware | array        | 路由特定的中间件类名。                                                                                                                      |
| priority   | int          | 在路由列表中的位置。数字越小，路由越重要。有助于解决一个请求匹配两条路由的情况。默认为 0。 |

使用这些属性，您可以以简洁和有组织的方式定义路由的详细信息。

### 路由名称

为您的路由指定一个名称通常是一个好主意，因为它可以在应用程序的其他地方更容易地引用它们。但是，如果您没有指定名称，Spiral 将自动为您生成一个名称，如果您不需要按名称引用该路由，这会很方便。

框架将根据路由模式和它匹配的 HTTP 方法自动为您生成一个默认名称。

```php
#Route(route: '/api/news', methods: ["POST", "PATCH"]) // => post,patch:/api/news
````

## 路由定义

:::: tabs

::: tab 路由配置器

Spiral 为开发人员提供了一种方便且有组织的方式，可以使用 `App\Application\Bootloader\RoutesBootloader` 类的 `defineRoutes` 方法定义路由。此方法提供一个 `Spiral\Router\Loader\Configurator\RoutingConfigurator` 实例，它提供了一系列用于定义和配置路由的方法。

> **警告**
> `App\Application\Bootloader\RoutesBootloader` 必须位于 Bootloader 列表的 `LOAD` 部分。

使用 `RoutingConfigurator`，开发人员可以轻松地将各种设置（例如中间件、前缀和 HTTP 方法）应用于他们的路由。它允许您创建并自动注册路由。

**以下是使用 `RoutingConfigurator` 定义路由的示例：**

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...
 
    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->add(name: 'news.show', pattern: '/news/<id:int>')
            ->group('web')
            ->methods(methods: ['GET'])
            ->action(NewsController::class, 'show');
            
        ...
    }
}
```

:::

::: tab 导入路由

`RoutingConfigurator` 具有一个方便的功能，允许您从特定文件中导入路由。

**以下是如何操作的示例：**

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

namespace App\Application\Bootloader;

use Spiral\Boot\DirectoriesInterface;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

final class RoutesBootloader extends BaseRoutesBootloader
{
    public function __construct(
        private readonly DirectoriesInterface $dirs
    ) {
    }
    
    // ...

    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->import($this->dirs->get('app') . '/routes/web.php')
            ->group('web');
            
        $routes->import($this->dirs->get('app') . '/routes/api.php')
            ->prefix('/api');
            ->group('api');
    }
}
```

打开 `web.php` 文件，使用 `add` 方法添加新路由。

```php app/routes/web.php
use App\Controller\HomeController;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;

return function (RoutingConfigurator $routes): void {
    $routes->add(name: 'news.show', pattern: '/news/<id:int>')
        ->methods(methods: ['GET'])
        ->action(NewsController::class, 'show');
};
```

:::

::::

## 路由配置器

路由配置器提供了多种用于定义和配置路由的方法。

### 路由目标

设置路由的目标控制器

:::: tabs

::: tab 命名空间

在某些情况下，可能需要路由到驻留在同一命名空间内的一组控制器。为此，请使用 `namespaced` 方法。 此目标需要指定路由参数 `<controller>` 和 `<action>`（除非强制使用默认值）。

```php
$routes->add(name: 'admin', pattern: '/admin/<controller>/<action>')
    ->namespaced(
        namespace: 'App\Controllers\Admin', // required
    );
```

**示例请求**

```bash
GET /admin/users/index
```

在这种情况下，`<controller>` 参数对应于 `users`，`<action>` 参数对应于 `index`。因此，该请求将被路由到位于 `App\Controllers\Admin` 命名空间中的 `UsersController` 类的 `index` 动作。

默认情况下，该方法假定控制器具有 `Controller` 后缀。但是，如果您希望更改默认后缀，则可以使用后缀参数来实现。

例如，如果您的控制器具有 `Handler` 后缀而不是 `Controller`，您可以如下设置命名空间的路由目标：

```php
$routes->add(name: 'admin', pattern: '/admin/<controller>/<action>')
    ->namespaced(
        namespace: 'App\Controllers\Admin',
        postfix: 'Handler'
    );
```

:::

::: tab 控制器

控制器方法使您可以一次性将路由定向到特定控制器中的所有操作。 此目标需要定义 `<action>` 参数（除非强制使用默认值）。

```php
$routes->add(name: 'user', pattern: '/user/<action>')
    ->controller(controller: UserController::class);
```

**示例请求**

```bash
GET /user/list
```

这里，`<action>` 参数是 `list`。 该请求将被路由到 `UserController` 类的 `list` 动作。

默认情况下，如果路由模式不包含 `<action>` 段，则 `index` 动作将用作指定控制器的默认动作。

您还可以使用 default 方法为控制器路由目标指定不同的默认动作。 例如，如果您想将默认动作设置为 `list` 而不是 `index`，则可以执行以下操作：

```php
$routes->add(name: 'user', pattern: '/user/list')
    ->controller(controller: UserController::class)
    ->defaults(['action' => 'list'])
```

:::

::: tab 动作

要将路由指定到特定的控制器动作，请使用 action 方法指定路由处理程序。

```php
$routes->add(name: 'post.show', pattern: '/post/<id:int>')
    ->action(PostController::class, 'show');
```

**示例请求**

```bash
GET /post/42
```

该请求将被路由到 `PostController` 类的 show 动作，并将值 `42` 作为参数传递给 `show` 动作。

:::

::: tab 可调用

它允许您定义直接指向可调用函数、闭包或包含一个对象和一个方法名称的数组的路由。当您希望使用少量逻辑处理特定路由而无需创建单独的控制器时，这可能是一个有用的选项。

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

$routes->add(name: 'greeting', pattern: '/greeting')
    ->callable(function (ServerRequestInterface $r, ResponseInterface $rsp) {
        return 'Hello, world!';
    });
```

```


:::

::::

### 默认路由参数

```php
$routes
    ->add(name: 'html', pattern: '/<action>.html')
    ->defaults(['action' => 'default'])
    ->...;
```

### 自定义域核心

```php
$core = new \Spiral\Core\InterceptableCore(...);
$core->addInterceptor(...);

$routes
    ->add(name: 'html', pattern: '/<action>.html')
    ->core($core)
    ->...;
```

### 路由前缀

```php
$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->prefix('/api')
    ->...;
```

### HTTP 方法（动词）

```php
$routes->add(name: 'html', pattern: '/<action>.html')
    ->methods('GET')
    ->...;

// 或

$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->methods(['GET', 'POST'])
    ->...;
```

### 添加中间件

```php
$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->middleware(LocaleSelector::class)
    ->...;
```

### 后备路由

在某些情况下，用户可能会请求不存在的页面，或者应用程序可能会收到与任何预定义路由都不匹配的 URL。为了优雅地处理这些场景，拥有一个可以捕获这些不匹配的请求并向用户返回有意义响应的后备路由至关重要。

提供的代码片段演示了 Spiral 中的后备路由示例：

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

$routes->default('/<path:.*>')
    ->callable(function (ServerRequestInterface $r, ResponseInterface $rsp) {
        return 'Page not found!';
    });
```

> **注意**
> 您不仅可以使用可调用的内容，还可以使用任何其他路由目标：控制器、动作、命名空间等。

## 路由组配置器

您可以轻松地将应用程序的路由组织成逻辑组，并使用几行代码将中间件、前缀和其他设置应用于组内的所有路由。这使得维护和扩展应用程序变得容易，并且在处理大型、复杂的项目时可以节省大量时间和精力。

您可以通过 `App\Application\Bootloader\RoutesBootloader` 设置路由组。此 Bootloader 包含 `configureRouteGroups` 方法，该方法在参数中包含 `Spiral\Router\GroupRegistry`。

以下是设置路由组的简单示例：

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Router\GroupRegistry;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function configureRouteGroups(GroupRegistry $groups): void
    {
        $groups->getGroup('api')
            ->setNamePrefix('api.')
            ->setPrefix('/api');
            
        $groups->getGroup('web')
            ->addMiddleware(MyMiddelware::class);
            ->setPrefix('/api');
    }
}
```

以下是向组分配路由的示例：

:::: tabs

::: tab 路由

您可以通过指定组的名称来将路由分配给组。

```php app/src/Application/Bootloader/RoutesBootloader.php
$routes->add(name: 'news', pattern: '/news/<id:int>')
    ->action(NewsController::class, 'show')
    ->group('auth');
    ->methods('GET');
```

:::

::: tab 属性

您可以通过在 group 属性中指定组的名称来将路由分配给组。

> **警告**
> 确保在 `AnnotatedRoutesBootloader` 之后注册一个 bootloader。

```php app/src/Endpoint/Web/HomeController.php
#[Route(route: '/', name: 'index', methods: 'GET', group: 'api')]  
public function index(): ResponseInterface
{
    // ...    
}
```

在此示例中，路由将被分配到 `api` 组，并将带有 `/api/v1` 前缀，并应用 `SomeMiddleware` 中间件。这允许您以方便且有组织的方式轻松地将共享规则和中间件应用于一组路由。

:::

::: tab 全局
您还可以为所有路由设置默认组：

```php app/src/Application/Bootloader/RoutesBootloader.php
protected function configureRouteGroups(GroupRegistry $groups): void
{
    // ...
    
    $groups->getGroup('api')
        ->setNamePrefix('api.')
        ->setPrefix('/api');
    
    $groups->setDefaultGroup('api');
}
```

:::

::::

## 路由器

您可以使用 `Spiral\Router\RouterInterface` 创建新的路由。

我们可以从一个简单的 `/` 处理程序开始：

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'home',                    // 路由名称 
            new Route(
                '/',                   // 模式
                fn () => 'hello world' // 处理程序
            )
        );
    }
}
```

> **注意**
> Route 类可以接受类型为 `Psr\Http\Server\RequestHandlerInterface`、闭包、可调用类或 `Spiral\Router\TargetInterface` 的处理程序。如果您希望按需构建，只需传递一个类或一个绑定名称而不是一个真实的对象。

### 闭包处理程序

可以将 `closure` 作为路由处理程序传递。在这种情况下，我们的函数将接收两个参数：`Psr\Http\Message\ServerRequestInterface` 和 `Psr\Http\Message\ResponseInterface`。

```php
$router->setRoute('home', new Route(
    '/<name>',
    function (ServerRequestInterface $request, ResponseInterface $response): ResponseInterface {
        $response->getBody()->write('hello world');

        return $response;
    }
));
```

### 路由模式和参数

您可以使用路由模式指定任意数量的必需和可选参数。 这些参数稍后将通过 `ServerRequestInterface` 属性 `matches` 传递给路由处理程序。

> **注意**
> 在请求过滤器中使用 `attribute:matches.id` 访问这些值。

使用 `<parameter_name:pattern>` 形式定义路由参数，其中模式是与 regexp 兼容的表达式。 您可以省略模式，只使用 `<parameter_name>`，在这种情况下，该参数将匹配 `[^\/]+`。

我们可以添加一个简单的参数 `name`：

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute('home', new Route(
            '/<name>',
            function (ServerRequestInterface $request, ResponseInterface $response): array {
                return $request->getAttribute('route')->getMatches(); // 返回 JSON ['name' => '']
            }
        ));
    }
}
```

使用 `[]` 使路由的一部分（包括参数）成为可选的，例如：

```php
$router->setRoute('home', new Route(
    '/[<name>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> **注意**
> 此路由将匹配 `/`，名称参数将为 `null`。

您可以指定任意数量的参数，并使其中一些参数成为可选的。 例如，我们可以匹配类似 `/group/user` 的 URL，其中 `user` 是可选的：

```php
$router->setRoute('home', new Route(
    '/<group>[/<user>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

您可以使用第三个路由参数指定默认参数值：

```php
$router->setRoute('home', new Route(
    '/<group>[/<user>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    },
    [
        'user' => 'default'
    ]
));
```

使用 `<parameter:pattern>` 指定参数模式：

```php
$router->setRoute('home', new Route(
    '/user/<id:\d+>',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> **注意**
> 此路由将仅匹配具有数字 `id` 的 URL，但这并不意味着路由属性 `id` 将包含整数值。在这种情况下，该属性将始终包含一个字符串值。

#### 路由预定义选项

您还可以指定多个预定义选项：

```php
$router->setRoute('home', new Route(
    '/do/<action:login|logout>',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> **注意**
> 此路由将仅匹配 `/do/login` 和 `/do/logout`。

#### 匹配主机

要匹配域名或子域名，请使用 `//` 作为模式的前缀：

```php
$router->setRoute('home', new Route(
    '//<host>/',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

要匹配子域：

```php
$router->setRoute('home', new Route(
    '//<sub>.domain.com/',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

您可以组合主机和路径匹配：

```php
$router->setRoute('home', new Route(
    '//<sub>.domain.com/[<action>]',
    function (ServerRequestInterface $request, ResponseInterface $response): array {
        return $request->getAttribute('route')->getMatches();
    }
));
```

#### 不可变性

所有路由对象都具有设计上的不可变性，您无法在创建后更改其状态，而只能使用新值进行复制。 要在构造函数之外设置默认路由参数：

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $route = new Route('/[<action>]', function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('home', $route->withDefaults([
            'action' => 'default'
        ]));
    }
}
```

#### 动词

使用 `withVerbs` 方法仅匹配具有特定 HTTP 动词的路由：

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $route = new Route('/[<action>]', function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('get.route',
            $route->withVerbs('GET')->withDefaults(['action' => 'GET'])
        );

        $router->setRoute(
            'post.route',
            $route->withVerbs('POST', 'PUT')->withDefaults(['action' => 'POST'])
        );
    }
}
```

#### 中间件

要关联特定于路由的中间件，请使用 `withMiddleware`。 您可以通过请求对象的 `route` 属性访问路由参数：

```php
namespace App\Bootloader;

use App\Middleware\ParamWatcher;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $route = new Route('/<param>', function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('home', $route->withMiddleware(
            ParamWatcher::class
        ));
    }
}
```

其中 `ParamWatcher` 是：

```php
namespace App\Middleware;

use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Http\Exception\ClientException\UnauthorizedException;
use Spiral\Router\RouteInterface;

class ParamWatcher implements MiddlewareInterface
{
    public function process(Request $request, RequestHandlerInterface $handler): Response
    {
        /** @var RouteInterface $route */
        $route = $request->getAttribute('route');

        if ($route->getMatches()['param'] === 'forbidden') {
           throw new UnauthorizedException();
        }

        return $handler->handle($request);
    }
}
```

此路由将在 `/forbidden` 上触发未经授权的异常。

> **注意**
> 您可以添加任意数量的中间件。

### 多条路由

路由器将按注册顺序匹配所有路由。 确保避免上一个路由匹配后续路由条件的情况。

```php
$router->setRoute(
    'home',
    new Route('/<param>',
        function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        }
    )
);

// 此路由永远不会触发
$router->setRoute(
    'hello',
    new Route('/hello',
        function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        }
    )
);
```

### 默认路由

Spiral Router 允许您指定默认/后备路由。 在每个其他路由之后，将始终调用此路由，并检查其模式是否匹配。

例如，如果您以以下方式设置默认路由，则无需为每个控制器和动作定义路由：

```php
$router->setRoute(
    'home',
    new Route('/<param>',
        function (ServerRequestInterface $request, ResponseInterface $response): array {
            return $request->getAttribute('route')->getMatches();
        }
    )
);

$router->setDefault(new Route('/', fn (): string => 'default'));
```

请参阅以下内容，了解如何使用默认路由快速搭建应用程序路径。

### 路由目标（控制器和动作）

使用路由器的最有效方法是将路由指向控制器及其操作。为了演示所有功能，我们需要在 `App\Controller` 命名空间中添加多个控制器：

```php
namespace App\Controller;

class HomeController
{
    public function index(): string
    {
        return 'index';
    }

    public function other(): string
    {
        return 'other';
    }

    public function user(int $id): string
    {
        return "hello {$id}";
    }
}
```

使用脚手架 `php ./app.php create:controller demo -a test` 创建第二个控制器：

```php
namespace App\Controller;

class DemoController
{
    public function test(): string
    {
        return 'demo test';
    }
}
```

#### 路由到动作

要将路由指向控制器操作，请将路由处理程序指定为 `Spiral\Router\Target\Action`：

```php
namespace App\Bootloader;

use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'index',
            new Route('/index', new Action(HomeController::class, 'index'))
        );
    }
}
```

您可以将此目标与必需或可选的参数结合使用。该参数将作为方法注入提供给所需的目标：

```php
$router->setRoute(
    'user',
    new Route('/user/<id:\d+>', new Action(HomeController::class, 'user'))
);
```

#### 通配符动作

我们可以同时将一条路由指向多个控制器动作。为此，我们必须在路由模式中定义参数 `<action>`。由于其中一种方法需要 `<id>` 参数，我们可以使其成为可选的：

```php
$router->setRoute(
    'home',
    new Route('/<action>[/<id>]', new Action(HomeController::class, ['index', 'user']))
);
```

> **注意**
> 此路由将同时匹配 `/index` 和 `/user/1` 路径。

在后台，该路由将被编译成一个了解操作约束的表达式 `/^(?P<action>index|user)(?:\/(?P<id>[^\/]+))?$/iu`。 这种方法不仅允许您提高性能，还可以重复使用同一模式，其中包含不同的操作集。

```php
// 匹配 "/index"
$router->setRoute(
    'home',
    new Route('/<action>', new Action(HomeController::class, 'index'))
);

// 匹配 "/other"
$router->setRoute(
    'home',
    new Route('/<action>', new Action(HomeController::class, 'other'))
);

// 匹配 "/test"
$router->setRoute(
    'demo',
    new Route('/<action>', new Action(DemoController::class, 'test'))
);
```

#### 路由到控制器

您可以使用 `Spiral\Router\Target\Controller` 将路由一次指向所有控制器操作。 此目标需要定义 `<action>` 参数（除非强制使用默认值）。

```php
namespace App\Bootloader;

use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Controller;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'home',
            new Route('/home/<action>[/<id>]', new Controller(HomeController::class))
        );
    }
}
```

> **注意**
> 路由匹配 `/home/index`、`/home/other` 和 `/home/user/1`。

将此目标与默认值结合使用以使您的 URL 更短。

```php
$router->setRoute(
    'home',
    (new Route('/home[/<action>[/<id>]]', new Controller(HomeController::class)))
        ->withDefaults(['action' => 'index'])
);
```

> **注意**
> 此路由将匹配 `/home`，`action=index`。 请注意，您必须将可选路径段 `[]` 扩展到路由模式的末尾。

#### 路由到命名空间

在某些情况下，您可能希望路由到位于同一命名空间中的一组控制器。 为此，请使用目标 `Spiral\Router\Target\Namespaced`。 此目标将需要路由参数 `<controller>` 和 `<action>`（除非强制使用默认值）。

您可以指定目标命名空间和控制器类后缀：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Namespaced;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute('app', new Route(
            '/<controller>/<action>',
            new Namespaced('App\Controller', 'Controller')
        ));
    }
}
```

> **注意**
> 此路由将匹配 `/home/index`、`/home/other` 和 `/demo/test`。

您可以将所有参数设为可选并设置默认值：

```php
$router->setRoute('app',
    (new Route(
        '[/<controller>[/<action>]]',
        new Namespaced('App\Controller', 'Controller')
    ))->withDefaults([
        'controller' => 'home',
        'action'     => 'index'
    ])
);
```

> **注意**
> 此路由将匹配 `/`（home->index）、`/home`（home->index）、`/home/index`、`/home/other` 和 `/demo/test`。`/demo` 将触发未找到的错误，因为 `DemoController` 未定义方法 `index`。

默认的 Web 应用程序包 [将此路由设置为默认值](https://github.com/spiral/app/blob/2.x/app/src/Bootloader/RoutesBootloader.php#L42)。您无需为添加到 `App\Controller` 的任何控制器创建路由，只需使用 `/controller/action` URL 访问所需的方法。如果未指定操作，则默认使用 `index`。路由将仅指向公共方法。

> **注意**
> 开发结束后，您可以关闭默认路由。

#### 路由到控制器组

另一种方法是手动指定控制器名称，而无需通用命名空间。 使用目标 `Spiral\Router\Target\Group`。 目标需要定义 `<controller>` 和 `<action>` 参数（除非强制使用默认值）。

```php
namespace App\Bootloader;

use App\Controller\DemoController;
use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Group;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute('app', new Route('/<controller>/<action>', new Group([
            'home' => HomeController::class,
            'demo' => DemoController::class
        ])));
    }
}
```

> **注意**
> 当您想将多个模块组装到一个路径下（即管理面板）时，这种方法很有用。

## 命名路由模式

如果您希望路由参数始终受给定的正则表达式约束，则可以使用命名模式。您应该通过 Bootloader 中的 `Spiral\Router\Registry\RoutePatternRegistryInterface` 定义这些模式：

```php
use Spiral\Router\Registry\RoutePatternRegistryInterface;

class AppBootloader extends Bootloader
{
   public function boot(RoutePatternRegistryInterface $patternRegistry): void
   {
      $patternRegistry->register(
          'uuid', 
          '[0-9a-fA-F]{8}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{12}'
      );
      $patternRegistry->register(
          'names', 
          new InArrayPattern(['tom', 'jerry'])
      );
   }
}
```

定义模式后，它将自动应用于使用该参数名称的所有路由：

#### 示例：

```php
#Route(uri: 'blog/post/<post:uuid>')  // <===== 将匹配: /blog/post/f403554a-e70f-479a-969b-3edc047912a3
public function show(string $post)
{ 
    \var_dump($post); // f403554a-e70f-479a-969b-3edc047912a3
}
```

```php
#Route(uri: 'user/<name:names>') // <===== 将匹配: /user/