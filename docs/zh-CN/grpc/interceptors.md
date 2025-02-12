# GRPC — 拦截器

Spiral 提供了 gRPC 服务的拦截器，允许你在请求生命周期的不同阶段拦截和修改请求与响应。

> **另请参阅**
> 在 [框架 — 拦截器](../framework/interceptors.md) 部分阅读更多关于拦截器的信息。

有两种类型的拦截器：

1.  服务器拦截器
2.  客户端拦截器

## 服务器拦截器

服务器拦截器用于拦截和修改服务器接收到的请求和响应。它们通常用于添加跨切功能，例如日志记录、身份验证或服务器监控。

### 日志记录拦截器

下面是一个在处理请求前后记录请求的简单拦截器示例：

```php
namespace App\Endpoint\GRPC\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;

final class LogInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly \Psr\Log\LoggerInterface $core,
    ) {
    }
    
    public function process(string $name, string $action, array $parameters, CoreInterface $core): string
    {
        $this->logger->info('Request received...', [
            'name' => $name,
            'action' => $action,
        ]);
        
        $response = $core->callAction($name, $action, $parameters);
        
        $this->logger->info('Request processed', [
            'name' => $name,
            'action' => $action,
        ]);

        return $response;
    }
}
```

### 异常处理拦截器

下面是一个处理服务器抛出异常的简单拦截器示例。它将捕获所有异常并将它们转换为 gRPC 异常。

```php
namespace App\Endpoint\GRPC\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Exceptions\ExceptionReporterInterface;
use Spiral\RoadRunner\GRPC\Exception\GRPCException;
use Spiral\RoadRunner\GRPC\Exception\GRPCExceptionInterface;

final class ExceptionHandlerInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly ExceptionReporterInterface $reporter
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        try {
            return $core->callAction($controller, $action, $parameters);
        } catch (\Throwable $e) {
            $this->reporter->report($e);

            if ($e instanceof GRPCExceptionInterface) {
                throw $e;
            }

            throw new GRPCException(
                message: $e->getMessage(),
                previous: $e
            );
        }
    }
}
```

### 从请求中接收跟踪上下文

下面是一个从请求中接收跟踪上下文的简单拦截器示例。

```php
namespace App\Endpoint\GRPC\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Telemetry\TraceKind;
use Spiral\Telemetry\TracerFactoryInterface;
use Spiral\Core\CoreInterface;

class InjectTelemetryFromContextInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly TracerFactoryInterface $tracerFactory
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        $traceContext = [];

        if (isset($parameters['ctx']) and $parameters['ctx'] instanceof RequestContext) {
            $traceContext = $parameters['ctx']->getValue('telemetry-trace-id') ?? [];
        }

        return $this->tracerFactory->make($traceContext)->trace(
            name: \sprintf('Interceptor [%s]', __CLASS__),
            callback: static fn(): mixed => $core->callAction($controller, $action, $parameters),
            attributes: [
                'controller' => $controller,
                'action' => $action,
            ],
            scoped: true,
            traceKind: TraceKind::SERVER
        );
    }
}
```

### 保护拦截器

下面是一个检查用户是否已通过身份验证的简单拦截器示例。它将使用 PHP 属性来确定哪些方法需要身份验证。身份验证令牌将通过请求元数据传递。

```php
namespace App\Endpoint\GRPC\Interceptor;

use App\Attribute\Guarded;
use Spiral\Attributes\ReaderInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\RoadRunner\GRPC\ContextInterface;

final class GuardedInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly ReaderInterface $reader
    ) {
    }

    public function process(string $class, string $method, array $parameters, CoreInterface $core): mixed
    {
        $reflMethod = new \ReflectionMethod($class, $method);
        $attribute = $this->reader->firstFunctionMetadata($reflMethod, Guarded::class);

        if ($attribute !== null) {
            $this->checkAuth($attribute, $parameters['ctx']);
        }

        return $core->callAction($class, $method, $parameters);
    }

    private function checkAuth(Guarded $attribute, ContextInterface $ctx): void
    {
        // Metadata always stores values as array. 
        $token = $ctx->getValue($attribute->tokenField)[0] ?? null;

        // Here you can implement your own authentication logic
        if ($token !== 'secret') {
            throw new \Exception('Unauthorized.');
        }
    }
}
```

一个需要身份验证的方法的示例：

```php
use App\Attribute\Guarded;

#[Guarded]
public function ping(GRPC\ContextInterface $ctx, PingRequest $in): PingResponse
{
    // ...
}
````

以及 Guarded 属性的示例：

```php
namespace App\Attribute;

use Doctrine\Common\Annotations\Annotation\NamedArgumentConstructor;

#[\Attribute(\Attribute::TARGET_METHOD), NamedArgumentConstructor]
class Guarded
{
    public function __construct(
        public readonly string $tokenField = 'token'
    ) {
    }
}
```

### 注册拦截器

要使用此拦截器，你需要在配置文件 `app/config/grpc.php` 中注册它们。

```php app/config/grpc.php
return [    
    'interceptors' => [
        \App\Endpoint\GRPC\Interceptor\LogInterceptor::class,
        \App\Endpoint\GRPC\Interceptor\ExceptionHandlerInterceptor::class,
        \App\Endpoint\GRPC\Interceptor\GuardedInterceptor::class,
    ]
];
```

## 客户端拦截器

客户端拦截器用于拦截和修改由客户端发送的请求和响应。它们通常用于添加跨切功能，例如日志记录、修改标头、处理响应错误。

### 可拦截的客户端类

如果想使用客户端拦截器，你将需要修改 [客户端 SDK](./client.md) 部分中的客户端类。

> **另请参阅**
> 在 [框架 — 拦截器](../framework/interceptors.md) 部分阅读更多关于拦截器的信息。

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Service\PingerClient;
use App\Service\RequestCore;
use GRPC\Pinger\PingerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Core\InterceptableCore;

final class AppBootloader extends Bootloader
{
    protected const SINGLETONS = [
        PingerInterface::class => [self::class, 'initPingService'],
    ];

    private function initPingService(
        EnvironmentInterface $env
    ): PingServiceInterface
    {
        $core = new InterceptableCore(
            new RequestCore(
                $env->get('PING_SERVICE_HOST', '127.0.0.1:9001'),
                ['credentials' => \Grpc\ChannelCredentials::createInsecure()]
            )
        );

        // Here you can register your interceptors
        $core->addInterceptor(new \App\Service\Interceptor\HandleResponseErrorsInterceptor());

        return new PingerClient($core);
    }
}
```

然后实现 `RequestCore` 类：

```php
namespace App\Service;

use Spiral\Core\CoreInterface;
use Spiral\RoadRunner\GRPC\ContextInterface;

final class RequestCore extends \Grpc\BaseStub implements CoreInterface
{
    public function callAction(string $controller, string $action, array $parameters = []): mixed
    {
        $ctx = $parameters['ctx'];
        \assert($ctx instanceof ContextInterface);

        return $this->_simpleRequest(
            $action,
            $parameters['in'],
            [$parameters['responseClass'], 'decode'],
            (array) $ctx->getValue('metadata'),
            (array) $ctx->getValue('options')
        )->wait();
    }
}
```

最后修改 `PingerClient` 类：

```php
namespace App\Service;

use App\GRPC\Pinger;
use Spiral\Core\CoreInterface;
use Spiral\RoadRunner\GRPC;

final class PingerClient implements Pinger\PingerInterface
{
    public function __construct(
        private readonly RequestCore $core
    ) {
    }

    public function ping(GRPC\ContextInterface $ctx, Pinger\PingRequest $in): Pinger\PingResponse
    {
        return $this->sendRequest(
            '/' . self::NAME . '/ping',
            $in,
            $ctx,
            Pinger\PingResponse::class
        );
    }

    /**
     * @template T of object
     * @param non-empty-string $method
     * @param class-string<T> $response
     * @return T
     */
    public function sendRequest(
        string $method,
        \GRPC\Ping\PingRequest $in,
        GRPC\ContextInterface $ctx,
        string $response
    ): object {
        [$response, $status] = $this->core->callAction(
            self::class, $method,
            [
                'responseClass' => $response,
                'ctx' => $ctx,
                'in' => $in,
            ]
        );

        return $response;
    }
}
```

### 处理响应错误拦截器

```php
namespace App\Service\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\RoadRunner\GRPC;

final class HandleResponseErrorsInterceptor implements CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        [$response, $status] = $core->callAction($controller, $action, $parameters);

        $code = $status->code ?? GRPC\StatusCode::UNKNOWN;

        if ($code !== GRPC\StatusCode::OK) {
            throw new GRPC\Exception\GRPCException(
                message: $status->details,
                code: $status->code
            );
        }

        return [$response, $code];
    }
}
```

### 将遥测跟踪 ID 传递到上下文

```php
namespace App\Service\Interceptor;

use Psr\Container\ContainerInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\RoadRunner\GRPC\ContextInterface;
use Spiral\RoadRunner\GRPC\ResponseHeaders;
use Spiral\Telemetry\TraceKind;
use Spiral\Telemetry\TracerInterface;
use Spiral\Core\CoreInterface;

class InjectTelemetryIntoContextInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly ContainerInterface $container
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        $tracer = $this->container->get(TracerInterface::class);
        \assert($tracer instanceof TracerInterface);

        if (isset($parameters['ctx']) and $parameters['ctx'] instanceof RequestContext) {
            $metadata = $parameters['ctx']->getValue('metadata');
            if(!\is_array($metadata)) {
                $metadata = [];
            }
            
            $metadata['telemetry-trace-id'] = $tracer->getContext();
            $parameters['ctx'] = $parameters['ctx']->withValue('metadata', $metadata);
        }

        return $tracer->trace(
            name: \sprintf('GRPC request %s', $action),
            callback: static fn() => $core->callAction($controller, $action, $parameters),
            attributes: compact('controller', 'action'),
            traceKind: TraceKind::PRODUCER
        );
    }
}
```
