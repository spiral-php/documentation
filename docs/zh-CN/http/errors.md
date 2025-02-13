# HTTP — 错误页面

你的应用程序会抛出一些错误和异常，其中一些必须传递给客户端，而另一些则不应传递。

## 配置

### Bootloader (引导加载器)

Spiral 应用程序 `spiral/app` bundle (包) 包含一个默认的异常处理器 bootloader `App\Application\Bootloader\ExceptionHandlerBootloader`，用于将一些默认实现绑定到容器：

- `Spiral\Http\Middleware\ErrorHandlerMiddleware\SuppressErrorsInterface` - 用于确定是否应该抑制错误
- `Spiral\Http\ErrorHandler\RendererInterface` - 用于渲染 HTTP 异常。

将 bootloader 添加到你的应用程序：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\ExceptionHandlerBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders (框架 — Bootloaders (引导加载器))](../framework/bootloaders.md) 部分阅读更多关于 bootloaders 的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\ExceptionHandlerBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders (框架 — Bootloaders (引导加载器))](../framework/bootloaders.md) 部分阅读更多关于 bootloaders 的信息。
:::

::::

> **Note (注意)**
> 如果你没有使用 `spiral/app` bundle (包)，你可以使用 `Spiral\Bootloader\Http\ErrorHandlerBootloader` 替代。

### 中间件

添加 bootloader 之后，你需要启用错误处理中间件。

HTTP 组件包含一个默认的错误处理中间件 `Spiral\Http\Middleware\ErrorHandlerMiddleware`，用于拦截和记录关键错误和用户级异常。

要启用中间件，请将其添加到全局中间件列表，例如通过 `RoutesBootloader`：

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Http\Middleware\ErrorHandlerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            ErrorHandlerMiddleware::class,
           // ...
        ];
    }

    // ...
}
```

> **See more (查看更多)**
> 在 [HTTP — Middleware (HTTP — 中间件)](middleware.md#global-middleware) 部分阅读更多关于全局中间件的信息。

该中间件将处理应用程序异常，并将它们以对开发者友好的模式呈现。

要抑制将异常详细信息传递到浏览器：

```dotenv .env
DEBUG=false
```

在这种情况下，将显示默认的错误页面。

> **Warning (警告)**
> 不要使用启用了调试模式的应用程序部署到生产环境中。

抑制传递异常详细信息的另一种方法是实现 `Spiral\Http\Middleware\ErrorHandlerMiddleware\SuppressErrorsInterface` 接口：

```php
use Spiral\Http\Middleware\ErrorHandlerMiddleware\SuppressErrorsInterface;

class SuppressErrors implements SuppressErrorsInterface
{
    public function suppressed(): bool
    {
        return true;
    }
}
```

并将其绑定到容器：

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
use Spiral\Http\Middleware\ErrorHandlerMiddleware\SuppressErrorsInterface;

class ExceptionHandlerBootloader extends Bootloader
{
    protected const SINGLETONS = [
        SuppressErrorsInterface::class => SuppressErrors::class,
        // ...
    ];
    
    // ...
}
```

## 客户端异常

你可以从你的控制器和中间件中抛出一些异常，以引发 HTTP 级别的错误页面。例如，我们可以使用 `NotFoundException` 触发 `404 Not Found`：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Core\Container\SingletonInterface;
use Spiral\Http\Exception\ClientException\NotFoundException;

class HomeController implements SingletonInterface
{
    public function index()
    {
        throw new NotFoundException();
    }
}
```

其他异常包括：

| 代码 | Exception (异常)                                                   |
|------|-------------------------------------------------------------|
| 400  | Spiral\Http\Exception\ClientException\BadRequestException   |
| 401  | Spiral\Http\Exception\ClientException\UnauthorizedException |
| 403  | Spiral\Http\Exception\ClientException\ForbiddenException    |
| 404  | Spiral\Http\Exception\ClientException\NotFoundException     |
| 500  | Spiral\Http\Exception\ClientException\ServerErrorException  |

> **Note (注意)**
> 不要将 HTTP 异常用于你的服务和仓库，因为它会将你的实现耦合到 HTTP 调度器。使用特定于领域的异常及其到 HTTP 异常的映射。

## 页面渲染器

默认情况下，中间件将使用 `Spiral\Http\ErrorHandler\PlainRenderer` - 一个没有任何样式的简单错误页面。如果你想使用自定义错误页面渲染器，你可以实现 `Spiral\Http\ErrorHandler\RendererInterface`：

```php app/src/Application/Exception/Renderer/ViewRenderer.php
namespace App\Application\Exception\Renderer;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface as Request;
use Spiral\Http\ErrorHandler\RendererInterface;
use Spiral\Http\Header\AcceptHeader;
use Spiral\Views\Exception\ViewException;
use Spiral\Views\ViewsInterface;

final class ViewRenderer implements RendererInterface
{
    private const GENERAL_VIEW = 'exception/error';
    private const VIEW = 'exception/%s';

    public function __construct(
        private readonly ViewsInterface $views,
        private readonly ResponseFactoryInterface $responseFactory
    ) {
    }

    public function renderException(Request $request, int $code, \Throwable $exception): ResponseInterface
    {
        $acceptItems = AcceptHeader::fromString($request->getHeaderLine('Accept'))->getAll();
        if ($acceptItems && $acceptItems[0]->getValue() === 'application/json') {
            return $this->renderJson($code, $exception);
        }

        return $this->renderView($code, $exception);
    }

    private function renderJson(int $code, \Throwable $exception): ResponseInterface
    {
        $response = $this->responseFactory->createResponse($code);

        $response = $response->withHeader('Content-Type', 'application/json; charset=UTF-8');
        $response->getBody()->write(\json_encode(['status' => $code, 'error' => $exception->getMessage()]));

        return $response;
    }

    private function renderView(int $code, \Throwable $exception): ResponseInterface
    {
        $response = $this->responseFactory->createResponse($code);

        try {
            // Try to find view for specific code
            $view = $this->views->get(\sprintf(self::VIEW, $code));
        } catch (ViewException) {
            // Otherwise use default error page
            $view = $this->views->get(self::GENERAL_VIEW);
        }

        $content = $view->render(['code' => $code, 'exception' => $exception]);
        $response->getBody()->write($content);

        return $response;
    }
}
```

并将其绑定到容器：

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
use Spiral\Http\ErrorHandler\RendererInterface;
use App\Application\Exception\Renderer\ViewRenderer;

class ExceptionHandlerBootloader extends Bootloader
{
    protected const SINGLETONS = [
        RendererInterface::class => ViewRenderer::class,
        // ...
    ];
    
    // ...
}
```

> **Note (注意)**
> 默认情况下，`App\Application\Service\ErrorHandler\ViewRenderer` 类已在 `spiral/app` 中包含和注册。

## 日志记录

默认应用程序包括 Monolog handler (处理器)，默认情况下，它订阅由 `ErrorHandlerMiddleware` 发送的消息。HTTP 错误日志位于 `app/runtime/logs/http.log` 中，并在 `App\Application\Bootloader\LoggingBootloader` 中配置：

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Monolog\Bootloader\MonologBootloader;

class LoggingBootloader extends Bootloader
{
    public function boot(MonologBootloader $monolog)
    {
        $monolog->addHandler(
            ErrorHandlerMiddleware::class,
            $monolog->logRotate(directory('runtime') . 'logs/http.log')
        );
    }
}
```

> **See more (查看更多)**
> 在 [The Basics — Logging (基础知识 — 日志记录)](../basics/logging.md) 部分阅读更多关于日志记录的信息。

<hr>

## 相关主题

- [Getting started — Configuration (入门 — 配置)](../start/configuration.md)
- [The Basics — Errors handling (基础知识 — 错误处理)](../basics/errors.md)
- [Getting started — Deployment (入门 — 部署)](../start/deployment.md)
