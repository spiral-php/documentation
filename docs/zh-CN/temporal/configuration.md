# Temporal — 入门

[Temporal](https://temporal.io/) 是一个开源的工作流引擎，它以分布式的方式管理和执行可靠、持久且容错的工作流。但要在 PHP 中使用它，唯一的方法是结合 [RoadRunner](https://roadrunner.dev/docs/workflow-temporal)。它使得将你的 PHP 应用与 Temporal 集成变得超级简单，你可以立即开始使用它。

关于 Temporal 的一个很酷的事情是，你可以使用任何支持的语言编写你的工作流和活动。例如，你可以有一个用 PHP 编写的工作流，但用 Go 或其他语言编写的代码来处理某些活动。如果你的团队对语言有不同的偏好，或者你想利用不同语言在不同任务上的优势，这会非常有帮助。

当你需要管理复杂的数据流或确保跨多个业务域的可靠事务处理时，可以使用 Temporal。它提供了计时器、重试机制等等。

## 安装

要在你的 PHP 项目中使用 Temporal，你需要安装 `spiral/temporal-bridge` 包。

步骤如下：

```terminal
composer require spiral/temporal-bridge
```

安装完该包后，你需要使用引导器来激活该组件：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\TemporalBridge\Bootloader\TemporalBridgeBootloader::class,
        // ...
    ];
}
```

在 [框架 — 引导器](../framework/bootloaders.md) 部分阅读更多关于引导器的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\TemporalBridge\Bootloader\TemporalBridgeBootloader::class,
    // ...
];
```

在 [框架 — 引导器](../framework/bootloaders.md) 部分阅读更多关于引导器的信息。
:::

::::

### 设置 Temporal 服务器

要快速开始使用 Temporal，可以使用开发服务器。

首先，安装 Temporal CLI - 按照 [Temporal 网站](https://docs.temporal.io/cli#install) 上的说明进行操作。

安装 CLI 后，你可以通过运行以下命令来启动 Temporal 服务器：

```terminal
temporal server start-dev
````

> **注意**
> 还有其他启动 Temporal 服务器的方法。例如，你可以使用 Docker。你可以在 [Temporal 存储库](https://github.com/temporalio/docker-compose) 中找到 docker-compose 文件。

## 配置

### PHP 应用程序

你只需要在 `.env` 文件中指定你的 Temporal 服务器的地址：

```dotenv .env
TEMPORAL_ADDRESS=127.0.0.1:7233
```

如果你想精确地配置你的应用程序，你可以创建 `temporal.php` 配置文件。在那里，你可以指定选项，如任务队列、命名空间和单个工作器配置。

以下是一个配置文件的示例：

```php app/config/temporal.php
use Spiral\TemporalBridge\Config\ConnectionConfig;
use Spiral\TemporalBridge\Config\ClientConfig;
use Temporal\Worker\WorkerFactoryInterface;
use Temporal\Worker\WorkerOptions;

return [
    'client' => env('TEMPORAL_CONNECTION', 'default'),
    'clients' => [
        'default' => new ClientConfig(
            new ConnectionConfig(
                address: env('TEMPORAL_ADDRESS', 'localhost:7233'),
            ),
        ),
    ],
    'defaultWorker' => WorkerFactoryInterface::DEFAULT_TASK_QUEUE,
    'workers' => [
        'workerName' => WorkerOptions::new(),
    ],
];
```

#### 客户端配置

`clients` 选项包含一个命名的 `ClientConfig` 设置列表，用于每个客户端。默认客户端的名称在 `client` 选项中指定。

`ClientConfig` 包含连接到 Temporal 服务器、gRPC 上下文设置和客户端设置。这些设置将在创建 Temporal 工作流和计划客户端时使用。

在连接设置中，你可以指定 Temporal 服务器地址、TLS 设置和其他凭据，例如身份验证令牌。顺便说一下，身份验证令牌也接受 `\Stringable` 类型，所以你可以在运行时更改其值，而无需重启工作器。

如果你需要指定自定义 Temporal 命名空间或身份，请自定义 `ClientOptions`。

使用上下文，你可以指定超时、RPC 调用重试设置以及与请求一起发送的元数据。

一个例子：

```php app/config/temporal.php
use Spiral\TemporalBridge\Config\TlsConfig;
use Spiral\TemporalBridge\Config\ConnectionConfig;
use Spiral\TemporalBridge\Config\ClientConfig;
use Temporal\Client\ClientOptions;
use Temporal\Client\Common\RpcRetryOptions;
use Temporal\Client\GRPC\Context;

return [
    // ...
    'clients' => [
        'default' => new ClientConfig(
            connection: new ConnectionConfig(
                address: 'foo-bar-default.baz.tmprl.cloud:7233',
                tls: new TlsConfig(),
                authToken: 'my-secret-token',
            ),
            options: (new ClientOptions())
                ->withNamespace('default')
                ->withIdentity('customer-service'),
            context: Context::default()
                ->withTimeout(4.5)
                ->withRetryOptions(
                    RpcRetryOptions::new()
                        ->withMaximumAttempts(5)
                        ->withInitialInterval(3)
                        ->withMaximumInterval(10)
                        ->withBackoffCoefficient(1.6)
                ),
];
```

#### 工作器选项

工作器选项用于配置 Temporal 工作器行为的各个方面。

1. **自定义工作器行为：** 该类提供了一系列选项，用于微调工作器如何处理活动和工作流任务。这包括设置并发限制、速率限制和管理任务轮询行为。
2. **资源管理：** 通过允许开发人员设置已执行活动数量和速率的限制，`WorkerOptions` 类有助于有效地管理系统资源。这对于维护系统稳定性和效率至关重要，尤其是在高负载场景中。
3. **控制任务处理：** 该类可以控制工作器如何检索和处理任务。诸如最大并发任务数和轮询器之类的选项有助于平衡工作负载和优化任务执行。
4. **增强的灵活性：** 它提供了高级选项，例如粘性调度和优雅的停止超时，从而更好地控制任务执行和工作器关闭过程。
5. **对基于会话的活动的支持：** 通过提供启用会话工作器和设置与会话相关的参数的选项，它促进了在会话上下文中执行活动，从而增强了 Temporal 工作流可以实现的内容范围。

你可以为每个任务队列定义工作器选项。例如：

```php app/config/temporal.php
use Temporal\Worker\WorkerOptions;

return [
    // ...
    'workers' => [
        'workerName' => WorkerOptions::new(),
        'default' => WorkerOptions::new()
           ->withMaxConcurrentActivityExecutionSize(10)
           ->withWorkerActivitiesPerSecond(100),
    ],
];
```

还有一种定义工作器选项的替代方法。例如：

```php app/config/temporal.php
use Temporal\Worker\WorkerOptions;

return [
    // ...
    'workers' => [
        'workerName' => [
            'options' => WorkerOptions::new(),
            'exception_interceptor' => new ExceptionInterceptor(),
        ]
    ],
];
```

使用这种方法，你可以另外定义一个异常拦截器。在 [Temporal — 拦截器](interceptors.md#exception-interceptor) 部分阅读更多关于拦截器的信息。

#### 拦截器

拦截器用于拦截工作流和活动调用。它们可以用于向调用过程添加自定义逻辑，例如日志记录或指标收集。

在 [Temporal — 拦截器](interceptors.md) 部分阅读更多关于拦截器的信息。

### RoadRunner

在你的 RoadRunner 配置文件 `.rr.yaml` 中，添加一个 `temporal` 部分。这允许你设置服务器地址和工作器数量。

例如：

```yaml .rr.yaml
...

temporal:
  address: ${TEMPORAL_ADDRESS:-localhost:7233}
  activities:
    num_workers: 10
```

有关使用 RoadRunner 配置 Temporal 的更多详细信息，请阅读 [RoadRunner](https://roadrunner.dev/docs/workflow-temporal) 文档。

## Temporal Cloud

如果你想使用 [Temporal Cloud](https://docs.temporal.io/cloud/get-started)，你必须配置一个安全连接和一个特定的命名空间。

```php app/config/temporal.php
use Spiral\TemporalBridge\Config\TlsConfig;
use Spiral\TemporalBridge\Config\ConnectionConfig;
use Spiral\TemporalBridge\Config\ClientConfig;
use Temporal\Client\ClientOptions;

return [
    'client' => 'production',
    'clients' => [
        'production' => new ClientConfig(
            connection: new ConnectionConfig(
                address: 'foo-bar-default.baz.tmprl.cloud:7233',
                tls: new TlsConfig(
                    privateKey: '/my-project.key',
                    certChain: '/my-project.pem',
                ),
            ),
            options: (new ClientOptions())
                ->withNamespace('foo-bar-default.baz'),
    ],
    // ...
];
```

```yaml .rr.yaml
...

temporal:
  address: foo-bar-default.baz.tmprl.cloud:7233
  namespace: foo-bar-default.baz
  tls:
    key: /my-project.key
    cert: /my-project.pem
```

就是这样！祝你工作流构建愉快！
