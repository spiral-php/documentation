# 配置 RoadRunner 队列管道

作业队列已成为现代 PHP 应用程序中不可或缺的组件，处理复杂、资源密集型任务，并显著提升应用程序性能。

一种利用 Go for PHP 应用程序潜力的解决方案是 RoadRunner，与 Spiral 框架结合使用。本教程将指导您在 Spiral 应用程序中配置 RoadRunner 的队列管道，提供高性能、强大的队列服务解决方案。

> **注意**
> 本教程涵盖了组件和方法的基础知识。有关更详细的信息，我们建议参考相关章节。

我们的应用程序将由两种类型的应用程序组成：

1.  **生产者 (Producer)** - 将作业推送到队列中
2.  **消费者 (Consumer)** - 将接收排队的任务并处理它们

## 生产者

要开始构建 **生产者 (producer)** 应用程序，您可以通过运行以下命令轻松安装默认的 `spiral/app` 包，其中包含了大多数必需的组件：

```terminal
composer create-project spiral/app my-app
```

在安装过程中，您将被提示使用 Spiral 安装程序选择各种选项，例如应用程序预设、是否使用 Cycle ORM、使用哪个集合、使用哪个验证器组件等等。 在我们的示例中，我们将使用 CLI 应用程序，该应用程序将从控制台命令将任务推送到队列中。

对于本教程，我们建议选择上面显示的选项：

```output
✔ 您想安装哪个应用程序预设？ > Cli
✔ 创建默认的应用程序结构和演示数据？ > 否
✔ 您需要 Cycle ORM 吗？ > 否
✔ 您想使用队列组件吗？ > 是
✔ 您想使用缓存组件吗？ > 否
✔ 您想使用邮件程序组件吗？ > 否
✔ 您想使用存储组件吗？ > 否
✔ 您需要 RoadRunner 吗？ > 是
✔ 您需要 RoadRunner 监控指标吗？ > 否
✔ 您需要 Temporal 吗？ > 否
```

安装完成后，您需要配置 RoadRunner 队列管道，您将在其中推送您的作业。

首先，您需要配置 RoadRunner 服务器以使用 `jobs` 插件：

```yaml .rr.yaml
amqp:
  addr: amqp://guest:guest@127.0.0.1:5672

jobs:
  consume: [ ]
  pipelines: { }
```

> **注意**
> 您可以在 [官方文档](https://roadrunner.dev/docs/queues-overview) 中阅读有关 RoadRunner Jobs 插件配置的更多信息

如您所见，我们在配置中没有指定任何管道。 RoadRunner 提供了动态创建管道的功能，所以我们稍后将在应用程序中创建它们。

配置完成后，您可以启动服务器。 RoadRunner 使用 [RPC](https://roadrunner.dev/docs/php-rpc/2023.x/en) 在 PHP 应用程序和 RoadRunner 之间进行通信，因此在将作业推送到队列之前，我们需要启动它。

让我们使用以下命令检查一切是否正常工作：

```terminal
./rr serve
```

### 配置

Spiral 应用程序的配置是通过位于 `app/config` 目录中的配置文件完成的。

:::: tabs

::: tab PHP

让我们在 `app/config/queue.php` 文件中定义我们的第一个管道：

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\AMQPCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'pipelines' => [
         'default' => [
             'connector' => new AMQPCreateInfo(
                  name: 'default',
                  priority: 100,
                  queue: 'default',
             ),
             // 在启动时不消耗此管道的作业
             'consume' => false,
         ],
    ],
    
    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'pipeline' => 'default',
        ],
    ],
];
```

> **注意**
> 您可以在 [队列 — RoadRunner 集成](../queue/roadrunner.md) 章节中阅读有关 roadrunner 队列配置的更多信息。

:::

::: tab YAML

让我们在 `.rr.yaml` 文件中定义我们的第一个管道：

```yaml
amqp:
  addr: amqp://guest:guest@127.0.0.1:5672

jobs:
  consume: [ ]
  pipelines:
    default:
      driver: amqp
      priority: 100
      queue: default
```

并在 `app/config/queue.php` 文件中选择已定义的管道：

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\AMQPCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'pipelines' => [],
    
    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'pipeline' => 'default',
        ],
    ],
];
```

> **注意**
> 您可以在 [队列 — RoadRunner 集成](../queue/roadrunner.md) 章节中阅读有关 roadrunner 队列配置的更多信息。

:::

::::

当您运行 `./rr serve` 命令时，RoadRunner 将创建一个名为 `default` 的管道，并将默认使用 `roadrunner` 连接将作业推送到队列中。

### 推送作业

让我们为其添加一些逻辑：

:::: tabs

::: tab 数组负载

```php app/src/Endpoint/Console/PingCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Command;
use Spiral\Queue\QueueInterface;

#[AsCommand(name: 'ping')]
final class PingCommand extends Command
{
    public function __invoke(QueueInterface $queue): int
    {
        $queue->push('ping', [
            'url' => 'https://spiral.dev',
        ]);

        return self::SUCCESS;
    }
}
```

如果我们使用 `array` 负载，我们可以使用简单的序列化器，例如 `json`。

> **注意**
> 在 [组件 — 序列化器](../advanced/serializer.md) 章节中阅读有关可用序列化器的更多信息。

让我们在 `app/config/queue.php` 文件中定义一个默认序列化器：

```php app/config/queue.php
return [
    // ...
    'defaultSerializer' => 'json',
];
```

> **注意**
> 在 [队列 — 运行作业](../queue/jobs.md#job-payload-serialization) 章节中阅读有关作业负载序列化的更多信息。

:::

::: tab 对象负载

Queue 组件提供了使用复杂对象作为负载的功能。要使用对象负载，我们需要首先安装 `spiral-packages/symfony-serializer` 包。 这将有助于序列化我们的负载：

```terminal
composer require spiral-packages/symfony-serializer
```

安装包后，您需要注册启动器：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Serializer\Symfony\Bootloader\SerializerBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动器](../framework/bootloaders.md) 章节中阅读有关启动器的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Serializer\Symfony\Bootloader\SerializerBootloader::class,
    // ...
];
```

在 [框架 — 启动器](../framework/bootloaders.md) 章节中阅读有关启动器的更多信息。
:::

::::

并在 `app/config/queue.php` 文件中定义默认序列化器：

```php app/config/queue.php
return [
    // ...
    'defaultSerializer' => 'symfony-json',
];
```

> **注意**
> 在 [队列 — 运行作业](../queue/jobs.md#job-payload-serialization) 章节中阅读有关作业负载序列化的更多信息。

就这样！现在您可以使用对象负载将作业推送到队列中，它将自动被序列化并作为 JSON 字符串发送到队列。

让我们首先创建一个 DTO 类，该类将携带我们需要的所有数据：

```php app/src/DTO/Ping.php
namespace App\DTO;

final class Ping 
{
    public function __construct(
        public readonly string $url,
    ) {}
}
```

现在我们可以使用它作为负载将作业推送到队列中：

```php app/src/Endpoint/Console/PingCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Command;
use Spiral\Queue\QueueInterface;
use App\DTO\PingDTO;

#[AsCommand(name: 'ping')]
final class PingCommand extends Command
{
    public function __invoke(QueueInterface $queue): int
    {
        $queue->push('ping', new Ping(url: 'https://spiral.dev'));

        return self::SUCCESS;
    }
}
```

:::

::::

一旦我们创建了一个控制台命令 `ping`，我们就可以将一个作业推送到队列中。

首先，我们需要启动 RoadRunner 服务器：

```terminal
./rr serve
```

然后运行我们的命令：

```terminal
php app.php ping
```

## 消费者

要开始构建 **消费者 (consumer)** 应用程序，您可以通过运行以下命令轻松安装默认的 `spiral/app` 包，其中包含了大多数必需的组件：

```terminal
composer create-project spiral/app my-consumer-app
```

对于消费者，我们还需要 `Queue 组件` 和 `RoadRunner`。其他组件是可选的，您可以在安装过程中选择您需要的组件。

```output
✔ 您想安装哪个应用程序预设？ > Cli
✔ 创建默认的应用程序结构和演示数据？ > 否
✔ 您想使用队列组件吗？ > 是
✔ 您需要 RoadRunner 吗？ > 是
```

安装完成后，您需要配置 RoadRunner 队列管道，您将在其中推送您的作业。

首先，您需要配置 RoadRunner 服务器以使用 `jobs` 插件：

```yaml .rr.yaml
amqp:
  addr: amqp://guest:guest@127.0.0.1:5672

jobs:
  consume: [ ]
  pipelines: { }
```

> **注意**
> `amqp` 部分应与生产者应用程序中的相同。 **消费者和生产者应该使用相同的 AMQP 服务器。**

### 配置

:::: tabs

::: tab PHP

让我们在 `app/config/queue.php` 文件中定义我们的管道：

```php app/config/queue.php
use Spiral\RoadRunner\Jobs\Queue\AMQPCreateInfo;

return [
    'default' => env('QUEUE_CONNECTION', 'roadrunner'),

    'pipelines' => [
         'default' => [
             'connector' => new AMQPCreateInfo(
                  name: 'default',
                  priority: 100,
                  queue: 'default',
             ),
             'consume' => true, // <===== 启用消费
         ],
    ],
    
    'connections' => [
        'roadrunner' => [
            'driver' => 'roadrunner',
            'pipeline' => 'default',
        ],
    ],
];
```

> **注意**
> 消费者和生产者配置之间的唯一区别是，消费者应该将 `consume` 选项设置为 `true`。 在这种情况下，RoadRunner 将自动从 AMQP 服务器消费作业。

:::

::: tab YAML

让我们在 `.rr.yaml` 文件中定义我们的管道：

```yaml
amqp:
  addr: amqp://guest:guest@127.0.0.1:5672

jobs:
  consume: [ default ]  # <===== 启用消费
  pipelines:
    default:
      driver: amqp
      priority: 100
      queue: default
```

> **注意**
> 消费者和生产者配置之间的唯一区别是，消费者应该将 `jobs.consume` 选项设置为包含 `default` 管道名称。 在这种情况下，RoadRunner 将自动从 AMQP 服务器消费作业。

:::

::::

### 作业

当一个作业将被消费时，它将被传递给作业处理程序类，该类具有处理它的所有逻辑。

> **注意**
> 在 [队列 — 作业处理程序](../queue/jobs.md) 章节中阅读有关作业处理程序的更多信息。

让我们使用脚手架创建我们的第一个作业：

```terminal
php app.php create:jobHandler Ping
```

> **注意**
> 您可以在 [基础知识 — 脚手架](../basics/scaffolding.md) 章节中阅读有关脚手架的更多信息。

运行此命令后，您将看到以下输出：

```output
声明 'PingJob' 已成功写入 '~/my-app/app/src/Endpoint/Job/PingJob.php'。
```

我们刚刚创建了一个作业处理程序，它将用于处理我们的作业。

让我们为其添加一些逻辑：

:::: tabs

::: tab 数组负载

如果您使用 JSON 负载，则可以使用简单的序列化器（例如 `json`）来反序列化从队列接收的负载。

让我们在 `app/config/queue.php` 文件中定义一个默认序列化器：

```php app/config/queue.php
return [
    // ...
    'defaultSerializer' => 'json',
];
```

现在让我们向作业处理程序添加一些逻辑：

```php app/src/Endpoint/Job/PingJob.php
namespace App\Endpoint\Job;

use Psr\Log\LoggerInterface;
use Spiral\Queue\JobHandler;
use App\DTO\Ping;

final class PingJob extends JobHandler
{
    public function invoke(
        LoggerInterface $logger, 
        string $id, 
        Ping $payload, 
        array $headers,
    ): void {
        $logger->info('Ping job received', [
            'id' => $id,
            'url' => $payload->url,
            'headers' => $headers,
        ]);
    }
}
```

我们的作业处理程序将仅记录从队列接收的所有数据。

:::

::: tab 对象负载

Spiral Queue 组件提供了多种方法来理解应该使用哪个类来反序列化负载。

1.  如果将一个对象推送到队列中，则组件还将发送一个类名作为标头 `x-job-class`。 在这种情况下，您可以使用我们上面提到的 `spiral-packages/symfony-serializer` 包来反序列化负载。
2.  如果没有 `x-job-class` 标头，但您在 `invoke` 方法中为 `payload` 参数指定了类类型，则组件将尝试使用指定的类型来反序列化负载。

> **警告**
> 请注意，如果您使用标头 `x-job-class` 来指定类名，则应该在生产者和消费者应用程序中拥有具有相同命名空间的此类。 也许您需要考虑使用共享存储库来在应用程序之间共享您的 DTO。

让我们在 `app/config/queue.php` 文件中定义一个默认序列化器：

```php app/config/queue.php
return [
    // ...
    'defaultSerializer' => 'symfony-json',
];
```

并创建一个类，我们将在此类中反序列化我们的负载：

```php app/src/DTO/Ping.php
namespace App\DTO;

final class Ping 
{
    public function __construct(
        public readonly string $url,
    ) {}
}
```

现在让我们向作业处理程序添加一些逻辑：

```php app/src/Endpoint/Job/PingJob.php
namespace App\Endpoint\Job;

use Psr\Log\LoggerInterface;
use Spiral\Queue\JobHandler;

final class PingJob extends JobHandler
{
    public function invoke(
        LoggerInterface $logger, 
        string $id, 
        array $payload,
        array $headers,
    ): void {
        $logger->info('Ping job received', [
            'id' => $id,
            'payload' => $payload,
            'headers' => $headers,
        ]);
    }
}
```

:::

::::

当我们使用 `push` 方法将作业推送到队列中时，我们在第一个参数中指定了一个任务名称 `ping`。 现在我们需要告诉消费者，如果它接收到具有此任务名称的作业，则应该由 `PingJob` 处理程序处理它。

让我们在 `app/config/queue.php` 文件中将我们的作业处理程序与任务名称连接起来：

```php app/config/queue.php
return [
    // ...
    'registry' => [
        'handlers' => [
            'ping' => App\Endpoint\Job\PingJob::class
        ],
    ],
];
```

> **注意**
> 您可以在 [队列 — 作业处理程序](../queue/jobs.md#job-handler-registry) 章节中阅读有关作业处理程序注册表的更多信息。

完成所有这些步骤后，我们就可以从队列中消费作业了。

让我们运行 RoadRunner 服务器：

```terminal
./rr serve
```

并从生产者应用程序将作业推送到队列中：

```terminal
php app.php ping
```

### 重试策略

让我们想象一下，我们有一个作业，如果它失败，应该重试。 例如，我们有一个作业，该作业向远程服务器发送请求。 如果服务器不可用，我们应该在一段时间后重试此作业。

在这种情况下，我们可以使用作业标头和队列拦截器来实现重试策略。

> **注意**
> 在 [队列 — 拦截器](../queue/interceptors.md) 章节中阅读有关队列拦截器的更多信息。

让我们创建一个拦截器，该拦截器将捕获作业处理程序中的所有异常，并在一段时间后尝试重试它：

```php app/src/Endpoint/Job/Interceptor/RetryPolicyInterceptor.php
namespace App\Endpoint\Job\Interceptor;

use Carbon\Carbon;
use Psr\Log\LoggerInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Exceptions\ExceptionReporterInterface;
use Spiral\Queue\Exception\FailException;
use Spiral\Queue\Exception\RetryException;
use Spiral\Queue\Options;

final class RetryPolicyInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
        private readonly ExceptionReporterInterface $reporter,
        private readonly int $maxAttempts = 3,
        private readonly int $delayInSeconds = 5,
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        try {
        
            // 尝试执行作业处理程序
            return $core->callAction($controller, $action, $parameters);
            
        } catch (\Throwable $e) {
        
            // 报告异常
            $this->reporter->report($e);
            
            $headers = $parameters['headers'] ?? [];
            
            // 从标头获取尝试次数，或者如果这是第一次尝试，则使用最大尝试次数
            $attempts = (int)($headers['attempts'] ?? $this->maxAttempts);
            
            // 如果尝试次数已用完，则抛出 FailException
            if ($attempts === 0) {
                $this->logger->warning('Job handling failed: ['.$e->getMessage().']');
                
                throw new FailException($e->getMessage(), $e->getCode(), $e);
            }

            throw new RetryException(
                reason: $e->getMessage(),
                options: (new Options())->withDelay($this->delay)->withHeader('attempts', (string)($attempts - 1))
            );
        }
    }
}
```

现在我们需要在 `app/config/queue.php` 文件中注册我们的拦截器：

```php app/config/queue.php
use App\Endpoint\Job\Interceptor\RetryPolicyInterceptor;

return [    
    // ...
    'interceptors' => [
        'consume' => [
            RetryPolicyInterceptor::class,
        ],
    ],
];
```

现在，如果我们的作业处理程序失败，它将在 5 秒后重试。 经过 3 次尝试后，该作业将被标记为失败。

## 想了解更多？释放高级工作流编排的力量

Spiral 框架提供了与 [Temporal](https://temporal.io/) 的集成，Temporal 是一个强大的工作流编排工具，允许您构建复杂的工作流。 现在，如果您熟悉 RoadRunner 等队列服务，那么您将会得到一个惊喜，因为 Temporal IO 将工作流管理提升到了一个全新的水平。 就像拥有超能力，可以以简单而优雅的方式处理复杂的工作流。

在 PHP 开发领域，我们经常发现自己需要同时处理需要按特定顺序执行的各种任务和流程。 这就是 Temporal 闪耀的地方。 它允许我们以一种易于理解和维护的方式编写富有表现力和强大的工作流。

### 让我们深入研究一个展示 Temporal 魅力的例子

想象一下，您有一个每月处理用户订阅的任务。 使用 Temporal，它变成了一个直接的过程。 这是一个简单的例子来说明它：

```php
<?php
/**
 * 此文件是 Temporal 包的一部分。
 *
 * 有关完整的版权和许可信息，请查看 LICENSE
 * 文件，该文件与此源代码一起分发。
 */
declare(strict_types=1);

namespace Temporal\Samples\Subscription;

use Carbon\CarbonInterval;
use Temporal\Activity\ActivityOptions;
use Temporal\Exception\Failure\CanceledFailure;
use Temporal\Workflow;

/**
 * 演示一个长期运行的进程来表示用户订阅工作流。
 */
class SubscriptionWorkflow implements SubscriptionWorkflowInterface
{
    private $account;

    // 工作流逻辑在这里...

    public function subscribe(string $userID)
    {
        yield $this->account->sendWelcomeEmail($userID);

        try {
            $trialPeriod = true;
            while (true) {
                // 降低周期持续时间以观察工作流行为
                yield Workflow::timer(CarbonInterval::days(30));

                if ($trialPeriod) {
                    yield $this->account->sendEndOfTrialEmail($userID);
                    $trialPeriod = false;
                    continue;
                }

                yield $this->account->chargeMonthlyFee($userID);
                yield $this->account->sendMonthlyChargeEmail($userID);
            }
        } catch (CanceledFailure $e) {
            yield Workflow::asyncDetached(
                function () use ($userID) {
                    yield $this->account->processSubscriptionCancellation($userID);
                    yield $this->account->sendSorryToSeeYouGoEmail($userID);
                }
            );
        }
    }
}
```

在此示例中，`subscribe` 方法表示管理每月订阅的工作流逻辑。 魔法在于 `Workflow::timer` 函数，它允许您为循环的每次迭代安排一个特定的持续时间。

通过将计时器设置为 `CarbonInterval::months(1)`，您可以确保每月执行订阅任务。 Temporal 负责调度和协调，从而免除了您手动管理的麻烦。

此外，Temporal 还提供了内置的容错性和可扩展性。 如果发生异常，例如指示订阅取消的 `CanceledFailure`，您可以在工作流中优雅地处理它。

使用 Temporal，管理每月订阅等复杂周期性工作流变得轻而易举。
