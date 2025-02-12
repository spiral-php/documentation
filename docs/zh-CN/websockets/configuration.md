# Websockets — 入门

RoadRunner 曾经拥有自己的 [WebSocket 插件](https://roadrunner.dev/docs/plugins-websockets/2.x)，用于向客户端发送消息，但我们发现它的功能过于有限。 我们希望创建一个更好的解决方案来处理实时数据传输，并决定停止开发我们自己的插件。 相反，我们寻找了市场上最好的开源工具之一，并找到了 [Centrifugo](https://centrifugal.dev/)。 我们创建了一个名为 [Centrifuge](https://roadrunner.dev/docs/plugins-centrifuge) 的 RoadRunner 插件，它使用 [GRPC API](https://centrifugal.dev/docs/server/server_api#grpc-api) 与 Centrifugo 服务器集成，这是一个功能更强大、更灵活的选择。

**以下是使用 Centrifugo 的一些好处：**

-   **实时数据传输：** 该集成提供了对实时数据传输的有效管理，使其成为需要实时更新和通知的应用程序的理想解决方案。
-   **可扩展性：** Centrifugo 工具集旨在高度可扩展，允许您随着应用程序的增长轻松扩展。
-   **灵活性：** Centrifugo 提供了强大而灵活的工具集，可以轻松定制以满足您应用程序的特定需求。
-   **大型社区：** Centrifugo 拥有一个庞大而活跃的社区，他们一直在努力改进该工具，这意味着您可以访问丰富的知识和资源。
-   **完整的 Javascript SDK：** Centrifugo 提供了全面的 [Javascript SDK](https://github.com/centrifugal/centrifuge-js)，使开发人员可以轻松地与服务器交互并构建客户端功能。

通过此集成，您可以将事件发送到 WebSocket 客户端，以及在服务器上处理来自客户端的事件。

Spiral 完全集成了 RoadRunner Centrifuge 插件，这促进了 Spiral 应用程序和 Centrifugo WebSocket 服务器之间的通信。

**以下是可以在服务器上接收的事件：**

-   [**连接请求：**](https://centrifugal.dev/docs/server/proxy#connect-proxy) 当客户端与 Centrifugo 服务器建立 WebSocket 连接时，会向 RoadRunner 发送连接请求事件。
-   [**刷新连接事件：**](https://centrifugal.dev/docs/server/proxy#refresh-proxy) 当需要刷新客户端和服务器之间的连接时，Centrifugo 会向 RoadRunner 发送刷新事件。
-   [**RPC 调用：**](https://centrifugal.dev/docs/server/proxy#rpc-proxy) Centrifugo 可以通过客户端连接向 RoadRunner 发送 RPC（远程过程调用）事件。 这些事件允许您从客户端调用服务器端函数。
-   [**订阅请求：**](https://centrifugal.dev/docs/server/proxy#subscribe-proxy) 当客户端订阅频道时，Centrifugo 会向 RoadRunner 发送订阅请求事件。
-   [**发布调用：**](https://centrifugal.dev/docs/server/proxy#publish-proxy) 此请求发生在消息发布到频道 **之前**，因此您的后端可以验证客户端是否可以向频道发布数据。

## 安装

首先，您需要安装 [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) 包。

安装该包后，您可以将 `Spiral\RoadRunnerBridge\Bootloader\CentrifugoBootloader` 添加到启动加载器列表中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\CentrifugoBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动加载器](../framework/bootloaders.md) 部分阅读有关启动加载器的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\CentrifugoBootloader::class,
    // ...
];
```

在 [框架 — 启动加载器](../framework/bootloaders.md) 部分阅读有关启动加载器的更多信息。
:::

::::

## 配置

### RoadRunner 服务器

要配置 RoadRunner 和 Centrifugo 之间的通信，您需要修改 `.rr.yaml` 文件。

```yaml .rr.yaml
rpc:
  listen: tcp://0.0.0.0:6001

server:
  command: "php app.php"
  relay: pipes

centrifuge:
  proxy_address: "tcp://0.0.0.0:10001"
  grpc_api_address: "centrifugo:10000"
```

其中 `proxy_address` 是 RoadRunner Centrifuge 插件的地址。 Centrifugo 服务器将使用它与 RoadRunner 通信。

而 `grpc_api_address` 是 Centrifugo GRPC 服务器的地址。 RoadRunner Centrifuge 插件将使用它与 Centrifugo 服务器通信。

> **了解更多**
> 在 [此处](https://roadrunner.dev/docs/plugins-centrifuge) 阅读有关 Centrifuge 插件配置的更多信息。

### Centrifugo 服务器

要完成集成，您需要通过创建 `config.json` 文件并指定 Centrifugo 和 RoadRunner 之间的通信设置来配置 Centrifugo 服务器。

**这是一个 `config.json` 文件的示例，它显示了如何配置服务器之间的通信**

```json config.json
{
  // ...
  "allowed_origins": [
    "*"
  ],
  "publish": true,
  "proxy_publish": true,
  "proxy_subscribe": true,
  "proxy_connect": true,
  "allow_subscribe_for_client": true,
  "grpc_api": true,
  "grpc_api_address": "0.0.0.0",
  "grpc_api_port": 10000,
  "proxy_connect_endpoint": "grpc://127.0.0.1:10001",
  "proxy_connect_timeout": "10s",
  "proxy_publish_endpoint": "grpc://127.0.0.1:10001",
  "proxy_publish_timeout": "10s",
  "proxy_subscribe_endpoint": "grpc://127.0.0.1:10001",
  "proxy_subscribe_timeout": "10s",
  "proxy_refresh_endpoint": "grpc://127.0.0.1c:10001",
  "proxy_refresh_timeout": "10s",
  "proxy_rpc_endpoint": "grpc://127.0.0.1:10001",
  "proxy_rpc_timeout": "10s"
}
```

> **了解更多**
> 您可以在官方 [文档](https://centrifugal.dev/docs/server/configuration) 上阅读有关 Centrifugo 服务器配置的更多信息

**在此配置中**

-   `proxy_connect_endpoint` - RoadRunner 服务器地址，它将处理新的 [连接事件](https://centrifugal.dev/docs/server/proxy#connect-proxy)
-   `proxy_publish_endpoint` - RoadRunner 服务器地址，它将处理 [发布事件](https://centrifugal.dev/docs/server/proxy#publish-proxy)
-   `proxy_subscribe_endpoint` - RoadRunner 服务器地址，它将处理 [订阅事件](https://centrifugal.dev/docs/server/proxy#subscribe-proxy)
-   `proxy_refresh_endpoint` - RoadRunner 服务器地址，它将处理 [刷新事件](https://centrifugal.dev/docs/server/proxy#refresh-proxy)
-   `proxy_rpc_endpoint` - RoadRunner 服务器地址，它将处理 [rpc 事件](https://centrifugal.dev/docs/server/proxy#rpc-proxy)

Centrifugo 允许您使用多个 RoadRunner 服务器来处理不同类型的事件。这在您希望扩大应用程序可以处理的事件数量或希望提高 Centrifugo 和 RoadRunner 之间通信的可靠性的情况下非常有用。

要使用多个服务器，您需要在 `config.json` 文件中指定服务器的地址。例如，您可以配置一台 RoadRunner 服务器来处理连接事件，而另一台服务器来处理 RPC 调用，如下例所示：

```json config.json
{
  // ...
  "proxy_connect_endpoint": "grpc://127.0.0.1:10001",
  "proxy_connect_timeout": "10s",
  "proxy_rpc_endpoint": "grpc://127.0.0.1:10002",
  "proxy_rpc_timeout": "10s"
}
```

您可以根据需要使用此方法在多个 RoadRunner 服务器之间分配工作负载。

> **注意**
> 请记住，使用多个 RoadRunner 服务器将需要额外的配置和设置，您将需要确保服务器正确协调以确保平稳运行。 但是，提高可伸缩性和可靠性的好处可能非常值得付出努力。

### Spiral 应用程序

要在您的应用程序中使用 Centrifuge，您需要在 `app/config` 目录中创建一个 `centrifugo.php` 文件。 在此文件中，您可以指定将处理来自 Centrifugo 服务器的传入事件的 **服务**，以及应应用于这些事件的任何拦截器。

以下是 `centrifugo.php` 配置文件的一个示例，它显示了如何指定服务和拦截器：

```php app/config/centrifugo.php
use RoadRunner\Centrifugo\Request\RequestType;

return [
    'services' => [
        RequestType::Connect->value => ConnectService::class,
        RequestType::Subscribe->value => SubscribeService::class,
        RequestType::Refresh->value => RefreshService::class,
        RequestType::Publish->value => PublishService::class,
        RequestType::RPC->value => RPCService::class,
    ],
    'interceptors' => [
        RequestType::Connect->value => [
            Interceptor\AuthInterceptor::class,
        ],
        RequestType::Subscribe->value => [
            Interceptor\AuthInterceptor::class,
        ],
        RequestType::RPC->value => [
            Interceptor\AuthInterceptor::class,
        ],
        '*' => [
            Interceptor\TelemetryInterceptor::class,
        ],
    ],
];
```

> **了解更多**
> 有关事件处理程序（**服务**）及其在 Spiral 应用程序中的使用方法的更多信息，您可以参考 [事件处理程序](./services.md) 部分的文档。 此页面提供了其他详细信息和示例，可帮助您开始使用事件处理。

### Javascript 客户端

您可以使用官方 [JavaScript SDK](https://github.com/centrifugal/centrifuge-js) for Centrifugo 来促进您的应用程序中 Centrifugo 服务器和客户端浏览器之间的通信。 它提供了一组 API，允许您连接到 Centrifugo 服务器、订阅频道并实时接收消息。

使用 JavaScript SDK，您可以在客户端和 Centrifugo 服务器之间建立 WebSocket 连接，并且 RoadRunner 和客户端之间的所有通信都将通过 Centrifugo 服务器进行路由。 这意味着您无需在服务器上打开任何其他端口即可支持客户端和服务器之间的实时通信。

## Nginx 代理

要使用 Nginx 代理启用与 Centrifugo 服务器的 WebSocket 连接，您需要相应地配置代理。

这可以通过在 Nginx 配置文件中包含以下配置来完成：

```
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 8000;

    server_name _;

    # Centrifugo WebSocket endpoint
    location /connection {
        proxy_pass http://127.0.0.1:8000/connection;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
    }
    
    # RoadRunner HTTP endpoint
    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

> **了解更多**
> 您可以在 [官方文档](https://centrifugal.dev/docs/server/load_balancing) 上阅读有关 Nginx 代理配置的更多信息。

## 示例应用程序

有一个很好的例子 [演示票务预订系统](https://github.com/spiral/ticket-booking) 应用程序构建在 Spiral 上，Spiral 是一个高性能 PHP 框架，它遵循微服务的原则，允许开发人员创建可重用、独立且易于维护的组件。

它演示了如何使用 RoadRunner 的 Centrifuge 插件来启用 Centrifugo 服务器和客户端浏览器之间的实时通信。
