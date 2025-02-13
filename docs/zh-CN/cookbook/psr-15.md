# Cookbook — 自定义 HTTP 请求处理器

Spiral 框架兼容多个社区标准，包括 [PSR-7](https://www.php-fig.org/psr/psr-7/) (HTTP 消息接口)、[PSR-15](https://www.php-fig.org/psr/psr-15/) (HTTP 服务器请求处理器) 和 [PSR-17](https://www.php-fig.org/psr/psr-17/) (HTTP 工厂)。

这意味着您可以使用任何您想要的符合 PSR-15 的请求处理器实现，这意味着您可以选择最适合您应用程序的解决方案。

默认情况下，Spiral 框架包含 `Spiral\Router\Router` 类，该类实现了 `Psr\Http\Server\RequestHandlerInterface` 接口并处理 HTTP 请求。然而，如果需要，开发人员可以禁用 `Spiral\Bootloader\Http\RouterBootloader` 并使用替代的请求路由解决方案。

## Fast Route

在本指南中，我们将向您展示如何使用 [FastRoute](https://github.com/nikic/FastRoute) 作为默认 Spiral 路由器的替代方案。FastRoute 是一个快速的路由库，它允许开发人员轻松地将 HTTP 请求路由到回调函数。

作为一个例子，我们将使用 [FastRoute](https://github.com/nikic/FastRoute) 替换默认的 Spiral 路由器。该实现由 [https://github.com/middlewares/fast-route](https://github.com/middlewares/fast-route) 提供。

### 预备知识

在使用 FastRoute 之前，您需要安装必要的软件包：

```terminal
composer require middlewares/fast-route middlewares/request-handler
```

### 实现 FastRoute

为了在 Spiral 框架中使用 FastRoute，您需要创建一个自定义的 Bootloader 将 FastRoute 的实现绑定到 HTTP 服务器。Bootloader 应该将此实现绑定到我们的 HTTP 服务器。只需在 `SINGLETONS` 常量中声明 `Psr\Http\Server\RequestHandlerInterface` 即可。

以下是 FastRoute Bootloader 的一个示例：

```php app/src/Application/Bootloader/FastRouteBootloader.php
namespace App\Application\Bootloader;

use FastRoute;
use Middlewares;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Psr\Http\Message\ServerRequestInterface;

final class FastRouteBootloader extends Bootloader
{
    protected const SINGLETONS = [
        RequestHandlerInterface::class => [self::class, 'psr15Handler'],
    ];

    private function defineRoutes(FastRoute\RouteCollector $router): void
    {
        $router->addRoute('GET', '/hello/{name}', static function (ServerRequestInterface $request): string {
            $name = $request->getAttribute('name');

            return \sprintf('Hello %s', $name);
        });
    }

    private function psr15Handler(ResponseFactoryInterface $responseFactory): RequestHandlerInterface
    {
        $dispatcher = FastRoute\simpleDispatcher(function (FastRoute\RouteCollector $r) {
            $this->defineRoutes($r);
        });

        return new Middlewares\Utils\Dispatcher([
            new Middlewares\FastRoute($dispatcher, $responseFactory),
            new Middlewares\RequestHandler(),
        ]);
    }
}
```

在上面的例子中，`defineRoutes` 函数用于定义将由 FastRoute 处理的路由。在这种情况下，我们定义了一个匹配 `/hello/{name}` URL 请求的路由，并将请求传递给一个返回响应的回调函数。

将此 Bootloader 添加到您的应用程序中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Http\HttpBootloader::class,
        \App\Bootloader\FastRouteBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 Bootloader 的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Http\HttpBootloader::class,
    \App\Bootloader\FastRouteBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 Bootloader 的信息。
:::

::::

一旦您实现了 FastRoute Bootloader 并将其添加到您的 Spiral 应用程序中，您将能够使用 FastRoute 处理 HTTP 请求。

恭喜您，现在您知道了如何在 Spiral 框架中使用 `Psr\Http\Server\RequestHandlerInterface` 接口的自定义实现！ 通过使用 FastRoute 库或任何其他 [符合 PSR-15 的库](https://packagist.org/?query=psr-15%20router)，您可以轻松地替换默认的 Spiral 路由器，并使用替代解决方案来处理 HTTP 请求。 无论您需要使用不同的路由库，实现自定义请求处理逻辑，还是使用来自多个来源的组件，Spiral 框架对 PSR-15 的兼容性都使您能够轻松地在应用程序中使用各种请求处理器实现。

**所以，继续构建您能构建的最好的应用程序吧！**
