# 框架 — Dispatcher

Spiral 的一个关键特性是它对多个内核 dispatcher 的支持，这些 dispatcher 负责根据当前环境将传入的请求路由到适当的处理器。

让我们设想一下，我们有两个 dispatcher：`console` 和 `http`。

下面是一个 `http` dispatcher 的例子：

```php
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\DispatcherInterface;

final class HttpDispatcher implements DispatcherInterface
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {
    }
    
    public function canServe(): bool
    {
        return $this->env->get('RR_MODE') === 'http';
    }
    
    public function serve(): void
    {
        // Handle HTTP requests
    }
}
```

以及一个 `console` dispatcher 的例子：

```php
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\DispatcherInterface;

final class ConsoleDispatcher implements DispatcherInterface
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {
    }
    
    public function canServe(): bool
    { 
        return (PHP_SAPI === 'cli' && $this->env->get('RR_MODE') === null);
    }
    
    public function serve(InputInterface $input = null, OutputInterface $output = null): int
    {
        // Handle console command
    }
}
```

现在，我们可以在我们的应用程序中注册这些 dispatcher：

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\KernelInterface;

class AppBootloader extends Bootloader
{
    public function boot(
      KernelInterface $kernel, 
      HttpDispatcher $http,
      ConsoleDispatcher $console,
    ): void  {
        $kernel->addDispatcher($http)
        $kernel->addDispatcher($console);
    }
}
```

并且我们的应用程序的入口点 `app.php` 看起来像这样：

```php app.php
use App\Application\Kernel;

\mb_internal_encoding('UTF-8');
\error_reporting(E_ALL | E_STRICT ^ E_DEPRECATED);
\ini_set('display_errors', 'stderr');

require __DIR__ . '/vendor/autoload.php';
$app = Kernel::create(
    directories: ['root' => __DIR__],
)->run();

$code = (int)$app->serve();  // <========== 将根据当前环境运行合适的 dispatcher
exit($code);
```

当我们运行我们的应用程序时，将根据当前环境选择合适的 dispatcher。例如，如果我们运行以下命令：

```terminal
php app.php db:migrate
```

框架将遍历已注册的 dispatcher 列表，并调用每个 dispatcher 的 `canServe` 方法。此方法是 dispatcher 告诉框架它是否可以根据当前环境处理请求的方式。框架将使用第一个返回 `true` 的 dispatcher。

如果当前环境是 `cli`，`ConsoleDispatcher` 将返回 `true`。当 RoadRunner 启动 HTTP 插件时，该插件将启动一个工作进程，并将 `RR_MODE=http` 环境变量传递给该工作进程。在这种情况下，将选择 `HttpDispatcher`。

如果没有 dispatcher 返回 `true`，框架将抛出一个异常。

## 可用的 Dispatcher

Spiral 附带了几个内置的 dispatcher：

- [Console dispatcher](https://github.com/spiral/framework/blob/master/src/Framework/Console/ConsoleDispatcher.php)：
  负责处理应用程序内的控制台命令。如果您想创建可以从命令行运行的自定义命令，这将非常有用。

- [RoadRunner HTTP dispatcher](https://github.com/spiral/roadrunner-bridge/blob/2.0/src/Http/Dispatcher.php)：
  负责处理传入的 HTTP 请求，并将它们路由到相应的控制器动作或函数。这是当您的应用程序作为 HTTP 服务运行时使用的 dispatcher。

- [RoadRunner GRPC dispatcher](https://github.com/spiral/roadrunner-bridge/blob/2.0/src/GRPC/Dispatcher.php)：
  负责处理传入的 GRPC 请求，并将它们路由到相应的服务。这是当您的应用程序作为 GRPC 服务运行时使用的 dispatcher。

- [RoadRunner TCP dispatcher](https://github.com/spiral/roadrunner-bridge/blob/2.0/src/Tcp/Dispatcher.php)：负责处理传入的 TCP 连接，并将它们路由到相应的处理器。这是当您的应用程序作为 TCP 服务运行时使用的 dispatcher。

- [Temporal dispatcher](https://github.com/spiral/temporal-bridge/blob/2.0/src/Dispatcher.php)：负责处理传入的工作流活动。Temporal 是一个分布式、可扩展且具有容错能力的工作流引擎，用于构建和编排长时间运行的业务逻辑。

- [RoadRunner Queue dispatcher](https://github.com/spiral/roadrunner-bridge/blob/2.0/src/Queue/Dispatcher.php)：允许
  您从队列中消费消息并将它们路由到适当的处理器。如果您想构建一个基于消息驱动架构的系统，这将非常有用。
  这是当您的应用程序作为队列消费者服务运行时使用的 dispatcher。

> **注意**
> 阅读如何创建自定义 dispatcher [here](../cookbook/custom-dispatcher.md)

Spiral 内核 dispatcher 提供了一种灵活而强大的方式来将传入的请求路由到适当的处理器。它们是至关重要的组件，也是其整体设计的重要组成部分。
