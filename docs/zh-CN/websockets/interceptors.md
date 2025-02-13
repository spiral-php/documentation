# Websockets — 拦截器

Spiral 框架为 WebSockets 服务提供了拦截器，允许你在请求生命周期的各个阶段拦截和修改请求及响应。

> **另请参阅**
> 在 [框架 — 拦截器](../framework/interceptors.md) 部分阅读更多关于拦截器的信息。

它们通常用于添加横切功能，例如向服务器添加日志记录、身份验证或监控。

## 示例

### 身份验证拦截器

以下示例展示了如何创建一个拦截器，该拦截器检查用户的身份验证令牌并将用户的身份提供给服务。其中 `authToken` 是请求数据中包含身份验证令牌的字段的名称。

```php app/src/Entrypoint/Centrifugo/Interceptor/AuthenticatorInterceptor.php
namespace App\Entrypoint\Centrifugo\Interceptor;

use Psr\EventDispatcher\EventDispatcherInterface;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\Auth\ActorProviderInterface;
use Spiral\Auth\AuthContext;
use Spiral\Auth\AuthContextInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Core\ScopeInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

final class AuthenticatorInterceptor implements CoreInterceptorInterface
{
    use PrototypeTrait;

    public function __construct(
        private readonly ScopeInterface $scope,
        private readonly ActorProviderInterface $actorProvider,
        private readonly ?EventDispatcherInterface $eventDispatcher = null,
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        $request = $parameters['request'];
        \assert($request instanceof RequestInterface);

        $authToken = $request->getData()['authToken'] ?? null;

        if (!$authToken || !$token = $this->authTokens->load($authToken)) {
            $request->error(403, 'Unauthorized');
            return null;
        }

        $auth = new AuthContext($this->actorProvider, $this->eventDispatcher);
        $auth->start($token);

        return $this->scope->runScope([
            AuthContextInterface::class => $auth,
        ], fn () => $core->callAction($controller, $action, $parameters));
    }
}
```

以及如何在服务中使用它的示例：

```php app/src/Entrypoint/Centrifugo/ConnectService.php
namespace App\Endpoint\Centrifugo;

use App\Database\User;
use RoadRunner\Centrifugo\Payload\ConnectResponse;
use RoadRunner\Centrifugo\Request\Connect;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

final class ConnectService implements ServiceInterface
{
    use PrototypeTrait;

    /** @param Connect $request */
    public function handle(RequestInterface $request): void
    {
        try {
            $user = $this->auth->getActor();

            $request->respond(
                new ConnectResponse(
                    user: (string)$user->getId(),
                    data: ['user' => $user->jsonSerialize()],
                    channels: ['chat'],
                ),
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

### 错误处理拦截器

以下示例展示了如何创建一个处理错误的拦截器。

```php app/src/Entrypoint/Centrifugo/Interceptor/ExceptionHandlerInterceptor.php
namespace App\Entrypoint\Centrifugo\Interceptor;

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
            \assert($parameters['request'] instanceof RequestInterface);

            return $core->callAction($controller, $action, $parameters);
        } catch (\Throwable $e) {
            $this->reporter->report($e);

            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

之后，你不需要在你的服务中编写 try/catch 块：

```php app/src/Entrypoint/Centrifugo/ConnectService.php
namespace App\Endpoint\Centrifugo;
/**
 * @param Connect $request
 */
public function handle(RequestInterface $request): void
{
    if (!$this->auth->isAuthenticated()) {
        thorw new \Exception('Unauthorized', 403);
    }
    
    $request->respond(
        ...
    );
}
```

## 注册拦截器

要使用此拦截器，你需要在配置文件 `app/config/centrifugo.php` 中注册它们。

```php app/config/centrifugo.php
use RoadRunner\Centrifugo\Request\RequestType;
use App\Centrifuge;

return [
    'services' => [
        //...
    ],
    'interceptors' => [
        RequestType::Connect->value => [
            Centrifuge\Interceptor\AuthInterceptor::class,
        ],
        //...
        '*' => [
            Centrifuge\Interceptor\ExceptionHandlerInterceptor::class,
            Centrifuge\Interceptor\TelemetryInterceptor::class,
        ],
    ],
];
```

你可以使用 `*` 通配符为特定请求或所有请求注册拦截器。
