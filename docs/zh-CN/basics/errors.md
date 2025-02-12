# 基础知识 — 错误处理

在开发过程中，错误和异常是常见的。调试这些异常可能是一项具有挑战性且耗时的任务，但它是开发过程中一个关键的方面。Spiral 提供了各种工具和技术来调试异常并识别问题的根本原因。

本文档将引导你了解 Spiral 中可用的异常处理、渲染和自定义功能。

## 异常处理器

Spiral 提供了由 `Spiral\Exceptions\ExceptionHandler` 类提供的强大异常处理机制。

它旨在管理全局和运行时错误，为你的应用程序提供结构化的异常处理方法。

#### 主要特性

-   **异常渲染：** 以多种格式渲染异常，包括 HTML、JSON 和纯文本。
-   **异常报告：** 将异常报告给外部服务，例如 [Sentry](#sentry-integration) 或 [S3 存储](#cloud-storage-reporter)。
-   **全局错误处理：** 该类用于处理全局错误，例如致命错误和关闭错误。
-   **可定制：** 该类允许添加自定义渲染器和报告器。

### 自定义异常处理器

对于需要特定错误处理策略的应用程序，Spiral 提供了用自定义实现替换此默认处理器的灵活性。

首先，创建一个扩展 `Spiral\Exceptions\ExceptionHandler` 类的类：

```php app/src/Application/Exception/Handler.php
<?php

declare(strict_types=1);

namespace App\Application\Exception;

use Spiral\Exceptions\ExceptionHandler;
use Throwable;

final class Handler extends ExceptionHandler
{
    // ...
}
```

接下来，在 `app.php` 文件中指定该类：

```php app.php
use App\Application\Kernel;
use App\Application\Exception\Handler;

// ...

$app = Kernel::create(
    directories: ['root' => __DIR__],
    exceptionHandler: Handler::class, // <--
)->run();

// ...
```

初始化处理器时，它将调用 `bootBasicHandlers` 方法，这是自定义处理器的方法之一。此方法用于注册基本渲染器和报告器。

```php app/src/Application/Exception/Handler.php
final class Handler extends ExceptionHandler
{
    protected function bootBasicHandlers(): void
    {
        parent::bootBasicHandlers();
        
        // 在这里注册你的渲染器和报告器
        // $this->addRenderer(new MyRenderer());
        // $this->addRenderer(new MyReporter());
    }
}
```

Handler 是处理应用程序启动过程中发生的异常的好地方。例如，如果你想跳过报告某些异常，你可以重写 `report` 方法并在那里处理它们。

```php app/src/Application/Exception/Handler.php
<?php

declare(strict_types=1);

namespace App\Application\Exception;

use Spiral\Exceptions\ExceptionHandler;
use Spiral\Http\Exception\ClientException;

final class Handler extends ExceptionHandler
{
    /**
     * @var class-string<\Throwable>[]
     */
    private array $nonReportableExceptions = [
        ClientException::class,
        // ...
    ];

    public function report(\Throwable $exception): void
    {
        foreach ($this->nonReportableExceptions as $nonReportableException) {
            if ($exception instanceof $nonReportableException) {
                return;
            }
        }

        parent::report($exception);
    }
}
```

## 异常渲染

Spiral 使用格式来确定哪个渲染器应该用于处理给定的异常。格式可以基于环境，例如用于控制台应用程序的 `cli` 或用于 HTTP 请求的 `http`。这允许根据遇到异常的上下文注册和使用不同的渲染器。

### 工作原理

有时，你可能希望以特殊方式显示错误，例如在 JSON 中显示给 API。以下是如何做到这一点：

1.  **创建渲染器**

这是一个关于如何实现 JSON 渲染器的例子：

```php app/src/Application/Exception/Renderer/JsonRenderer.php
<?php

namespace Spiral\YiiErrorHandler;

use Spiral\Exceptions\ExceptionRendererInterface;
use Spiral\Exceptions\Verbosity;
use Yiisoft\ErrorHandler\Renderer\JsonRenderer as YiiJsonRenderer;
use Yiisoft\ErrorHandler\ThrowableRendererInterface;

final class JsonRenderer implements ExceptionRendererInterface
{
    public const FORMATS = ['application/json', 'json'];

    public function __construct(
        private readonly ?ThrowableRendererInterface $renderer = new YiiJsonRenderer()
    ) {
    }

    public function render(
        \Throwable $exception,
        ?Verbosity $verbosity = Verbosity::BASIC,
        string $format = null,
    ): string {
        if ($verbosity >= Verbosity::VERBOSE) {
            return (string)$this->renderer->renderVerbose($exception);
        }

        return (string)$this->renderer->render($exception);
    }

    public function canRender(string $format): bool
    {
        return \in_array($format, self::FORMATS, true);
    }
}
```

>   **注意**
>   `Spiral\YiiErrorHandler\JsonRenderer` 是 `spiral-packages/yii-error-handler-bridge` 包的一部分。

2.  **注册你的渲染器**

使用启动器注册自定义渲染器

```php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;
use Spiral\YiiErrorHandler\JsonRenderer;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler): void
    {
        $handler->addRenderer(new JsonRenderer());
    }
}
```

>   **警告**
>   不要忘记将此启动器添加到 `app/src/Application/Kernel.php` 中的启动器列表的顶部：

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

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\ExceptionHandlerBootloader::class,
    // ...
];
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::::

3.  **使用你的渲染器**

要使用此渲染器处理 Web 应用程序中的异常，我们可以创建一个新的中间件，该中间件将捕获所有异常，并且仅当请求中存在特定标头（例如 `Accept=application/json`）时，才使用此渲染器呈现它们。这允许更精细地控制如何根据客户端所需的格式处理和显示异常。

```php
namespace App\Endpoint\Web\Middleware;

use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\RequestHandlerInterface as Handler;
use Psr\Http\Message\ResponseFactoryInterface;
use Spiral\Exceptions\ExceptionRendererInterface;
use Spiral\Http\Exception\ClientException;
use Spiral\Router\Exception\RouterException;

class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ExceptionRendererInterface $renderer,
        private readonly ResponseFactoryInterface $responseFactory,
    ) {
    }

    public function process(Request $request, Handler $handler): Response
    {
        try {
            return $handler->handle($request);
        } catch (ClientException|RouterException $e) {
            $code = $e instanceof ClientException ? $e->getCode() : 404;
        } catch (\Throwable $e) {
            $code = 500;
        }
        
        $response = $this->responseFactory->createResponse($code);
        $response->getBody()->write(
            (string) $this->renderer->render(
                exception: $e,
                format: $request->getHeaderLine('Accept') ?? 'application/json'
            )
        );

        return $response;
    }
}
```

如你所见，不同的渲染器可以用于不同的环境，例如用于命令行应用程序的控制台渲染器或用于 API 响应的 JSON 渲染器。此外，不同的渲染器可以用于不同的格式。

### 现有渲染器

以下是 Spiral 已经提供的一些渲染器：

| 渲染器                                     | 格式                                         |
|----------------------------------------------|-------------------------------------------------|
| `Spiral\Exceptions\Renderer\ConsoleRenderer` | `console`, `cli`                                |
| `Spiral\Exceptions\Renderer\JsonRenderer`    | `application/json`, `json`                      |
| `Spiral\Exceptions\Renderer\PlainRenderer`   | `text/plain`, `text`, `plain`, `cli`, `console` |

在某些情况下，例如当 `DEBUG=true` 时，你可能更喜欢呈现一个美观的错误页面，其中包含代码高亮，例如 [filp/whoops](https://github.com/filp/whoops) 或 [yiisoft/error-handler](https://github.com/spiral-packages/yii-error-handler-bridge)。

### Yii 错误渲染器

Yii 错误处理器是 Spiral 的一个桥接包，它提供了与 Yii 框架的错误处理器的集成。

![screenshot](https://user-images.githubusercontent.com/773481/215085868-a7228f6c-1be0-460d-b910-85fa2cd1195b.png)

#### 安装

要安装该组件：

```terminal
composer require spiral-packages/yii-error-handler-bridge
```

安装完成后，你需要从该包中注册 `Spiral\YiiErrorHandler\Bootloader\YiiErrorHandlerBootloader` 启动器：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\YiiErrorHandler\Bootloader\YiiErrorHandlerBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\YiiErrorHandler\Bootloader\YiiErrorHandlerBootloader::class,
    // ...
];
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::::

`YiiErrorHandlerBootloader` 将在初始化期间注册所有可用的渲染器。如果你希望注册特定的渲染器。

#### 内置渲染器

该桥接提供了几个内置渲染器用于显示错误：

-   `HtmlRenderer`：将错误页面呈现为 HTML。
-   `JsonRenderer`：将错误页面呈现为 JSON。这对于处理 API 请求中的错误非常有用。
-   `PlainTextRenderer`：将错误页面呈现为纯文本。

### 详细程度级别

详细程度级别可用于控制在呈现异常时显示的信息量。你可以在 `.env` 文件中设置 `VERBOSITY_LEVEL`：

```dotenv .env
# 详细程度级别
VERBOSITY_LEVEL=verbose # basic, verbose, debug
```

可能的值由 `Spiral\Exceptions\Verbosity` 枚举定义：

#### basic 或 0

表示应该仅显示有关异常的基本信息。如果发生错误，你将看到：

```output
[Spiral\Router\Exception\RouteNotFoundException]
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75
```

#### verbose 或 1

表示应该显示关于异常的更详细信息。如果发生错误，你将看到：

```output

[Spiral\Router\Exception\RouteNotFoundException]
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75

 1. Spiral\Router\Router->Spiral\Router\{closure}() at vendor/spiral/framework/src/Router/src/Router.php:75
 2. Spiral\Router\Router->Spiral\Router\{closure}()
 3. ReflectionFunction->invokeArgs() at vendor/spiral/framework/src/Core/src/Internal/Invoker.php:73
 4. ...
```

#### debug 或 2

表示应该显示关于异常的最详细信息。如果发生错误，你将看到：

```output
[Spiral\Router\Exception\RouteNotFoundException]
Unable to route `http://127.0.0.1`. in vendor/spiral/framework/src/Router/src/Router.php:75

 1. Spiral\Router\Router->Spiral\Router\{closure}() at vendor/spiral/framework/src/Router/src/Router.php:75
   73                 if ($route === null) {
   74                     $this->eventDispatcher?->dispatch(new RouteNotFound($request));
>  75                     throw new RouteNotFoundException($request->getUri());
   76                 }
   77

 2. ...
```

<hr />

## 异常报告

在 Spiral 中，你可以使用报告器来跟踪应用程序中的问题，例如错误。报告器可以做两件主要的事情：

-   它们可以将有关这些问题的信息保存在文件中。这样，你以后可以查看该文件以找出出了什么问题。
-   它们还可以将关于这些问题的报告发送到其他服务，例如 [Sentry](https://sentry.io)。这些服务可以提供关于应用程序中问题的更多详细信息。

想象一下你有一个网站，有时事情不能像它们应该的那样工作。这可能是由于代码中的错误，例如文件丢失或数据库出现问题。报告器帮助你有效地处理这些错误。

### 报告器的工作方式

#### 1. 实现 `ExceptionReporterInterface` 或使用内置报告器：

你将创建一个类（例如示例中的 `CustomReporter`），该类实现了 `Spiral\Exceptions\ExceptionReporterInterface`。将此类视为报告器代理，它知道当发生异常时该做什么。

>   **注意**
>   在下面的 [可用报告器](#available-reporters) 部分阅读更多关于可用报告器的信息。

```php app/src/Application/Exception/Reporter/CustomReporter.php
namespace App\Application\Exception\Reporter;

final class CustomReporter implements ExceptionReporterInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {}

    public function report(\Throwable $exception): void
    {
        // 将异常信息存储在文件中或将其发送到外部服务
        $this->logger->error($exception->getMessage(), ['exception' => $exception]);
    }
}
```

#### 2. 报告器的注册：

要使用报告器，首先需要使用 `addReporter` 方法将它们注册到 `ExceptionHandler` 类，并提供实现 `Spiral\Exceptions\ExceptionReporterInterface` 的类的实例。

```php app/src/Application/Bootloader/ExceptionHandlerBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Exceptions\ExceptionHandler;
use App\Application\Exception\Reporter\CustomReporter;

final class ExceptionHandlerBootloader extends Bootloader
{
    public function init(ExceptionHandler $handler): void
    {
        $handler->addReporter(new CustomReporter());
    }
}
```

#### 3. 在你的代码中使用报告器：

现在，在你的应用程序代码中，你可以在预期可能发生异常时使用这些报告器。

例如，在 `PingSiteJob` 示例中，如果尝试 ping 网站时出现问题（例如，网站已关闭），则会捕获异常并使用报告器进行报告。

```php
<?php

declare(strict_types=1);

namespace App\Job;

use Spiral\Exceptions\ExceptionReporterInterface;

final class PingSiteJob
{
    public function __construct(
        private PingClient $client,
        private ExceptionReporterInterface $reporter,
    ) {
    }

    public function handle(string $url): void
    {
        try {
            $this->client->ping($url);
        } catch (\Throwble $e) {
            $this->reporter->report($e);
        }
    }
}
```

>   **注意**
>   报告器将通过所有已注册的报告器发送异常。

### 可用报告器

Spiral 带有两个开箱即用的内置报告器、`Spiral\Exceptions\Reporter\LoggerReporter`、`Spiral\Exceptions\Reporter\FileReporter` 和 `Spiral\Exceptions\Reporter\StorageReporter`。

#### 日志记录器报告器

`Spiral\Exceptions\Reporter\LoggerReporter` 默认启用，允许你使用应用程序中注册的记录器记录异常。这对于跟踪和分析随时间推移发生的错误非常有用。

#### 文件报告器

`Spiral\Exceptions\Reporter\FileReporter` 也默认启用，它允许你将有关异常的详细信息保存到位于 `runtime/snapshots` 目录中的名为 `snapshot` 的文件中。

#### 云存储报告器

你是否曾经在处理无状态应用程序时遇到存储应用程序的异常快照的挑战？我们有一些好消息。我们已经让你变得非常容易。

通过与 `spiral/storage` 组件集成，我们使你的无状态应用程序能够将异常快照直接保存到云存储中，例如 **S3**。

**这对你来说为什么很棒？**

1.  **简化的存储：** 不再需要处理复杂的存储解决方案。轻松地将快照直接保存到 S3。
2.  **专为无状态应用程序量身定制：** 专为无状态应用程序设计，使你的部署更顺畅、更轻松。
3.  **可靠性：** 凭借 S3 久经考验的良好记录，知道你的快照被安全地存储，并且可以在你需要的任何时候访问。

`Spiral\Exceptions\Reporter\StorageReporter` 也默认启用，它允许你将关于异常的详细信息保存到位于 `runtime/snapshots` 目录中的名为 `snapshot` 的文件中。

**要使用此报告器，你需要：**

1.  设置 `spiral/storage` 组件。在 [组件 — 存储和云分发](../advanced/storage.md) 部分阅读更多关于它的信息。
2.  注册 `Spiral\Bootloader\StorageSnapshotsBootloader`
3.  使用 `SNAPSHOTS_BUCKET` 环境变量指定你希望存储快照的存储桶。
4.  在 `Spiral\Exceptions\ExceptionHandler` 类中注册 `Spiral\Exceptions\Reporter\StorageReporter`。

## Sentry 集成

Spiral 提供了一个 Sentry 桥接包，它促进了与 [Sentry](https://sentry.io/) 服务的轻松集成。本文档将指导你完成在你的 Spiral 应用程序中集成和自定义此工具的过程。

### 安装

1.  安装 Sentry 桥接组件：

```terminal
composer require spiral/sentry-bridge
```

2.  安装完成后，将该包中的 `Spiral\Sentry\Bootloader\SentryReporterBootloader` 启动器注册到你的

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::::

它将在 `Spiral\Exceptions\ExceptionHandler` 中注册 `Spiral\Sentry\SentryReporter`。

### 配置

你所需要做的就是将 `SENTRY_DSN` 环境变量设置为你的 Sentry DSN。

```dotenv .env
SENTRY_DSN=https://...
```

自 **v2.2** 起，你还可以使用其他环境变量来配置报告器：

-   `SENTRY_DSN`：Sentry 数据源名称 (DSN)。
-   `SENTRY_SAMPLE_RATE`：采样事件的速率（例如，`0.4`）。
-   `SENTRY_TRACES_SAMPLE_RATE`：用于跟踪采样的速率（例如，`1.0`）。
-   `SENTRY_SEND_DEFAULT_PII`：是否发送默认的个人身份信息 (`true`/`false`)。
-   `SENTRY_ENVIRONMENT`：环境（例如，`develop`）。你可以选择使用 `APP_ENV`。
-   `SENTRY_RELEASE`：版本（例如，`1.0.0`）。或者，使用 `APP_VERSION`。

这是一个例子：

```dotenv .env
SENTRY_DSN=https://...
SENTRY_SAMPLE_RATE=0.4
SENTRY_TRACES_SAMPLE_RATE=1.0
SENTRY_SEND_DEFAULT_PII=false

SENTRY_ENVIRONMENT=develop
SENTRY_RELEASE=1.0.0
# or
APP_ENV=develop
APP_VERSION=1.0.0
```

我们还提供了一种使用 `config/sentry.php` 文件配置报告器的方法：

```php config/sentry.php
return [
  'dsn' => 'http://...',
  'environment' => 'develop',
  'release' => '1.0.0',
  'sample_rate' => 1.0,
  'traces_sample_rate' => null,
  'send_default_pii' => true,
];
```

### Sentry 集成 [自 **v2.2** 起]

自 **v2.2** 起，我们添加了对 [Sentry 集成](https://docs.sentry.io/platforms/php/integrations/) 的支持。

你可以通过 `Spiral\Sentry\Bootloader\ClientBootloader` 注册特定于应用程序的集成。这使得添加针对你的应用程序需求定制的自定义功能变得简单直接。

**注册自定义集成的示例：**

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Sentry\Bootloader\ClientBootloader;

use Spiral\Boot\Bootloader\Bootloader;

final class AppBootloader extends Bootloader
{
    public function init(ClientBootloader $sentry): void
    {
        $sentry->addIntegration(new ExceptionContextIntegration());
    }
}
```

#### HTTP 请求集成

Sentry 将使用内置的 `Sentry\Integration\RequestIntegration` 集成自动收集有关当前请求的信息。还有 `Spiral\Sentry\Http\SetRequestIpMiddleware`。当启用 `send_default_pii` 时，此中间件对于收集用户 IP 地址至关重要。对于那些需要详细用户见解的人来说，这是一个可选但强大的功能。

>   **注意**
>   在 [HTTP — 路由](../http/routing.md#add-middleware) 部分阅读更多关于中间件的信息。

### 可用容器绑定 [自 **v2.2** 起]

对于寻求更深层次集成和控制的开发人员，我们引入了新的容器绑定：

-   `Sentry\Options`：用于微调 Sentry 客户端设置的配置容器。
-   `Sentry\State\HubInterface`：提供对 Sentry 的状态和上下文管理的访问。
-   `Sentry\ClientInterface`：促进与 Sentry 客户端的直接交互。

这些绑定提供了对 Sentry 客户端的精细控制，满足了高级用例。

### 提供附加数据

要公开当前应用程序日志，例如应用程序日志或 PSR-7 请求状态，请启用调试信息收集器。这些收集器收集有关当前应用程序状态的相关数据。

发生异常时，Sentry 报告器将从 IoC 容器请求 `Spiral\Debug\StateInterface` 类，并且在创建期间，该对象将填充来自已注册收集器的信息。

>   **警告**
>   从容器请求 `Spiral\Debug\StateInterface` 时要小心。该对象将在每次从容器请求时创建，并且你无法在收集器之外填充它。如果需要向 `Spiral\Debug\StateInterface` 对象添加其他信息，则应使用收集器。

#### Http 收集器

HTTP 收集器是向 Sentry 发送有关当前请求信息的良好方式。

>   **注意**
>   自 **v2.2** 起，可以避免使用 HTTP 收集器，因为报告器将使用内置的 `Sentry\Integration\RequestIntegration` 集成以更好的方式自动收集有关当前请求的信息。

它将发送有关当前请求的以下信息：

-   `method`
-   `url`
-   `headers`
-   `query params`
-   `request body`

要启用 HTTP 收集器，首先需要在 `SentryReporterBootloader` 之前注册 `Spiral\Bootloader\Debug\HttpCollectorBootloader`。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Debug\HttpCollectorBootloader::class,
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Debug\HttpCollectorBootloader::class,
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::::

然后你需要在应用程序中注册中间件 `Spiral\Debug\StateCollector\HttpCollector`。

>   **查看更多**
>   在 [HTTP — 路由](../http/routing.md#add-middleware) 部分阅读更多关于如何注册中间件的信息。

#### 日志收集器

使用日志收集器将所有接收到的日志发送到 Sentry。

要启用日志收集器，你只需在 `SentryBootaloder` 之前注册 `Spiral\Bootloader\Debug\LogCollectorBootloader`。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Debug\LogCollectorBootloader::class,
        \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Debug\LogCollectorBootloader::class,
    \Spiral\Sentry\Bootloader\SentryReporterBootloader::class,
    // ...
];
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::::

#### 创建自定义收集器

对于专门的数据收集，你可以创建自定义收集器。收集器应该实现 `Spiral\Debug\StateCollectorInterface` 接口。

例如，考虑一个 SQL 收集器：

```php app/src/Application/Debug/Collector/SqlCollector.php
namespace App\Application\Debug\Collector;

use Spiral\Logger\Event\LogEvent;
use Spiral\Debug\StateCollectorInterface;

final class SqlCollector implements StateCollectorInterface
{
    public function __construct(
        private readonly Database $db
    ) {
    }

    public function collect(\Spiral\Debug\StateInterface $state): void
    {
       foreach($this->db->getQueries() as $query) {
            $state->addLogEvent(new LogEvent(
                time: $query->getTime(),
                channel: 'sql',
                level: 'info',
                message: $query->getQuery(),
                context: $query->getParameters()
            ));
       }
    }
}
```

>   **警告**
>   上面的示例使用了不存在的 Database 类，这意味着你需要自己实现它。

以下是 `Spiral\Debug\StateInterface` 对象的一些有用方法：

**添加标签**

该方法将添加与当前范围关联的标签

```php
$state->addTag('IP address', $currentRequest->getIpAddress());
$state->addTag('Environment', $env->get('APP_ENV'));
```

**添加变量**

该方法将添加与当前范围关联的额外数据

```php
$state->setVariable('query', $currentRequest->getQueryParams());
```

**添加日志事件**

该方法将添加一个日志事件作为当前范围的面包屑。

```php
$state->addLogEvent(new \Spiral\Logger\Event\LogEvent(
    time: new \DateTimeImmutable(),
    channel: 'default',
    level: 'info',
    message: 'Something went wrong',
    context: ['foo' => 'bar']
));
```

#### 自定义收集器注册

你可以在启动器中注册你的收集器。

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\DebugBootloader;
use App\Application\Exception\Reporter\CustomReporter;

final class AppBootloader extends Bootloader
{
    public function init(DebugBootloader $debug, SqlCollector $sqlCollector): void
    {
        $debug->addStateCollector($sqlCollector);
    }
}
```
