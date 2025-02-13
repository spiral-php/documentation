# 基础知识 — 日志

Spiral 提供了与 [PSR-3](https://www.php-fig.org/psr/psr-3/) 标准兼容的 `spiral/logger` 组件。该组件可用于记录各种类型的信息，例如错误、警告和调试消息，这有助于识别和解决应用程序中的问题。

默认情况下，框架不提供自己的实现，但是，可以使用 `spiral/monolog-bridge` 组件，它与 [Seldaek/monolog](https://github.com/Seldaek/monolog) 包完全集成，并支持各种强大的日志处理程序。

该框架使配置这些处理程序变得简单，允许通过使用不同的通道来自定义日志处理。

## 配置

要配置此组件，可以通过配置文件或启动器将其设置为您的偏好。此组件的配置文件通常位于 `app/config/monolog.php` 中。通过此文件，您可以选择默认处理程序，设置全局日志级别，并自定义处理程序和处理器以满足您的特定需求。

以下是配置文件的示例：

```php app/config/monolog.php
use Monolog\Handler\ErrorLogHandler;
use Monolog\Handler\SyslogHandler;
use Monolog\Logger;
use Monolog\Processor\PsrLogMessageProcessor;

return [
    /**
     * -------------------------------------------------------------------------
     *  Default Monolog handler
     * -------------------------------------------------------------------------
     */
    'default' => env('MONOLOG_DEFAULT_CHANNEL', 'default'),

    /**
     * -------------------------------------------------------------------------
     *  Global logging level
     * -------------------------------------------------------------------------
     *
     * Monolog supports the logging levels described by RFC 5424.
     *
     * @see https://seldaek.github.io/monolog/doc/01-usage.html#log-levels
     */
    'globalLevel' => Logger::toMonologLevel(
        env('MONOLOG_DEFAULT_LEVEL', \Monolog\Logger::DEBUG)
    ),

    /**
     * -------------------------------------------------------------------------
     *  Handlers
     * -------------------------------------------------------------------------
     *
     * @see https://seldaek.github.io/monolog/doc/02-handlers-formatters-processors.html#handlers
     */
    'handlers' => [
        'default' => [
            [
                'class' => 'log.rotate',
                'options' => [
                    'filename' => directory('runtime') . 'logs/app.log',
                    'level' => \Monolog\Logger::DEBUG,
                ],
            ],
        ],
        'stderr' => [
            ErrorLogHandler::class,
        ],
        'stdout' => [
            [
                'class' => SyslogHandler::class,
                'options' => [
                    'ident' => 'app',
                    'facility' => LOG_USER,
                ],
            ],
        ],
    ],

    /**
     * -------------------------------------------------------------------------
     *  Processors
     * -------------------------------------------------------------------------
     *
     * Processors allows adding extra data for all records.
     *
     * @see https://seldaek.github.io/monolog/doc/02-handlers-formatters-processors.html#processors
     */
    'processors' => [
        'default' => [
            [
                'class' => PsrLogMessageProcessor::class,
                'options' => [
                    'dateFormat' => 'Y-m-d\TH:i:s.uP',
                ],
            ],
        ],
    ],
];
```

> **注意**
> 使用 `MONOLOG_DEFAULT_CHANNEL` 环境变量来指定应用程序中应使用的默认处理程序。

### 日志格式

默认情况下，处理程序将使用以下结构格式化日志消息：`[%datetime%] %level_name%: %message% %context%\n`。

如果想更改日志消息结构，可以像下面的示例一样设置 `MONOLOG_FORMAT` 环境变量：

```dotenv .env
MONOLOG_FORMAT=[%datetime%] %level_name%: %message% %context%\n
```

> **了解更多**
> 了解有关可用占位符的更多信息，请参阅 [Monolog 文档](https://seldaek.github.io/monolog/doc/message-structure.html)。

## 注册处理程序

除了通过配置文件或启动器配置日志记录组件之外，您还可以通过 `Spiral\Monolog\Bootloader\MonologBootloader` 类注册处理程序。

### 日志轮转处理程序

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            'my-channel',
            $monolog->logRotate(directory('runtime') . 'logs/my-channel.log')
        );
    }
}
```

> **警告**
> 不要忘记将此启动器添加到 `app/src/Application/Kernel.php` 中的启动器列表顶部：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\LoggingBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分了解有关启动器的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\LoggingBootloader::class,
    // ...
];
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分了解有关启动器的更多信息。
:::

::::

### RoadRunner 处理程序

RoadRunner bridge 包提供了 `Spiral\RoadRunnerBridge\Logger\Handler` 处理程序，用于将日志发送到 [RoadRunner 应用程序日志记录器](https://roadrunner.dev/docs/plugins-applogger)。

您只需要将 `Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader` 添加到启动器列表的顶部：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分了解有关启动器的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\LoggerBootloader::class,
    // ...
];
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分了解有关启动器的更多信息。
:::

::::

> **警告**
> 确保您已安装 [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) 包。此包提供了将 RoadRunner 与 Monolog 集成的必要类。

并将默认通道更改为 `roadrunner`：

:::: tabs

::: tab 环境变量

```dotenv .env
MONOLOG_DEFAULT_CHANNEL=roadrunner
```

:::

::: tab 配置文件

```php app/config/monolog.php
return [
    'default' => 'roadrunner',
    // ...
];
```

:::

::::

## 使用

### 将日志发送到默认通道

为了使用日志记录组件，该框架使用 `Psr\Log\LoggerInterface` 类，该类可用于将消息记录到默认通道。

```php
use Psr\Log\LoggerInterface;

final class UserService
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {}

    public function register(string $email, string $password): void
    {
        // Register user ...
        
        $this->logger->info('User has been registered', ['email' => $email]);
    }
}
```

### 将日志发送到特定通道

有几种方法可以获取具有特定通道的 logger 实例：
- 使用实现 `Spiral\Logger\LogsInterface` 的 logger 工厂。
- 在自动装配期间使用 `Psr\Log\LoggerInterface` 参数上的属性 `Spiral\Logger\Attribute\LoggerChannel`。

:::: tabs

::: tab 使用工厂

```php
use Psr\Log\LoggerInterface;
use Spiral\Logger\LogsInterface;

final class UserService
{
    private readonly LoggerInterface $logger;

    public function __construct(LogsInterface $logs) 
    {
        $this->logger = $logs->channel('my-channel');
    }

    public function register(string $email, string $password): void
    {
        // Register user ...
        
        $this->logger->info('User has been registered', ['email' => $email]);
    }
}
```

:::

::: tab 使用属性

```php
use Psr\Log\LoggerInterface;
use Spiral\Logger\Attribute\LoggerChannel;

final class UserService
{
    public function __construct(
        #[LoggerChannel('my-channel')]
        private readonly LoggerInterface $logger
    ) {}

    public function register(string $email, string $password): void
    {
        // Register user ...

        $this->logger->info('User has been registered', ['email' => $email]);
    }
}
```

:::

::::

### Logger trait

Spiral 提供了一种方便的方法，可以通过使用 `Spiral\Logger\Traits\LoggerTrait` trait 快速将 Logger 分配给任何类。只需在类中包含此 trait，您就可以轻松访问 Logger 实例并记录消息。

> **警告**
> 默认情况下，用于日志记录的 **通道名称** 将是类名。 此 trait 允许您从任何类记录消息，而无需在构造函数中显式注入 Logger。

```php
use Spiral\Logger\Traits\LoggerTrait;

final class UserService
{
    use LoggerTrait;

    public function register(string $email, string $password): void
    {
        // Register user ...
        
        $this->getLogger()->info('User has been registered', ['email' => $email]);
    }
}
```

并为其分配一个 logger：

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    // ...
    
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            UserService::class,
            $monolog->logRotate(directory('runtime') . 'logs/user-dervice.log')
        );
    }
}
```

> **警告**
> LoggerTrait 仅在全局 [IoC 作用域](../framework/scopes.md) 内有效。

### 仅处理特定日志级别

在某些情况下，您可能只想记录特定的日志级别。 例如，您可能只想将应用程序错误聚合到一个日志文件中。

**为此，您可以订阅默认通道：**

```php app/src/Application/Bootloader/LoggingBootloader.php
namespace App\Application\Bootloader;

use Monolog\Logger;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;

final class LoggingBootloader extends Bootloader
{
    // ...
    
    public function boot(MonologBootloader $monolog): void
    {
        $monolog->addHandler(
            'default',
            $monolog->logRotate(directory('runtime') . 'logs/errors.log', Logger::ERROR) // only ERROR and above
        );
    }
}
```
