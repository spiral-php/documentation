# 队列 — 拦截器

Spiral 为开发者提供了一种方式，通过使用拦截器来定制其作业处理管道的行为。拦截器是在推送或消费作业之前或之后执行的一段代码，它允许开发者介入作业处理管道来执行某些操作。

> **另请参阅**
> 在 [框架 — 拦截器](../framework/interceptors.md) 部分阅读更多关于拦截器的内容。

拦截器可用于多种用途，例如处理错误，向作业添加额外上下文，或者根据正在处理的作业执行其他操作。

要使用拦截器，您需要实现 `Spiral\Core\CoreInterceptorInterface` 接口。此接口要求您实现 `process` 方法。

## 用于推送的拦截器

这是一个在推送作业前后记录消息的简单拦截器的示例：

```php
namespace App\Application\Job\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Queue\OptionsInterface;

final class LogInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $logger,
    ) {
    }
    
    /**
     * @param array{options: ?OptionsInterface, payload: array} $parameters
     */
    public function process(string $name, string $action, array $parameters, CoreInterface $core): string
    {
        $this->logger->info('Job pushing...', [
            'name' => $name,
            'action' => $action,
        ]);
        
        $id = $core->callAction($name, $action, $parameters);
        
        $this->logger->info('Job pushed', [
            'id' => $id,
            'name' => $name,
            'action' => $action,
        ]);

        return $id;
    }
}
```

要使用此拦截器，您需要在配置文件 `app/config/queue.php` 中注册它。

```php app/config/queue.php
use App\Application\Job\Interceptor\LogInterceptor;

return [    
    'interceptors' => [
        'push' => [
            LogInterceptor::class,
        ],
        'consume' => [
            //...
        ],
    ],
    // ...
];
```

> **注意**
> `callAction` 方法将推送并返回 `id` 字符串。

## 用于消费的拦截器

要创建一个拦截器，您需要创建一个类并实现接口 `Spiral\Core\CoreInterceptorInterface`。`callAction` 方法将执行一个消费作业，它不返回任何内容。

让我们创建一个 `JobExceptionsHandlerInterceptor`。这是一个在取消作业之前提供 3 次尝试在消费者中执行作业的类。

```php
namespace App\Application\Job\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Exceptions\ExceptionReporterInterface;

final class JobExceptionsHandlerInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly ExceptionReporterInterface $reporter,
    ) {
    }
 
    public function process(string $name, string $action, array $parameters, CoreInterface $core): mixed
    {
        try {
            return $core->callAction($name, $action, $parameters);
        } catch (\Throwable $e) {
             $this->reporter->report($e);
        }
    }
}
```

要使用此拦截器，您需要在配置文件 `app/config/queue.php` 中注册它。

```php app/config/queue.php
use App\Application\Job\Interceptor\JobExceptionsHandlerInterceptor;
use Spiral\Core\Container\Autowire;

return [    
    'interceptors' => [
        'consume' => [
            new Autowire(JobExceptionsHandlerInterceptor::class),
        ],
        //...
    ],
    // ...
];
```

## 注册新拦截器

有几种方法可以添加新的拦截器。

### 通过配置

将其添加到配置文件 `app/config/queue.php`。

```php app/config/queue.php
use App\CustomInterceptor;
use Spiral\Core\Container\Autowire;

return [    
    /**
     * -------------------------------------------------------------------------
     *  拦截器列表
     * -------------------------------------------------------------------------
     */
     'interceptors' => [
        // 用于推送的拦截器
        'push' => [
            // 通过完全限定的类名
            CustomInterceptor::class,
        
            // 通过 Autowire
            new Autowire(CustomInterceptor::class),     
            
            // 通过拦截器实例
            new CustomInterceptor(),
        ],
        
        // 用于消费的拦截器
        'consume' => [
            // 通过完全限定的类名
            CustomInterceptor::class,
        
            // 通过 Autowire
            new Autowire(CustomInterceptor::class),     
            
            // 通过拦截器实例
            new CustomInterceptor(),
        ],
     ],
];
```

#### 通过 QueueBootloader

调用方法 `addPushInterceptor` 为 `push` 动作添加一个拦截器。或者，在 `Spiral\Queue\Bootloader\QueueBootloader` 类中调用方法 `addConsumeInterceptor` 为 `consume` 动作添加一个拦截器。

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Interceptor\CustomInterceptor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container\Autowire;
use Spiral\Queue\Bootloader\QueueBootloader;

class AppBootloader extends Bootloader
{
    public function boot(QueueBootloader $queue): void
    {
        // 通过完全限定的类名
        $queue->addPushInterceptor(CustomInterceptor::class);
        $queue->addConsumeInterceptor(CustomInterceptor::class);

        // 通过 Autowire
        $queue->addPushInterceptor(new Autowire(CustomInterceptor::class));
        $queue->addConsumeInterceptor(new Autowire(CustomInterceptor::class));
        
        // 通过拦截器实例
        $queue->addPushInterceptor(new CustomInterceptor());
        $queue->addConsumeInterceptor(new CustomInterceptor());
    }
}
```

## 示例

### 失败作业的重试策略

它为失败的作业提供了一种强大而灵活的重试机制。实现重试策略在分布式系统中至关重要，因为它确保了瞬时问题（例如网络故障或临时资源不可用）不会导致系统丢失重要消息或数据。

拦截器的主要职责是执行给定的作业并监视它是否出现任何异常。如果抛出异常，拦截器将记录错误，更新作业的尝试计数，并使用更新的尝试计数和延迟抛出一个 `RetryException`。这允许作业使用新信息重新排队，确保按照指定的策略重试。

当消费者遇到 RetryException 时，它知道应该使用异常中提供的更新选项将作业返回到队列。这使得作业可以根据指定的策略进行重试，从而提高了系统的整体可靠性，因为它允许作业从瞬时问题中恢复。

```php
declare(strict_types=1);

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
        private readonly int $delay = 5,
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        try {
            return $core->callAction($controller, $action, $parameters);
        } catch (\Throwable $e) {
            $this->reporter->report($e);
            
            $headers = $parameters['headers'] ?? [];
            $attempts = (int)($headers['attempts'] ?? $this->maxAttempts);
            if ($attempts === 0) {
                $this->logger->warning('Attempt to fetch package [%s] statistics failed', $controller);
                return null;
            }

            throw new RetryException(
                reason: $e->getMessage(),
                options: (new Options())->withDelay($this->delay)->withHeader('attempts', (string)($attempts - 1))
            );
        }
    }
}
```

拦截器将尝试处理作业指定次数（`$maxAttempts`），然后放弃。它还包括一个可配置的延迟（`$delay`），在每次重试尝试之间，这有助于避免系统中的重试风暴。

使用此拦截器的主要优点之一是，它使开发人员能够以集中、一致的方式处理异常并应用重试策略。这种方法可以使代码更简洁、更易于维护，因为它抽象了单个作业处理程序的错误处理逻辑。
