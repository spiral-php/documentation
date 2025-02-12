# 队列 — RoadRunner 集成

Spiral 框架与 [RoadRunner jobs](https://roadrunner.dev/docs/queues-overview) 插件完全集成，该插件为各种队列代理提供了统一的队列 API，例如：

- [Kafka](https://roadrunner.dev/docs/queues-kafka),
- [AMQP (RabbitMQ)](https://roadrunner.dev/docs/queues-amqp),
- [Amazon SQS](https://roadrunner.dev/docs/queues-sqs),
- [Beanstalk](https://roadrunner.dev/docs/queues-beanstalk),
- 以及 [memory](https://roadrunner.dev/docs/queues-memory) 驱动。

> **注意**
> 有关支持的代理的更多信息，请访问 [官方网站](https://roadrunner.dev/docs/plugins-jobs)。

为了实现与 RoadRunner 的集成，Spiral 框架通过 [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) 包提供了内置支持。

> **警告**
> `spiral/roadrunner-bridge` 包的实际版本是 `3.0`。 如果您使用的是较旧版本的软件包，本文档中描述的某些功能可能不可用。

## 安装

要开始，您需要安装 [Roadrunner bridge](../start/server.md#roadrunner-bridge) 包。 安装后，将 `Spiral\RoadRunnerBridge\Bootloader\QueueBootloader` 添加到您的 Kernel 类的引导程序列表中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\QueueBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\QueueBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::::

通过这样做，将自动注册一个名为 `roadrunner` 的附加队列驱动程序。 然后，您可以在您的 `app/config/queue.php` 配置文件中为 RoadRunner 添加一个新的队列连接。

## 配置

RoadRunner 提供了两种声明管道的方法：

- 使用 `.rr.yaml` 配置文件来声明管道和代理。
- 在您的应用程序的 `app/config/queue.php` 中动态声明管道。

### 在 `.rr.yaml` 中声明管道

这是声明管道的最常用方法。 但通过这种方式，您只能声明静态管道。

> **注意**
> 如果您需要声明动态管道，则应考虑使用第二种方法。

**这是一个简单的例子：**

```yaml .rr.yaml
amqp:
  addr: amqp://guest:guest@localhost:5672

jobs:
  consume: [ "in-memory", "high-priority" ]
  pipelines:
    in-memory:
      driver: memory
      config:
        priority: 10
    high-priority:
      driver: amqp
      config:
        priority: 1
```

在这种情况下，您的 `app/config/queue.php` 配置文件应如下所示：

```php app/config/queue.php
return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'pipeline' => 'in-memory', // Pipeline name from .rr.yaml
        ],
    ],
];
```

### 在配置文件中声明管道

当您需要声明动态管道或只想将所有配置保存在一个地方时，此方法很有用。

**这种方法有一些好处：**

- **灵活性**：通过动态管道声明，您可以根据应用程序的当前状态即时创建管道。
- **控制**：您可以更好地控制应用程序中管道的创建和管理。 您可以仅在需要时创建管道，并在不再需要时将其删除，从而减少不必要的开销。

您可以创建动态管道以跨多个队列代理分配工作负载或平衡跨多个工作进程的负载。 这在需要跨多个服务器或实例分配作业处理的大型应用程序中可能很有用。

**这是一个简单的例子：**

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\MemoryCreateInfo;
use Spiral\RoadRunner\Jobs\Queue\AMQPCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'rr-amqp'),

    'pipelines' => [
         'high-priority' => [
             'connector' => new AMQPCreateInfo(
                  name: 'high-priority',
                  priority: 1,
                  queue: 'default',
             ),
             'consume' => true, // Consume jobs for this pipeline on startup
         ],
         'low-priority' => [
             'connector' => new AMQPCreateInfo(
                  name: 'low-priority',
                  priority: 100,
                  queue: 'default',
             ),
             'consume' => false, // Do not consume jobs for this pipeline on startup
         ],
         'in-memory' => [
            'connector' => new MemoryCreateInfo(name: 'local'),
            'consume' => true, 
         ]
    ],
            
    'connections' => [
        // ...
        'rr-amqp' => [
            'driver' => 'roadrunner',
            'pipeline' => 'low-priority',
        ],
        'rr-memory' => [
            'driver' => 'roadrunner',
            'pipeline' => 'in-memory',
        ],
    ],
];
```

> **警告**
> 所有定义了 `'consume' => true` 的管道都将在应用程序启动期间初始化。 这意味着推送到这些管道的所有作业都将立即被消费。 所有定义了 `'consume' => false` 的管道将在您将作业推送到它们时才会被初始化。

在某些情况下，您可能需要创建一个具有自定义默认选项或选项是强制性的管道。 例如，您可能希望为 Kafka 代理创建一个管道，该管道需要设置其他选项。

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\KafkaCreateInfo;
use Spiral\RoadRunner\Jobs\KafkaOptions;

'pipelines' => [
    'event-bus' => [
        'connector' => new KafkaCreateInfo(name: 'kafka', topic: 'events', ...),
        'options' => new KafkaOptions(topic: 'events', ...),
        'consume' => true, 
    ]
],
```

## 用法

默认情况下，当您使用 `roadrunner` 驱动程序将作业推送到队列时，该作业将被推送到在您的 `app/config/queue.php` 配置文件中定义的默认管道。

```php
'rr-amqp' => [
    'driver' => 'roadrunner',
    'pipeline' => 'low-priority',
],
```

如果您想将作业推送到特定的管道，您可以使用 `Spiral\Queue\Options::onQueue` 方法指定管道名称。

```php
use App\Endpoint\Job\SampleJob;
use Spiral\Queue\QueueInterface;
use Spiral\Queue\Options;

public function createJob(QueueInterface $queue): void
{
    $queue->push(
        SampleJob::class, 
        ['value' => 123],
        Options::onQueue('high-priority')
    );
}
```

### 选项

Spiral 队列组件提供了 `Spiral\Queue\Options` 类，但在某些情况下，您可能需要使用特定于 roadrunner 驱动程序的选项。 在这种情况下，您可以使用实现 `Spiral\RoadRunner\Jobs\OptionsInterface` 的选项。

例如，如果您想将其他选项传递给 Kafka 代理，您可以使用 `Spiral\RoadRunner\Jobs\KafkaOptions` 类。

```php
use App\Endpoint\Job\SampleJob;
use Spiral\Queue\QueueInterface;
use Spiral\RoadRunner\Jobs\KafkaOptions;

public function createJob(QueueInterface $queue): void
{
    $queue->push(
        SampleJob::class, 
        ['value' => 123],
        new KafkaOptions(topic: 'events')
    );
}
```

## 自定义驱动程序

在某些情况下，您可能需要为特定的队列代理创建自定义驱动程序。

例如，您可能想为特定的队列代理创建一个驱动程序。

```php app/src/Infrastructure/Queue/KafkaQueue.php
namespace App\Infrastructure\Queue;

use Spiral\Queue\OptionsInterface;
use Spiral\Queue\QueueInterface;
use Spiral\RoadRunner\Jobs\JobsInterface;
use Spiral\RoadRunner\Jobs\Queue\KafkaCreateInfo;
use Spiral\RoadRunner\Jobs\KafkaOptions;

final class KafkaQueue imlements QueueInterface
{
    private readonly QueueInterface $queue;
    
    public function __construct(JobsInterface $jobs, string $name, string $topic) 
    {
        $this->queue = $jobs->create(
            new KafkaCreateInfo(
                name: $name, 
                topic: $topic
            ), 
            new KafkaOptions($topic)
        );
    }

    public function push(string $name, array $payload = [], OptionsInterface|KafkaOptions $options = null): string
    {
        return $this->queue->push($name, $payload, $options)->getId();
    }
}
```

就是这样。 现在您可以使用您的驱动程序：

```php app/config/queue.php
'connections' => [
    'mail' => [
        'driver' => \App\Infrastructure\Queue\KafkaQueue::class,
        'name' => 'mail',
        'topic' => 'mail',
    ],
],
```

> **另请参阅**
> 在 [Queue — Getting started](./configuration.md) 部分阅读有关创建自定义驱动程序的更多信息。
