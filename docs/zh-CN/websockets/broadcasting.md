# Websockets — 事件广播

能够以实时方式从服务器向客户端广播数据是许多现代 Web 和移动应用程序的需求。当服务器上的一些数据更新时，通常会通过 WebSocket 连接发送一条消息，由客户端处理。WebSocket 提供了更高效的替代方案，可以持续轮询应用程序的服务器以获取应在 UI 中反映的数据更改。

## 安装

要启用该组件，您只需要将 `Spiral\Broadcasting\Bootloader\BroadcastingBootloader` 类添加到应用程序类中的 bootloaders 列表中。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Broadcasting\Bootloader\BroadcastingBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloaders 的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Broadcasting\Bootloader\BroadcastingBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloaders 的信息。
:::

::::

## 配置

Spiral 中用于广播的配置存储在 `app/config/broadcasting.php` 文件中。该框架包含几个内置的广播驱动程序，包括用于本地开发和调试的 `log` 驱动程序，以及用于在测试期间禁用广播的 `null` 驱动程序。

可以使用 `BROADCAST_CONNECTION` 环境变量更改默认驱动程序：

```dotenv .env
# Broadcasting
BROADCAST_CONNECTION=log
```

或者通过修改配置文件中的 `default` 设置。

```php app/config/broadcasting.php
use Spiral\Broadcasting\Driver\LogBroadcast;
use Spiral\Broadcasting\Driver\NullBroadcast;

return [
    'default' => 'log',
    'authorize' => [],
    'aliases' => [],
    'connections' => [
        'log' => [
            'driver' => 'log',
        ],
        'null' => [
            'driver' => NullBroadcast::class,
        ],
    ],
    'driverAliases' => [
        'log' => LogBroadcast::class,
    ],
];
```

## 使用

Spiral 提供了 `Spiral\Broadcasting\BroadcastInterface` 接口，用于使用 `publish` 方法将事件发送到默认连接。

```php
use Spiral\Broadcasting\BroadcastInterface;
use Spiral\Serializer\SerializerInterface;

class OrderService
{
    public function __construct(
        private readonly BroadcastInterface $broadcast,
        private readonly SerializerInterface $serializer
    ) {
    }

    public function purchase(string $orderUuid): void
    {
        // ...

        $this->broadcast->publish(
            "order.{$orderUuid}",
            $this->serializer->serialize(['status' => 'purchased'])
        );
    }
}
```

在某些情况下，您可能希望将事件发送到特定的广播驱动程序，而不是默认连接。为此，Spiral 提供了 `Spiral\Broadcasting\BroadcastManagerInterface`。它允许您获取特定驱动程序的连接实例，并使用它来发布事件。

```php
use Spiral\Broadcasting\BroadcastManagerInterface;
use Spiral\Serializer\SerializerInterface;

class OrderService
{
    public function __construct(
        private readonly BroadcastManagerInterface $broadcast,
        private readonly SerializerInterface $serializer
    ) {
    }

    public function send(string $orderUuid): void
    {
        $this->broadcast
            ->connection('log')
            ->publish(
               "order.{$orderUuid}",
                $this->serializer->serialize(['status' => 'purchased'])
            );
    }
}
```

`BroadcastInterface` 的 `publish` 方法也可以接受 `\Stringable` 类型作为 topic 参数。这允许您使用实现它的对象创建 topic。

```php
namespace App\Broadcast\Topic;

final class Order implements \Stringable
{
    public function __construct(
        public readonly string $orderUuid
    ) {
    }

    public function __toString(): string
    {
        return \sprintf('order.%s', $this->orderUuid);
    }
}
```

下面是一个示例，说明如何将 `Order` 类与 `publish` 方法一起使用：

```php
use Spiral\Broadcasting\BroadcastManagerInterface;
use Spiral\Serializer\SerializerInterface;

class OrderService
{
    public function __construct(
        private readonly BroadcastManagerInterface $broadcast,
        private readonly SerializerInterface $serializer
    ) {
    }

    public function send(Order $order): void
    {
        $this->broadcast
            ->connection('log')
            ->publish(
               $order,
               $this->serializer->serialize(['status' => 'purchased'])
            );
    }
}
```

## 驱动程序

### Centrifugo

> **另请参阅**
> 在 [Websockets — Installation and Configuration](./configuration.md) 部分阅读更多关于与 Centrifugo Websocket 服务器的集成信息。

安装后，您可以使用 `BROADCAST_CONNECTION` 环境变量激活驱动程序：

```dotenv .env
# Broadcasting
BROADCAST_CONNECTION=centrifugo
```

### 自定义驱动程序

您可以通过实现 `Spiral\Broadcasting\BroadcastInterface` 接口来创建自己的驱动程序。该驱动程序应实现 `publish` 方法，该方法接受一个 topic 和一个 payload。

```php
namespace App\Broadcast;

use Pusher\Pusher;
use Spiral\Broadcasting\Driver\AbstractBroadcast;

final class PusherBroadcast extends AbstractBroadcast
{ 
    public function __construct(
        private readonly Pusher $pusher
    ){
    }
    
    public function publish(iterable|string|\Stringable $topics, iterable|string $messages): void
    {
        $topics = $this->formatTopics($this->toArray($topics));
        
        /** @var string $message */
        foreach ($this->toArray($messages) as $message) {
            \assert(\is_string($message), 'Message argument must be a type of string');

            $this->pusher->trigger($topics, $message, []);
        }
    }
}
```

然后，您可以在 `app/config/broadcasting.php` 文件中注册它。

```php app/config/broadcasting.php
use Spiral\Broadcasting\Driver\LogBroadcast;
use Spiral\Broadcasting\Driver\NullBroadcast;
use App\Broadcast\PusherBroadcast;

return [
    'default' => 'log',
    'connections' => [
        'log' => [
            'driver' => 'log',
        ],
        'pusher' => [
            'driver' => 'pusher',
        ],
        'null' => [
            'driver' => NullBroadcast::class,
        ],
    ],
    'driverAliases' => [
        'log' => LogBroadcast::class,
        'pusher' => PusherBroadcast::class,
    ],
];
```

如果您的应用程序中存在需要授权的 topic，您可以使用 `Spiral\Broadcasting\GuardInterface` 来实现授权检查。`GuardInterface` 定义了单个方法 `authorize`，该方法接受一个 `Psr\Http\Message\ServerRequestInterface` 实例作为参数，并返回一个 `Spiral\Broadcasting\AuthorizationStatus` 实例。

```php
use Pusher\Pusher;
use Spiral\Broadcasting\AuthorizationStatus;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Core\InvokerInterface;
use Spiral\Core\ScopeInterface;
use Spiral\Broadcasting\TopicRegistryInterface;

final class PusherBroadcaster extends AbstractBroadcast implements \Spiral\Broadcasting\GuardInterface
{
    public function __construct(
        private readonly Pusher $pusher,
        private readonly InvokerInterface $invoker,
        private readonly ScopeInterface $scope,
        private readonly TopicRegistryInterface $topics
    ) {
    }

    // ...
    
    public function authorize(ServerRequestInterface $request): AuthorizationStatus
    {
        $topic = $request->getQueryParams()['channel_name'] ?? null;
        
        if (!\is_string($topic)) {
            return new AuthorizationStatus(false, []);
        }
        
        if (!$this->authorizeTopic($request, $topic)) {
            return new AuthorizationStatus(false, [$topic]);
        }

        return new AuthorizationStatus(true, [$topic]);
    }
    
    
    private function authorizeTopic(ServerRequestInterface $request, string $topic): bool
    {
        $parameters = [];
        $callback = $this->topics->findCallback($topic, $parameters);
        if ($callback === null) {
            return false;
        }

        return $this->invoke($request, $callback, $parameters + ['topic' => $topic]);
    }

    private function invoke(ServerRequestInterface $request, callable $callback, array $parameters = []): bool
    {
        return $this->scope->runScope(
            [
                ServerRequestInterface::class => $request,
            ],
            fn (): bool => $this->invoker->invoke($callback, $parameters)
        );
    }
}
```

`Spiral\Broadcasting\TopicRegistryInterface` 是一个中心位置，用于注册广播组件中特定 topic 的授权规则。您可以使用 `app/config/broadcasting.php` 文件注册授权规则。

```php app/config/broadcasting.php
return [
    'authorize' => [
        'path' => env('BROADCAST_AUTHORIZE_PATH'),
        'topics' => [
            'topic' => static fn (ServerRequestInterface $request): bool => $request->getHeader('SECRET')[0] == 'secret',
            'order.{uuid}' => static fn (string $uuid, Actor $actor): bool => $actor->getId() === $id
        ],
    ],
];
```

许多 websocket 服务器使用 HTTP 授权，这要求客户端以 HTTP 标头或查询参数的形式提供凭据。为了在广播组件中处理 HTTP 授权，您可以使用 `Spiral\Broadcasting\Middleware\AuthorizationMiddleware` 中间件，并在您的应用程序中指定一个授权端点，websocket 服务器将向该端点发送授权请求。

`AuthorizationMiddleware` 中间件负责验证客户端的凭据，并返回一个适当的响应，指示客户端是否有权访问 websocket 服务器。要使用该中间件，您需要 [注册](../http/routing.md#middleware) 它并在 `app/config/broadcasting.php` 配置文件中指定授权端点。

```php app/config/broadcasting.php
return [
    'authorize' => [
        'path' => '/pusher/user-auth', // <===============
        'topics' => [
            'topic' => static fn (ServerRequestInterface $request): bool => $request->getHeader('SECRET')[0] == 'secret',
            'order.{uuid}' => static fn (string $uuid, Actor $actor): bool => $actor->getId() === $id
        ],
    ],
];
```

或者使用 `BROADCAST_AUTHORIZE_PATH` 环境变量。

```dotenv .env
# Broadcasting
BROADCAST_AUTHORIZE_PATH=/pusher/user-auth
```
