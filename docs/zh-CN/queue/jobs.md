# 队列 — 任务处理器

任务处理器是 Spiral 中至关重要的一部分，它提供了一种结构化的方法，用于高效地执行任务并管理其负载。任务处理器是负责在系统中执行特定任务或操作的类。

## 创建处理器

要创建任务处理器，您需要实现 `Spiral\Jobs\HandlerInterface` 接口。此接口定义了处理任务、执行任务以及处理任务过程中可能发生的任何潜在错误所需的方法。Spiral 还提供了一个方便的抽象类 `Spiral\Queue\JobHandler`，您可以扩展它来简化任务处理器的实现。

为了轻松地创建任务处理器，您可以使用脚手架命令：

```terminal
php app.php create:jobHandler Sample
```

> **注意**
> 在 [基础知识 — 脚手架](../basics/scaffolding.md#job-handler) 部分阅读更多关于脚手架的内容。

执行此命令后，以下输出将确认创建成功：

```output
Declaration of ' [32mSampleJob [39m' has been successfully written into ' [33mapp/src/Endpoint/Job/SampleJob.php [39m'.
```

现在您可以在 `app/src/Endpoint/Job` 目录中找到 `SampleJob` 类。

```php app/src/Endpoint/Job/SampleJob.php
namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

final class SampleJob extends JobHandler
{
    public function invoke(string $id, array $payload, array $headers): void
    {
        // 使用服务做些什么
    }
}
```

目前，新的任务处理器不执行任何操作。

## 派发任务

### 推送到默认队列

您可以通过 `Spiral\Queue\QueueInterface` 或通过原型属性 `queue` 派发您的任务。当您从容器中请求 `Spiral\Queue\QueueInterface` 时，您将收到默认队列连接的实例。

`QueueInterface` 的 `push` 方法接受任务名称、负载和附加选项。

```php
use App\Endpoint\Job\SampleJob;
use Spiral\Queue\QueueInterface;

public function createJob(QueueInterface $queue): void
{
    $queue->push(SampleJob::class);
}
```

您可以使用您的处理器名称作为任务名称。它将自动转换为 `-` 标识符，例如，`App\Endpoint\Job\SampleJob` 将表示为 `app-jobs-sampleJob`。

### 推送到特定队列

如果您需要使用特定的队列连接推送任务，您可以使用 `Spiral\Queue\QueueConnectionProviderInterface`。

```php
use App\Endpoint\Job\SampleJob;
use Spiral\Queue\QueueConnectionProviderInterface;

final class MyService
{
    public function __construct(
        private readonly QueueConnectionProviderInterface $provider
    ) {
    }

    public function createJob(): void
    {
        $this->provider->getConnection('sync')->push(SampleJob::class);
    }
}
```

## 传递参数

`QueueInterface->push()` 方法的第二个参数可以接受任何类型的变量，例如 **数组**，**对象**，**字符串** 等等。但是，重要的是要注意框架使用的默认序列化程序是 `json`。

```php
use App\Endpoint\Job\SampleJob;
use Spiral\Queue\QueueInterface;

public function createJob(QueueInterface $queue): void
{
    // 数组负载
    $queue->push(SampleJob::class, ['value' => 123]);
    
    // 对象负载
    $queue->push(SampleJob::class, new User(id: 123, name: 'John'));
    
    // 一些字符串负载
    $queue->push(SampleJob::class, 'some string');
}
```

## 处理任务

当任务被派发时，队列服务将自动找到该任务的处理器并执行它。`invoke` 方法负责处理任务处理器接收到的已排队任务。

```php app/src/Endpoint/Job/SampleJob.php
namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

class SampleJob extends JobHandler
{
    public function invoke(string $id, array $payload): void
    {
        // 使用服务做些什么
    }
}
```

该方法接受一些参数，这些参数描述如下：

#### 负载

`$payload` 参数包含任务排队时添加到队列中的数据。它可以是任何类型，例如 `array`、`object`、`string` 等。

> **警告**
> `$payload` 参数的类型应与您传递给 `push` 方法的负载类型相同。

#### 任务 ID

`$id` 参数是一个可选的字符串，包含任务的唯一标识符。这可用于跟踪应用程序中任务的进度。

#### 任务头

`$headers` 参数是一个可选的数组，包含当任务被推送到队列时可以添加的附加头或上下文。这对于提供关于任务的附加信息或将上下文数据传递给 invoke 方法很有用。

**可以添加到头中的一些上下文数据示例包括：**

- **重试次数：** 任务已重试的次数。这对于确定任务是否已多次失败并需要以不同的方式处理很有用。

- **优先级：** 任务的优先级。这对于确保首先处理重要任务，或者根据任务的重要性对任务进行优先级排序很有用。

- **时间戳：** 任务添加到队列中的时间戳。这对于跟踪任务的进度或用于日志记录目的很有用。

- **用户 ID：** 启动任务的用户的 ID。这对于跟踪单个用户的操作或实施用户特定的策略很有用。

#### 依赖注入

您可以在您的处理器的 `invoke` 方法中自由使用方法注入。当调用任务处理器时，依赖注入容器将自动为该方法提供指定的依赖项。

```php app/src/Endpoint/Job/SampleJob.php
namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;
use Psr\Log\LoggerInterface;

class SampleJob extends JobHandler
{
    public function invoke(LoggerInterface $logger, array $payload): void
    {
        $logger->debug('Job processing...', ['id' => $id]);
        
        // 使用服务做些什么
        
        $logger->debug('Job processed', ['id' => $id]);
    }
}
```

> **注意**
> 将处理器定义为单例以获得更好的性能。

重要的是要注意，`invoke` 方法必须始终具有 `void` 返回类型，因为它不返回任何值。

## 任务负载序列化

队列组件支持使用序列化程序将对象转换为适合存储在队列中的序列化形式，并从该形式转换回来。这允许您轻松地将复杂的对象排队和出队，而无需手动序列化和反序列化它们。

> **另请参阅**
> [序列化程序组件](../advanced/serializer.md) 用于在将任务负载添加到队列时对其进行序列化，并在从队列中检索任务负载并将其传递给任务处理器进行处理时进行反序列化。

### 配置默认序列化程序

队列组件的默认序列化程序可以通过 `queue.php` 配置文件指定。

**示例：**

```php app/config/queue.php
use Spiral\Core\Container\Autowire;
use Spiral\Serializer\Serializer\JsonSerializer;
use Spiral\Serializer\Serializer\PhpSerializer;

return [
    // 通过序列化程序名称
    'defaultSerializer' => 'json',

    // 通过类名
    'defaultSerializer' => JsonSerializer::class,
    
    // 通过实例
    'defaultSerializer' => new JsonSerializer(),
    
    // 通过自动装配
    'defaultSerializer' => new Autowire(PhpSerializer::class)
];
```

> **注意**
> 这允许您轻松地自定义队列的序列化策略，并选择最适合您需求的策略。在 [组件 — 序列化程序](../advanced/serializer.md) 中阅读更多关于可用序列化程序的内容。

### 更改序列化程序

有几种方法可以更改序列化程序。您可以全局更改应用程序的默认序列化程序。或者您可以为任务类型设置特定的序列化程序。特定的序列化程序由 `Spiral\Serializer\SerializerRegistryInterface` 选择。

您可以使用 `Spiral\Queue\Attribute\Serializer` 属性配置特定任务类型的序列化程序：

```php
use Spiral\Queue\Attribute\Serializer;
use Spiral\Queue\JobHandler;

#[Serializer('json')]
final class Ping extends JobHandler
{
    public function invoke(array $payload): void
    {
        // ...
    }
}
```

或者使用 `app/config/queue.php` 配置文件。

```php app/config/queue.php
use Spiral\Core\Container\Autowire;

return [
    'registry' => [
        'serializers' => [
            'ping.job' => 'json',
            TestJob::class => 'serializer',
            OtherJob::class => CustomSerializer::class,
            FooJob::class => new CustomSerializer(),
            BarJob::class => new Autowire(CustomSerializer::class),
        ]
    ],
];
```

或者，使用 `Spiral\Queue\QueueRegistry` 类的 `setSerializer` 方法注册序列化程序。

```php app/src/ApplicationBootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container\Autowire;
use Spiral\Queue\QueueRegistry;

class AppBootloader extends Bootloader
{
    public function boot(QueueRegistry $registry): void
    {
        $registry->setSerializer('ping.job', 'json');
        $registry->setSerializer(TestJob::class, 'serializer');
        $registry->setSerializer(OtherJob::class, CustomSerializer::class);
        $registry->setSerializer(FooJob::class, new CustomSerializer());
        $registry->setSerializer(BarJob::class, new Autowire(CustomSerializer::class));
    }
}
```

## 任务处理器注册表

如果您不想像以下示例那样使用任务处理器类名作为队列任务名称：

```php
use Spiral\Queue\QueueInterface;

public function createJob(QueueInterface $queue): void
{
    $queue->push('sample::job');
}
```

您需要告诉队列如何处理名为 `sample::job` 的任务。

您可以通过属性 `Spiral\Queue\Attribute\JobHandler` 来完成：

```php
use Spiral\Queue\Attribute\JobHandler as Handler;
use Spiral\Queue\JobHandler;

#[Handler('sample::job')]
final class Ping extends JobHandler
{
    public function invoke(array $payload): void
    {
        // ...
    }
}
```

或者通过 `app/config/queue.php` 配置：

```php app/config/queue.php
return [
    'registry' => [
        'handlers' => [
            'sample::job' => App\Endpoint\Job\SampleJob::class
        ],
    ],
];
```

或者通过 `Spiral\Queue\QueueRegistry`：

```php
use Spiral\Boot\Bootloader\Bootloader;

class MyBootloader extends Bootloader
{
    public function boot(\Spiral\Queue\QueueRegistry $registry): void
    {
        $registry->setHandler('sample::job', \App\Endpoint\Job\SampleJob::class);
    }
}
```

## 任务选项

`Spiral\Queue\Options` 类允许您在使用 `QueueInterface::push()` 方法将任务推送到队列时为任务指定附加上下文。

#### `withHeader(string $name, string|array $value)`

此方法允许您为任务设置标头值。标头可用于将有关任务的其他元数据传递给消费者服务器。

```php
$options = new \Spiral\Queue\Options();

$queue->push(
    SampleJob::class, 
    ['value' => 123], 
    $options->withHeader('user_id', 123)
);
```

#### `withQueue(?string $queue)`

此方法允许您指定应将任务推送到哪个队列的名称。如果未指定队列，则任务将被推送到默认队列。

```php
$options = new \Spiral\Queue\Options();

$queue->push(
    SampleJob::class, 
    ['value' => 123], 
    $options->withQueue('high_priority')
);
```

#### `withDelay(?int $delay)`

此方法允许您指定任务在可供处理之前以秒为单位的延迟时间。如果未指定延迟，则任务将在默认延迟时间段后处理。

```php
$options = new Options();

$queue->push(
    SampleJob::class, 
    ['value' => 123], 
    $options->withDelay(3600) // 任务将在 1 小时后可供处理
);
```

## 处理失败的任务

默认情况下，所有失败的任务都将发送到 Spiral 日志中。但您可以更改默认行为。首先，您需要为 `Spiral\Queue\Failed\FailedJobHandlerInterface` 创建自己的实现。

### 自定义处理器示例

```php app/src/Infrastructure/Queue/DatabaseFailedJobsHandler.php
namespace App\Infrastructure\Queue;

use Spiral\Queue\Failed\FailedJobHandlerInterface;
use Cycle\Database\DatabaseInterface;
use Spiral\Queue\SerializerInterface;

class DatabaseFailedJobsHandler implements FailedJobHandlerInterface
{
    private DatabaseInterface $database;
    private SerializerInterface $serializer;
    
    public function __construct(DatabaseInterface $database, SerializerInterface $serializer)
    {
        $this->database = $database;
        $this->serializer = $serializer;
    }

    public function handle(string $driver, string $queue, string $job, array $payload, \Throwable $e): void
    {
        $this->database
            ->insert('failed_jobs')
            ->values([
                'driver' => $driver,
                'queue' => $queue,
                'job_name' => $job,
                'payload' => $this->serializer->serialize($payload),
                'error' => $e->getMessage(),
            ])
            ->run();
    }
}
```

然后，您需要将您的实现绑定到 `Spiral\Queue\Failed\FailedJobHandlerInterface` 接口。

```php app/src/ApplicationBootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\RoadRunnerBridge\Queue\Failed\FailedJobHandlerInterface;

final class QueueFailedJobsBootloader extends Bootloader
{
    protected const SINGLETONS = [
        FailedJobHandlerInterface::class => \App\Infrastructure\Queue\DatabaseFailedJobsHandler::class,
    ];
}
```

并在您的应用程序中在 `QueueFailedJobsBootloader` 之后注册此启动加载程序

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\QueueFailedJobsBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动加载程序](../framework/bootloaders.md) 部分阅读更多关于启动加载程序的内容。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\QueueFailedJobsBootloader::class,
    // ...
];
```

在 [框架 — 启动加载程序](../framework/bootloaders.md) 部分阅读更多关于启动加载程序的内容。
:::

::::

## 重试失败的任务

在分布式系统中，由于网络错误等临时故障，有时任务会失败。Spiral 通过自动重试这些任务来确保这些任务不会丢失。

### 工作原理

1. **拦截器的作用：** 一个拦截器负责监督任务的执行。如果任务失败，它将更新任务的重试计数，并发出 `Spiral\Queue\Exception\RetryException` 信号。通过这种方式，该任务被标记为重试，并带有更新的详细信息。

2. **消费者的作用：** 当消费者遇到 `RetryException` 时，它知道该任务应该被放回队列中，并使用在异常中提供的新设置进行重试。

### 配置

要使此机制启动并运行，请在 `app/config/queue.php` 配置文件中进行一些调整。默认情况下，拦截器已开启。但是，如果您更改了配置中的 `interceptors` 部分，请记住包含 `Spiral\Queue\Interceptor\Consume\RetryPolicyInterceptor`，如下所示：

```php app/config/queue.php
use Spiral\Queue\Interceptor\Consume\RetryPolicyInterceptor;

return [    
    'interceptors' => [
        'push' => [
            // ...
        ],
        'consume' => [
            RetryPolicyInterceptor::class
        ],
    ],
    // ...
];
```

> **另请参阅**
> 在 [队列 — 拦截器](./interceptors.md) 部分阅读更多关于拦截器的内容。

### 使用

主要有两种方法可以使用此重试系统：

#### `Spiral\Queue\Attribute\RetryPolicy` 属性

您可以使用 `Spiral\Queue\Attribute\RetryPolicy` 属性来指定任务的重试策略，如下例所示：

```php app/src/Endpoint/Job/PingJob.php
use Spiral\Queue\Attribute\RetryPolicy;
use Spiral\Queue\JobHandler;

#[RetryPolicy(maxAttempts: 3, delay: 5, multiplier: 2)]
final class PingJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        // ...
    }
}
```

在这种情况下，如果任务失败，它将使用属性中指定的重试策略自动重新排队。

#### 使用自定义异常

您可以在您的异常类中实现 `Spiral\Queue\Exception\RetryableExceptionInterface` 接口，并从您的任务处理器中抛出它，如下例所示：

```php app/src/Exception/Job/RetryableException.php
use Spiral\Queue\Exception\RetryableExceptionInterface;
use Spiral\Queue\RetryPolicy;

class RetryableException extends \Exception implements RetryableExceptionInterface
{
    public function isRetryable(): bool
    {
        return true;
    }

    public function getRetryPolicy(): ?RetryPolicyInterface
    {
        return new RetryPolicy(
            maxAttempts: 3,
            delay: 5,
            multiplier: 2
        );
    }
}
```

然后从您的任务处理器中抛出它：

```php app/src/Endpoint/Job/PingJob.php
use Spiral\Queue\JobHandler;

final class PingJob extends JobHandler
{
    public function invoke(array $payload): void
    {
        // ...
        
        throw new RetryableException('Something went wrong');
    }
}
```

如果此任务遇到自定义异常，它将知道根据您设置的条件进行重试。

正如您所看到的，Spiral 提供了一个强大的重试机制，易于配置并适应特定需求。通过利用内置的 RetryPolicy 属性或创建自定义异常，您可以有效地指示如何重试任务。

## 事件

| 事件                            | 描述                                                     |
|----------------------------------|----------------------------------------------------------|
| Spiral\Queue\Event\JobProcessing | 该事件将在 `before` 执行任务处理器时触发。                |
| Spiral\Queue\Event\JobProcessed  | 该事件将在任务处理器执行 `after` 后触发。                |

> **注意**
> 要了解更多关于派发事件的信息，请参阅我们文档中的 [事件](../advanced/events.md) 部分。
