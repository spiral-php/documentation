# Websockets — 事件处理器

在 Spiral 应用中，一个 *service（服务）* 是负责处理传入消息、根据消息内容执行操作并向发送者返回响应的类。服务可以处理几种类型的事件：

- **连接请求**：当客户端建立到 Centrifugo 服务器的 WebSocket 连接时，会向 RoadRunner 发送一个连接请求事件。
- **刷新连接请求**：当需要刷新客户端和服务器之间的连接时，Centrifugo 会向 RoadRunner 发送一个刷新事件。
- **RPC 调用请求**：Centrifugo 可以通过客户端连接向 RoadRunner 发送 RPC（远程过程调用）事件，允许您从客户端调用服务器端函数。
- **订阅请求**：当客户端订阅一个频道时，Centrifugo 会向 RoadRunner 发送一个订阅请求事件。
- **发布请求**：此请求发生在消息发布到频道之前，允许您的后端验证客户端是否可以将数据发布到频道。

## 创建服务

要创建 *service（服务）*，您需要实现 `Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface` 接口，并提供 `handle` 方法来处理传入的请求。

`handle` 方法将接收一个实现 `RoadRunner\Centrifugo\Request\RequestInterface` 接口的请求对象。接收到的请求对象的具体类型将取决于正在处理的事件类型。例如，连接请求事件将传递一个 `RoadRunner\Centrifugo\Request\Connect` 对象，而订阅请求事件将传递一个 `RoadRunner\Centrifugo\Request\Subscribe` 对象。

请求对象有一个 `respond` 方法，该方法应该用于向 Centrifugo 服务器发送响应。响应数据将作为对象传递，该对象实现 `RoadRunner\Centrifugo\Payload\ResponseInterface` 接口。

要注册一个服务，您需要指定事件类型和服务的类名。事件类型使用 `RoadRunner\Centrifugo\Request\RequestType` 枚举指定，该枚举具有每个支持的事件类型的常量。

### 服务注册

以下是关于如何在 `app/config/centrifugo.php` 配置文件中注册事件处理器服务的示例：

```php app/config/centrifugo.php
use RoadRunner\Centrifugo\Request\RequestType;
use App\Centrifuge;

return [
    'services' => [
        RequestType::Connect->value => Centrifuge\ConnectService::class,
        //...
    ],
    'interceptors' => [
        //...
    ],
];
```

> **注意**
> 有关拦截器以及如何在 Spiral 应用程序中使用拦截器的更多信息，您可以参考关于 [拦截器](./interceptors.md) 的文档部分。此页面提供了额外的详细信息和示例，以帮助您开始使用拦截器。

## 连接请求

此服务接收一个 `RoadRunner\Centrifugo\Request\Connect` 对象，并根据连接请求执行某些操作。它应该使用 `RoadRunner\Centrifugo\Payload\ConnectResponse` 对象向 Centrifugo 服务器做出响应。

### 简单示例

以下是一个接受所有连接请求的服务的示例：

```php app/src/Endpoint/Centrifugo/ConnectService.php
namespace App\Endpoint\Centrifugo;

use RoadRunner\Centrifugo\Request\Connect;
use RoadRunner\Centrifugo\Payload\ConnectResponse;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

class ConnectService implements ServiceInterface
{
    /** @param Connect $request */
    public function handle(RequestInterface $request): void
    {
        try {
            $request->respond(
                // Return an empty string for accepting unauthenticated requests
                new ConnectResponse(
                  user: ''
                )
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

要创建服务类，请使用以下命令：

```terminal
php app.php create:centrifugo-handler Connect -t=connect
```

Centrifugo JavaScript SDK 允许您在通过 WebSocket 连接时将其他数据传递到服务器。此数据可用于各种目的，例如客户端身份验证。

### 自动订阅频道

Centrifugo 最酷的功能之一是能够在客户端连接时自动订阅频道。

例如，当用户连接到服务器时，您可以自动将他们订阅到公共频道，而无需在客户端执行任何其他操作。

您只需要在 `channels` 字段中返回一个频道列表：

```php app/src/Endpoint/Centrifugo/ConnectService.php
namespace App\Endpoint\Centrifugo;

use RoadRunner\Centrifugo\Request\Connect;
use RoadRunner\Centrifugo\Payload\ConnectResponse;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

class ConnectService implements ServiceInterface
{
    /** @param Connect $request */
    public function handle(RequestInterface $request): void
    {
        try {
            $request->respond(
                // Return an empty string for accepting unauthenticated requests
                new ConnectResponse(
                  user: '',
                   // List of channels to subscribe to on connect to Centrifugo
                  channels: [
                     'public',
                     ...
                  ],
                )
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

### 仅接受已认证请求的服务

已认证的请求是在 `authToken` 字段中包含有效令牌的请求。此令牌可用于在服务器端对用户进行身份验证。

**例如，它对于以下情况很有用：**

- 在服务器端进行用户身份验证
- 自动订阅私有频道
- 根据用户角色自动订阅频道

```php app/src/Endpoint/Centrifugo/ConnectService.php
namespace App\Endpoint\Centrifugo;

use RoadRunner\Centrifugo\Request\Connect;
use RoadRunner\Centrifugo\Payload\ConnectResponse;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;
use Spiral\Auth\TokenStorageInterface;

class ConnectService implements ServiceInterface
{
    public function __construct(
        private readonly TokenStorageInterface $tokenStorage,
    ) {
    }
    
    /** @param Connect $request */
    public function handle(RequestInterface $request): void
    {
        try {
            $userId = null;
            
            // Authenticate user with a given token from the connection request
            $authToken = $request->getData()['authToken'] ?? null;
            if ($authToken && $token = $this->tokenStorage->load($authToken)) {
                $userId = $token->getPayload()['userID'] ?? null;
            }
            
            // You can also disconnect connection
            if (!$userId) {
                $request->disconnect('403', 'Connection is not allowed.');
                return;
            }
            
            $user = $this->users->getById($userId);
            $roles = $user->getRoles();
        
            $request->respond(
                new ConnectResponse(
                    user: (string) $userId,
                    channels: [
                        (string) new UserChannel($user->uuid), // user-{uuid}
                        (string) new ChatChannel($user->uuid), // chat-{uuid}
                        'public',
                    ],
                )
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

> **另请参阅**
> 在 [Centrifugo documentation](https://centrifugal.dev/docs/server/proxy#connect-request-fields) 中阅读有关连接请求的更多信息。

以下是使用 Centrifugo JavaScript SDK 传递 `authToken` 进行客户端身份验证的示例：

```javascript
import {Centrifuge} from 'centrifuge';

const centrifuge = new Centrifuge('http://127.0.0.18000/connection/websocket', {
    data: {
        authToken: 'my-app-auth-token'
    }
});
```

> **另请参阅**
> 有关使用 JavaScript SDK 和将其他数据传递到服务器的更多信息，请参阅 [documentation](https://github.com/centrifugal/centrifuge-js#data)。

## 订阅请求

此服务接收一个 `RoadRunner\Centrifugo\Request\Subscribe` 对象，并根据连接请求执行某些操作。它应该使用 `RoadRunner\Centrifugo\Payload\SubscribeResponse` 对象向 Centrifugo 服务器做出响应。

要创建服务类，请使用以下命令：

```terminal
php app.php create:centrifugo-handler Subscribe -t=subscribe
```

在此示例中，我们将创建一个服务，该服务将允许用户仅在他们通过 `Spiral\Broadcasting\TopicRegistryInterface` 接口提供的规则进行身份验证后才能订阅频道。

```php app/src/Endpoint/Centrifugo/SubscribeService.php
namespace App\Endpoint\Centrifugo;

use RoadRunner\Centrifugo\Payload\SubscribeResponse;
use RoadRunner\Centrifugo\Request\Subscribe;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;
use Spiral\Broadcasting\TopicRegistryInterface;

final class SubscribeService implements ServiceInterface
{
    public function __construct(
        private readonly InvokerInterface $invoker,
        private readonly ScopeInterface $scope,
        private readonly TopicRegistryInterface $topics,
    ) {
    }

    /**
     * @param Subscribe $request
     */
    public function handle(RequestInterface $request): void
    {
        try {
            if (!$this->authorizeTopic($request)) {
                $request->disconnect('403', 'Channel is not allowed.');
                return;
            }
        
            $request->respond(
                new SubscribeResponse()
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
    
    private function authorizeTopic(Subscribe $request): bool
    {
        $parameters = [];
        $callback = $this->topics->findCallback($request->channel, $parameters);
        if ($callback === null) {
            return false;
        }

        return $this->invoke(
            $request, 
            $callback, 
            $parameters + ['topic' => $request->channel, 'userId' => $request->user]
        );
    }

    private function invoke(Subscribe $request, callable $callback, array $parameters = []): bool
    {
        return $this->scope->runScope(
            [
                RequestInterface::class => $request,
            ],
            fn (): bool => $this->invoker->invoke($callback, $parameters)
        );
    }
}
```

您可以在 `app/config/broadcasting.php` 文件中注册频道授权规则：

```php app/src/Endpoint/Centrifugo/SubscribeService.php
namespace App\Endpoint\Centrifugo;

use RoadRunner\Centrifugo\Payload\SubscribeResponse;
use RoadRunner\Centrifugo\Request\Subscribe;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;
use Spiral\Broadcasting\TopicRegistryInterface;

final class SubscribeService implements ServiceInterface
{
    public function __construct(
        private readonly InvokerInterface $invoker,
        private readonly ScopeInterface $scope,
        private readonly TopicRegistryInterface $topics,
    ) {
    }

    /**
     * @param Subscribe $request
     */
    public function handle(RequestInterface $request): void
    {
        try {
            if (!$this->authorizeTopic($request)) {
                $request->disconnect('403', 'Channel is not allowed.');
                return;
            }
        
            $request->respond(
                new SubscribeResponse()
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
    
    private function authorizeTopic(Subscribe $request): bool
    {
        $parameters = [];
        $callback = $this->topics->findCallback($request->channel, $parameters);
        if ($callback === null) {
            return false;
        }

        return $this->invoke(
            $request, 
            $callback, 
            $parameters + ['topic' => $request->channel, 'userId' => $request->user]
        );
    }

    private function invoke(Subscribe $request, callable $callback, array $parameters = []): bool
    {
        return $this->scope->runScope(
            [
                RequestInterface::class => $request,
            ],
            fn (): bool => $this->invoker->invoke($callback, $parameters)
        );
    }
}
```

> **另请参阅**
> 在 [Centrifugo documentation](https://centrifugal.dev/docs/server/proxy#subscribe-request-fields) 中阅读有关订阅请求的更多信息。

## 刷新连接请求

此服务接收一个 `RoadRunner\Centrifugo\Request\Refresh` 对象，并根据连接请求执行某些操作。它应该使用 `RoadRunner\Centrifugo\Payload\RefreshResponse` 对象向 Centrifugo 服务器做出响应。

要创建服务类，请使用以下命令：

```terminal
php app.php create:centrifugo-handler Refresh -t=refresh
```

以下是服务的示例：

```php app/src/Endpoint/Centrifugo/RefreshService.php
namespace App\Endpoint\Centrifugo;

use RoadRunner\Centrifugo\Request\Refresh;
use RoadRunner\Centrifugo\Payload\RefreshResponse;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

class RefreshService implements ServiceInterface
{
    /** @param Refresh $request */
    public function handle(RequestInterface $request): void
    {
        try {
            $request->respond(
                new RefreshResponse(...)
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

> **另请参阅**
> 在 [Centrifugo documentation](https://centrifugal.dev/docs/server/proxy#refresh-request-fields) 中阅读有关刷新连接请求的更多信息。

## RPC 调用请求

此服务接收一个 `RoadRunner\Centrifugo\Request\RPC` 对象，并根据连接请求执行某些操作。它应该使用 `RoadRunner\Centrifugo\Payload\RPCResponse` 对象向 Centrifugo 服务器做出响应。

要创建服务类，请使用以下命令：

```terminal
php app.php create:centrifugo-handler Rpc -t=rpc
```

#### 简单示例

以下是接收 `ping` RPC 调用并以 `pong` 响应的服务示例：

```php app/src/Endpoint/Centrifugo/RPCService.php
namespace App\Endpoint\Centrifugo;

use RoadRunner\Centrifugo\Payload\RPCResponse;
use RoadRunner\Centrifugo\Request\RequestInterface;
use RoadRunner\Centrifugo\Request\RPC;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

final class RPCService implements ServiceInterface
{
    /**
     * @param RPC $request
     */
    public function handle(RequestInterface $request): void
    {
        $result = match ($request->method) {
            'ping' => 'pong',
            default => ['error' => 'Not found', 'code' => 404]
        };

        try {
            $request->respond(
                new RPCResponse(
                    data: $result
                )
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

> **另请参阅**
> 在 [Centrifugo documentation](https://centrifugal.dev/docs/server/proxy#rpc-request-fields) 中阅读有关 RPC 请求的更多信息。

#### 将 RPC 方法代理到 HTTP 层

以下是将 RPC 方法代理到 HTTP 层并将结果返回给 Centrifugo 服务器的示例：

```php app/src/Endpoint/Centrifugo/RPCService.php
namespace App\Endpoint\Centrifugo;

use Psr\Http\Message\ServerRequestFactoryInterface;
use Psr\Http\Message\ServerRequestInterface;
use RoadRunner\Centrifugo\Payload\RPCResponse;
use RoadRunner\Centrifugo\Request\RPC;
use Spiral\Filters\Exception\ValidationException;
use Spiral\Http\Http;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

final class RPCService implements ServiceInterface
{
    public function __construct(
        private readonly Http $http,
        private readonly ServerRequestFactoryInterface $requestFactory,
    ) {
    }

    /**
     * @param RPC $request
     */
    public function handle(Request\RequestInterface $request): void
    {
        try {
            $response = $this->http->handle($this->createHttpRequest($request));

            $result = \json_decode((string)$response->getBody(), true);
            $result['code'] = $response->getStatusCode();
        } catch (ValidationException $e) {
            $result['code'] = $e->getCode();
            $result['errors'] = $e->errors;
            $result['message'] = $e->getMessage();
        } catch (\Throwable $e) {
            $result['code'] = $e->getCode();
            $result['message'] = $e->getMessage();
        }

        try {
            $request->respond(new RPCResponse(data: $result));
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }

    public function createHttpRequest(Request\RPC $request): ServerRequestInterface
    {
        if(!\str_contains($request->method, ':')) {
            throw new \InvalidArgumentException('Method must be in format "method:uri"');
        }

        // Example of method string: get:users/1 , post:news/store, delete:news/1
        // split received method string to HTTP method and uri  
        [$method, $uri] = \explode(':', $request->method, 2);
        
        $method = \strtoupper($method);

        $httpRequest = $this->requestFactory->createServerRequest($method, \ltrim($uri, '/'))
            ->withHeader('Content-Type', 'application/json');

        return match ($method) {
            'GET', 'HEAD' => $httpRequest->withQueryParams($request->getData()),
            'POST', 'PUT', 'DELETE' => $httpRequest->withParsedBody($request->getData()),
            default => throw new \InvalidArgumentException('Unsupported method'),
        };
    }
}
```

以及在 JavaScript 端的用法示例：

```javascript
import {Centrifuge} from 'centrifuge';

const centrifuge = new Centrifuge('http://127.0.0.18000/connection/websocket');

// Post request
centrifuge.rpc("post:news/store", {"title": "News title"}).then(function (res) {
    console.log('rpc result', res);
}, function (err) {
    console.error('rpc error', err);
});

// Get request with query params
centrifuge.rpc("get:news/123", {"lang": "en"}).then(function (res) {
    console.log('rpc result', res);
}, function (err) {
    console.error('rpc error', err);
});
```

> **另请参阅**
> 有关使用 JavaScript SDK 和 RPC 方法的更多信息，请参阅 [documentation](https://github.com/centrifugal/centrifuge-js#rpc-method)。

## 发布请求

此服务接收一个 `RoadRunner\Centrifugo\Request\Publish` 对象，并根据连接请求执行某些操作。它应该使用 `RoadRunner\Centrifugo\Payload\PublishResponse` 对象向 Centrifugo 服务器做出响应。

要创建服务类，请使用以下命令：

```terminal
php app.php create:centrifugo-handler Publish -t=publish
```

```php app/src/Endpoint/Centrifugo/PublishService.php
namespace App\Endpoint\Centrifugo;

use RoadRunner\Centrifugo\Payload\PublishResponse;
use RoadRunner\Centrifugo\Request;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

final class PublishService implements ServiceInterface
{
    /**
     * @param Request\Publish $request
     */
    public function handle(RequestInterface $request): void
    {
        try {
            $request->respond(
                new PublishResponse(...)
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

> **另请参阅**
> 在 [Centrifugo documentation](https://centrifugal.dev/docs/server/proxy#publish-request-fields) 中阅读有关发布请求的更多信息。
