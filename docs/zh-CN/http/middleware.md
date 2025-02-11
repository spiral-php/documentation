# HTTP - 中间件

Spiral 框架使用 [与 PSR-15 兼容](https://www.php-fig.org/psr/psr-15/) 的 HTTP 中间件。

中间件负责处理与请求和响应相关的功能，例如身份验证、缓存或日志记录。它可以在将请求和响应传递给路由之前修改它们，但它不能决定应该由应用程序处理哪些路由。这是路由器的职责，中间件不应尝试绕过或覆盖路由器的决策。

[拦截器](./interceptors.md) 非常适合处理与应用程序路由器相关的功能。它们在请求传递给应用程序后执行，并且可以更多地访问应用程序的内部状态，包括路由器。

## 创建中间件

`Psr\Http\Server\MiddlewareInterface` 是 PSR-15 提供的一个标准接口，用于在 PHP 中创建中间件。要创建您自己的中间件，您需要实现此接口并定义它所需的方法。

```php app/src/Endpoint/Web/Middleware/MyMiddleware.php
namespace App\Endpoint\Web\Middleware;

use Psr\Http\Server\MiddlewareInterface;

class MyMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        return $handler->handle($request)->withAddedHeader('My-Header', 'my-value');
    }
}
```

> **注意**
> 访问 https://github.com/middlewares/psr15-middlewares 可以找到许多公开维护的中间件。

Spiral 提供了几种设置中间件的方法，允许开发人员选择最适合他们需求的方法。

### 全局中间件

这些中间件应用于所有路由和请求。它们通常用于应该应用于所有请求的功能，例如身份验证或日志记录。

您可以在 `RoutesBootloader` 中激活全局中间件：

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use App\Endpoint\Web\Middleware\LocaleSelector;
use Spiral\Auth\Middleware\AuthTransportMiddleware;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Cookies\Middleware\CookiesMiddleware;
use Spiral\Core\Container\Autowire;
use Spiral\Csrf\Middleware\CsrfMiddleware;
use Spiral\Debug\StateCollector\HttpCollector;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Http\Middleware\JsonPayloadMiddleware;
use Spiral\Session\Middleware\SessionMiddleware;
use App\Endpoint\Web\Middleware\MyMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            LocaleSelector::class,
            ErrorHandlerMiddleware::class,
            JsonPayloadMiddleware::class,
            HttpCollector::class,
            MyMiddleware::class,
        ];
    }
    
    // ...
}
```

或者，您可以使用 `Spiral\Bootloader\Http\HttpBootloader` 为每个用户请求激活全局中间件。您只能在应用程序引导程序中设置此值。

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\HttpBootloader;
use Spiral\Core\Container\Autowire;
use Psr\Container\ContainerInterface;
use App\Endpoint\Web\Middleware\MyMiddleware;

class AppBootloader extends Bootloader
{
    public function boot(HttpBootloader $http, ContainerInterface $container): void
    {
        // automatically resolved by Container
        $http->addMiddleware(MyMiddleware::class);
        
        // automatically resolved by Container
        $container->bind('my:middleware', fn() => new MyMiddleware);
        $http->addMiddleware('my:middleware');
        
        // Autowire allows creating an object with dependency resolving from the container
        // and passing some parameters manually
        $http->addMiddleware(new Autowire(MyMiddleware::class, ['someParameter' => 'value']));
    }
}
```

中间件对象将被按需实例化。

或者，您可以在配置文件 `app/config/http.php` 中配置中间件：

```php app/config/http.php
use App\Endpoint\Web\Middleware\MyMiddleware;
use Spiral\Core\Container\Autowire;

return [
    // ...
    'middleware' => [
        // via fully qualified class name
        MyMiddleware::class,
        
        'my:middleware',
        
        // via Autowire 
        new Autowire(MyMiddleware::class, ['someParameter' => 'value']),
        
        // or manual instantiating object
        new MyMiddleware(),
    ],
];
```

## 中间件组

分组的中间件将仅应用于相应组内的路由。这些组在应用程序的容器中注册为名为 `middleware:{group}` 的管道，因此您可以在任何路由上使用它们。

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use App\Middleware\LocaleSelector;
use Spiral\Auth\Middleware\AuthTransportMiddleware;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Cookies\Middleware\CookiesMiddleware;
use Spiral\Core\Container\Autowire;
use Spiral\Csrf\Middleware\CsrfMiddleware;
use Spiral\Debug\StateCollector\HttpCollector;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Http\Middleware\JsonPayloadMiddleware;
use Spiral\Session\Middleware\SessionMiddleware;
use App\Endpoint\Web\Middleware\MyMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                CookiesMiddleware::class,
                SessionMiddleware::class,
                CsrfMiddleware::class,
                MyMiddleware::class,
                // new Autowire(AuthTransportMiddleware::class, ['transportName' => 'cookie'])
            ],
            'api' => [
                // new Autowire(AuthTransportMiddleware::class, ['transportName' => 'header'])
            ],
        ];
    }
    
    // ...
}
```

## 特定于路由的中间件

这些中间件应用于特定路由。这允许开发人员将中间件应用于单个路由，例如特定的 API 终点。

:::: tabs

::: tab 路由配置器

要将中间件添加到路由对象，请使用 `middleware` 方法：

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;
use App\Endpoint\Web\Middleware\MyMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...
 
    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->add(name: 'news.show', pattern: '/news/<id:int>')
            ->middleware(['middleware:web', MyMiddleware::class]);
        ...
    }
}
```

:::

::: tab 路由属性

要使用路由属性 `Spiral\Router\Annotation\Route` 添加中间件，请使用 `middleware` 属性：

```php app/src/Application/Controller/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Router\Annotation\Route;
use App\Endpoint\Web\Middleware\MyMiddleware;

class HomeController
{
    #[Route(route: '/', name: 'index', methods: 'GET', middleware: [MyMiddleware::class])] 
    public function index(): string
    {
        // ...
    }
}
```

:::

::: tab 路由器

要将中间件添加到路由对象，请使用 `withMiddleware` 方法。确保使用新创建的路由实例，这是一个示例：

```php app/src/Application/Bootloader/AppBootloader.php
use App\Controller\HomeController;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;
use App\Endpoint\Web\Middleware\MyMiddleware;

// ...

public function boot(RouterInterface $router): void
{
    $route = new Route('/index', new Action(HomeController::class, 'index'));
    $route = $route->withMiddleware(MyMiddleware::class);

    $router->addRoute('index', $route);
}
```

:::

::::

## 与 IoC 范围结合

Spiral 框架允许开发人员将中间件与 [IoC 范围](../framework/scopes.md) 结合起来，以创建特定于请求的应用程序上下文。

这允许开发人员为当前请求设置一个上下文，应用程序的其他部分可以访问该上下文。这对于任务（例如日志记录或数据访问）非常有用，其中请求的上下文很重要。

```php
class UserContext
{
    public int $id;
    public string $name;

    public function __construct(int $id, string $name)
    {
        $this->id = $id;
        $this->name = $name;
    }
}
```

通过在您的中间件中使用 `Spiral\Core\ScopeInterface`，您可以设置一个特定于当前请求的应用程序范围。

```php app/src/Endpoint/Web/Middleware/MyMiddleware.php
namespace App\Endpoint\Web\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Core\ScopeInterface;

class MyMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ScopeInterface $scope
    ) {
    }

    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        return $this->scope->runScope([
            UserContext::class => new UserContext(123, 'test')
        ], function () use ($handler, $request) {
            return $handler->handle($request);
        });
    }
}
```

一旦在您的中间件中设置了特定于请求的上下文，您就可以从容器中请求它，或者通过在您的控制器中进行方法注入。

```php
public function index(UserContext $ctx): void
{
    dump($ctx);
}
```

> **注意**
> 同样重要的是要注意，在中间件中设置的范围仅在请求期间有效，并且不会影响其他请求。 这允许您为每个请求维护上下文的隔离性和完整性。

## 非直接范围配置

您可以使用已存在的请求范围来携带用户值。 创建一个引导程序，提供对上下文特定值的访问方法：

```php app/src/Endpoint/Web/Middleware/MyMiddleware.php
namespace App\Endpoint\Web\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class MyMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        return $handler->handle($request->withAttribute('userContext', new UserContext(123, 'test')));
    }
}
```

从容器中访问此值：

```php app/src/Application/Bootloader/UserContextBootloader.php
namespace App\Application\Bootloader;

use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Exception\ScopeException;

class UserContextBootloader extends Bootloader 
{
    protected const BINDINGS = [
        UserContext::class => [self::class, 'userContext']
    ];
    
    private function userContext(ServerRequestInterface $request): UserContext
    {
        $userContext = $request->getAttribute('userContext', null);
        if ($userContext === null) {
            throw new ScopeException('Unable to resolve UserContext, invalid request scope');
        }
        
        return $userContext;
    }
}
```

## 可用的中间件

HTTP 扩展包括多个中间件，您可能需要在您的项目中激活它们：

| 引导程序                                    | 中间件                                                            |
|-----------------------------------------------|-----------------------------------------------------------------------|
| Spiral\Bootloader\Http\ErrorHandlerBootloader | 在非调试模式下隐藏异常并呈现 HTTP 错误页面。                            |
| Spiral\Bootloader\Http\JsonPayloadsBootloader | 解析 `application/json` 请求的主体。                                |
| Spiral\Bootloader\Http\PaginationBootloader   | 使用请求查询参数自动配置分页器。                                |
| Spiral\Bootloader\Http\DiactorosBootloader    | 使用 Zend/Diactoros 作为 PSR-7 实现（旧版）。                          |

## 事件

| 事件                                  | 描述                                              |
|----------------------------------------|----------------------------------------------------------|
| Spiral\Http\Event\MiddlewareProcessing | 事件将在调用中间件 `之前` 触发。 |

> **注意**
> 要了解有关分派事件的更多信息，请参阅我们文档中的 [事件](../advanced/events.md) 部分。
