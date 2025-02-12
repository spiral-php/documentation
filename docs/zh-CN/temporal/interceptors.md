# Temporal — 拦截器

拦截器是一种用于修改传入和传出的 SDK 调用的机制。拦截器通常用于在工作流（Workflows）和活动（Activities）的调度和执行中添加跟踪和授权。你可以将它们比作其他框架中的“中间件”。

## 可用的拦截器类型

- `Temporal\Interceptor\WorkflowClientCallsInterceptor`：拦截 WorkflowClient 的方法，例如启动或发送 Workflow 信号。
- `Temporal\Interceptor\WorkflowInboundCallsInterceptor`：拦截传入的工作流调用，包括执行、信号（Signals）和查询（Queries）。
- `Temporal\Interceptor\WorkflowOutboundCallsInterceptor`：拦截传出的工作流调用到 Temporal API，例如调度活动和启动计时器。
- `Temporal\Interceptor\ActivityInboundCallsInterceptor`：拦截传入活动的调用，例如 execute。
- `Temporal\Interceptor\GrpcClientInterceptor`：拦截所有服务客户端 gRPC 调用（参见 ServiceClientInterface）。
- `Temporal\Interceptor\WorkflowOutboundRequestInterceptor`：拦截发送到 RoadRunner 服务器的所有命令（参见 RequestInterface 实现）。
- `Temporal\Exception\ExceptionInterceptorInterface`：提供一种机制，让工作流知道异常是否应被视为致命（停止执行）或仅导致任务执行失败（并进行连续重试）。

每个上述拦截器都有对应的 trait，你可以使用它们来实现自己的拦截器。

> **警告**
> 某些拦截器接口将在未来扩展。为了避免兼容性问题，请始终在你的实现中使用相应的 trait。当新方法添加到接口时，这些 trait 将防止实现中断。

- `Temporal\Interceptor\Trait\Temporal\Interceptor\Trait`
- `Temporal\Interceptor\Trait\Temporal\Interceptor\WorkflowClientCallsInterceptorTrait`
- `Temporal\Interceptor\Trait\Temporal\Interceptor\WorkflowInboundCallsInterceptorTrait`
- `Temporal\Interceptor\Trait\Temporal\Interceptor\WorkflowOutboundCallsInterceptorTrait`
- `Temporal\Interceptor\Trait\Temporal\Interceptor\WorkflowOutboundRequestInterceptorTrait`

## 创建拦截器

要创建拦截器，你需要创建一个类并实现特定的接口。

例如，让我们创建一个拦截器，该拦截器将根据属性配置活动选项。

```php app/Temporal/Interceptors/ActivityAttributesInterceptor.php
<?php

declare(strict_types=1);

namespace App\Temporal\Interceptors;

use React\Promise\PromiseInterface;
use ReflectionAttribute;
use Temporal\Interceptor\Trait\WorkflowOutboundCallsInterceptorTrait;
use Temporal\Interceptor\WorkflowOutboundCalls\ExecuteActivityInput;
use Temporal\Interceptor\WorkflowOutboundCallsInterceptor;
use Temporal\Samples\Interceptors\Attribute;
use Temporal\Samples\Interceptors\Attribute\ActivityOption;

/**
 * implement {@see ActivityOption} interface.
 */
final class ActivityAttributesInterceptor implements WorkflowOutboundCallsInterceptor
{
    use WorkflowOutboundCallsInterceptorTrait;

    public function executeActivity(ExecuteActivityInput $input, callable $next): PromiseInterface
    {
        if ($input->method === null) {
            return $next($input);
        }

        $options = $input->options;

        foreach ($this->iterateOptions($input->method) as $attribute) {
            if ($attribute instanceof Attribute\StartToCloseTimeout) {
                \error_log(\sprintf('Redeclare start_to_close timeout of %s to %s', $input->type, $attribute->timeout));
                $options = $options->withStartToCloseTimeout($attribute->timeout);
            }
        }

        return $next($input->with(options: $options));
    }

    /**
     * @return iterable<int, ActivityOption>
     */
    private function iterateOptions(\ReflectionMethod $method): iterable
    {
        $class = $method->getDeclaringClass();
        foreach ($class->getAttributes(Attribute\ActivityOption::class, ReflectionAttribute::IS_INSTANCEOF) as $attr) {
            yield $attr->newInstance();
        }

        foreach ($method->getAttributes(Attribute\ActivityOption::class, ReflectionAttribute::IS_INSTANCEOF) as $attr) {
            yield $attr->newInstance();
        }
    }
}
```

## 注册拦截器

### 通过配置文件

要注册拦截器，你需要将其添加到 `config/temporal.php` 配置文件中。

```php
use App\Temporal\Interceptors\ActivityAttributesInterceptor;

return [
  // ...
  'interceptors' => [
    ActivityAttributesInterceptor::class,
  ],
];
```

你可以将拦截器定义为一个类、实例或自动装配：

```php
use Spiral\Core\Container\Autowire;
use App\Temporal\Interceptors\ActivityAttributesInterceptor;

return [
  'interceptors' => [
    ActivityAttributesInterceptor::class,
    
    new ActivityAttributesInterceptor(),
    
    new Autowire(ActivityAttributesInterceptor::class),
  ],
];
```

### 通过引导加载程序（Bootloaders）

你还可以通过引导加载程序注册拦截器。

```php
namespace App\Application\Bootloader;

use App\Temporal\Interceptors\ActivityAttributesInterceptor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\TemporalBridge\Bootloader\TemporalBridgeBootloader;
use Spiral\Core\Container\Autowire;

class AppBootloader extends Bootloader
{
    public function init(TemporalBridgeBootloader $temporal): void
    {
        $temporal->addInterceptor(ActivityAttributesInterceptor::class);
        // or
        $temporal->addInterceptor(new ActivityAttributesInterceptor());
        // or
        $temporal->addInterceptor(new Autowire(ActivityAttributesInterceptor::class));
    }
}
```

## 异常拦截器

异常拦截器提供一种机制，让工作流知道异常是否应被视为致命（停止执行）或仅导致任务执行失败（并进行连续重试）。

**这是一个异常拦截器的示例：**

```php app/Temporal/Interceptors/ExceptionInterceptor.php
<?php

declare(strict_types=1);

namespace App\Temporal\Interceptors;

use Temporal\Exception\ExceptionInterceptorInterface;

class ExceptionInterceptor implements ExceptionInterceptorInterface
{
    public function isRetryable(\Throwable $e): bool
    {
        if ($e instanceof ApiRateLimitException) {
            return true;
        }
        
        return false;
    }
}
```

**注册异常拦截器：**

```php app/config/temporal.php
use Temporal\Worker\WorkerOptions;
use App\Temporal\Interceptors\ExceptionInterceptor;

return [
    // ...
    'workers' => [
        'workerName' => [
            'exception_interceptor' => new ExceptionInterceptor(),
        ]
    ],
];
```
