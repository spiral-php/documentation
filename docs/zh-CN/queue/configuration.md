# 队列 — 入门

Spiral 提供了对 PHP 后台处理和队列系统的支持。该特性开箱即用，允许您使用各种消息代理。

确保将 `Spiral\Queue\Bootloader\QueueBootloader` 添加到您的应用程序内核中。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Queue\Bootloader\QueueBootloader::class,
        // ...
    ];
}
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloaders 的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Queue\Bootloader\QueueBootloader::class,
    // ...
];
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloaders 的信息。
:::

::::

## 配置

默认的队列配置可以在 `app/config/queue.php` 文件中找到。此配置文件包含针对每个队列驱动程序的选项集以及别名，这些别名允许您为应用程序指定一个默认的队列驱动程序来使用。

```php app/config/queue.php
return [
    /** 默认队列连接名称 */
    'default' => env('QUEUE_CONNECTION', 'sync'),

    /** 队列连接的别名，如果您想使用特定于域的队列 */
    'aliases' => [
        // 'mailQueue' => 'null',
        // 'ratingQueue' => 'sync',
    ],
    
    'connections' => [
        'sync' => [
            // 作业将立即处理，无需排队
            'driver' => 'sync',
        ],
        'null' => [
            // 什么都不做
            'driver' => 'null',
        ],
    ],
    
    'registry' => [
        'handlers' => [
            'sample::job' => App\Jobs\SampleJob::class
        ],
        'serializers' => [
            ObjectJob::class => 'json',
        ]
    ],
    
    'driverAliases' => [
        'sync' => \Spiral\Queue\Driver\SyncDriver::class,
        'null' => \Spiral\Queue\Driver\NullDriver::class,
    ],
];
```

### 声明连接

要在您的应用程序中创建一个新的队列连接，您可以在 `queue` 配置文件的 `connections` 部分添加一个新节。

### 别名

队列别名是一个特性，允许您的应用程序和模块以多种方式访问队列系统，使用与单个物理队列相关的独立连接。

```php app/config/queue.php
return [
    'aliases' => [
        'mailQueue' => 'roadrunner',
        'ratingQueue' => 'sync',
    ],
];
```

要通过其名称或别名获取队列实例，您可以使用 `Spiral\Queue\QueueConnectionProviderInterface` 的 `getConnection` 方法。此方法接受一个字符串参数，表示所需队列连接的名称，并返回 `Spiral\Queue\QueueInterface` 的一个实例。

例如，如果您想获取名为 `mailQueue` 的名称或别名的队列实例，您可以使用以下代码：

```php
use Spiral\Queue\QueueConnectionProviderInterface;

$container->bind(MyService::class, function(QueueConnectionProviderInterface $provider) {
    return new MyService($provider->getConnection('mailQueue'));
})
```

```php
final class MyService
{
    public function __construct(
        private readonly QueueInterface $queue
    ) {
    }
}
```

### 作业处理器 (Job handlers)

您可以为您的作业类注册处理器，以关联应该为特定作业处理哪个处理器。

```php app/config/queue.php
'registry' => [
    'handlers' => [
        'sample::job' => App\Jobs\SampleJob::class
    ],
],
```

## 自定义驱动程序

如果可用的队列驱动程序不能满足您的特定需求，您可以通过实现 `Spiral\Queue\QueueInterface` 接口来创建您自己的自定义驱动程序。

这是一个假队列驱动程序的示例实现：

```php app/src/Infrastructure/Queue/RedisQueue.php
namespace App\Infrastructure\Queue;

use Spiral\Queue\SerializerRegistryInterface;
use Spiral\Queue\QueueInterface;

final class RedisQueue implements QueueInterface
{
    public function __construct(
        private readonly SerializerRegistryInterface $redis,
        private readonly Redis $redis,
        private readonly string $server,
        private readonly string $queueName,
    ) {
    }

    public function push(string $name, array $payload = [], OptionsInterface $options = null): string
    {
        $payload = $this->serializerRegistry->getSerializer($name)->serialize($payload);
        
        $id = // generate job id
        
        // Push job to the redis broker and return job id
        $this->redis->...
        
        return $id;
    }
}
```

定义完自定义驱动程序后，您可以在配置文件中注册它：

```php app/config/queue.php
'connections' => [
    'mail' => [
        'driver' => \App\Infrastructure\Queue\RedisQueue::class,
        'server' => 'redis://localhost:6379',
        'queueName' => 'mail',
    ],
],
```

因此，当您在应用程序中使用 `mail` 连接时，Spiral IoC 容器将创建 `RedisQueue` 类的实例，并将 `redis://localhost:6379` 和 `mail` 值分别传递给 `$server` 和 `$queueName` 参数。

您也可以将您的驱动程序注册为队列配置文件的 `driverAliases` 部分中的别名。

```php app/config/queue.php
'driverAliases' => [
    'redis' => \App\Infrastructure\Queue\RedisQueue::class,
    // ...
],
```

或者通过 Bootloader 注册驱动程序别名：

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use App\Infrastructure\Queue\RedisQueue;
use Spiral\Queue\Bootloader\QueueBootloader;

class AppBootloader extends Bootloader
{
    public function init(QueueBootloader $queue): void
    {
        $queue->registerDriverAlias(RedisQueue::class, 'redis');
    }
}
```

> **另请参阅**
> 在 [队列 — 运行作业](./jobs.md) 部分阅读更多关于作业处理器 (job handler) 的信息。
