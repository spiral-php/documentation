# GRPC — 服务代码

在某些情况下，使用 gRPC 可能比构建 REST API 更简单，因为它提供了一种更结构化和高效的方式来定义和在 API 之间进行通信。

与 REST API 相比，使用 gRPC 具有以下优势：

- **强类型接口：** 在 gRPC 中，服务和消息类型在 `.proto` 文件中定义，这允许创建强类型接口。这可以更容易地确保在客户端和服务器之间传递正确的数据，因为类型是在编译时检查的。

- **高效的二进制编码：** gRPC 使用 Protocol Buffers（一种二进制编码格式）来序列化和传输数据。这可能比 JSON（REST API 的常用编码格式）更高效，因为它需要更少的字节来传输相同的数据。

- **语言和平台无关性：** gRPC 使用通用的 `.proto` 文件来定义服务和消息类型，该文件可用于在多种语言和平台上生成代码。这允许创建跨平台的 API，这些 API 可以被任何语言编写的客户端使用。

> **注意**
> 使用 https://github.com/spiral/ticket-booking 作为基础来加快入门速度。

## 定义服务

要声明我们的第一个服务，请在所需位置创建一个 proto 文件。默认情况下，GRPC 构建建议在 `/proto` 目录中创建 proto 文件。创建一个文件 `proto/pinger.proto`：

```proto proto/pinger.proto
syntax = "proto3";

option php_namespace = "GRPC\\Pinger";
option php_metadata_namespace = "GRPC\\GPBMetadata";

package pinger;

service Pinger {
  rpc ping (PingRequest) returns (PingResponse) {
  }
}

message PingRequest {
  string url = 1;
}

message PingResponse {
  int32 status_code = 1;
}
```

此 `.proto` 文件定义了一个名为 **Pinger** 的服务，该服务具有一个方法 `ping`，该方法接受 `PingRequest` 消息作为输入并返回 `PingResponse` 消息。

> **了解更多**
> 确保使用选项 `php_namespace` 和 `php_metadata_namespace` 正确配置 PHP 命名空间。您可以在[此处](https://grpc.io/docs/guides/concepts/)阅读有关 GRPC 服务声明的更多信息。

## 生成服务

要将 `.proto` 文件编译成 PHP 代码，您需要使用 `protoc` 编译器和 `protoc-gen-php-grpc`。

> **注意**
> 可以在[此处](./configuration.md)找到 `grpc` PHP 扩展、`protoc` 编译器和 `protoc-gen-php-grpc` 插件的[安装说明]。

首先，您需要生成服务存根。为此，您需要将它们添加到 `app/config/grpc.php` 配置文件中：

```php app/config/grpc.php
return [
    /**
     * 生成的 DTO（数据传输对象）文件将存储的路径。
     */
    'generatedPath' => directory('root') . '/generated',

    /**
     * 所有 proto 文件的根目录，将在此处搜索导入。
     */
    'servicesBasePath' => directory('root') . '/proto',

    /**
     * protoc-gen-php-grpc 库的路径。
     */
    'binaryPath' => directory('root').'protoc-gen-php-grpc',

    /**
     * 应该由 grpc:generate 控制台命令编译成 PHP 的 proto 文件路径数组。
     */
    'services' => [
        directory('root').'proto/pinger.proto',
    ],
];
```

然后，您可以使用以下命令编译 `pinger.proto` 文件：

```terminal
php app.php grpc:generate
```

您应该看到以下输出：

```bash
Compiling `proto/pinger.proto`:
• generated/GRPC/Pinger/PingerInterface.php
• generated/GRPC/Pinger/PingRequest.php
• generated/GRPC/Pinger/PingResponse.php
• generated/GRPC/GPBMetadata/Pinger.php
```

代码将被生成在 `generated/GRPC/Pinger` 和 `generated/GRPC/GPBMetadata` 目录中。

最后一步是在 `composer.json` 文件中注册 `GRPC` 命名空间：

```json composer.json
{
    ...
    "autoload": {
        "psr-4": {
            "App\\": "app/src",
            "GRPC\\": "generated"
        }
    }
}
```

并运行 `composer dump-autoload` 来更新自动加载器。

## 实现服务

接下来，您需要创建一个 PHP 类，该类实现 `.proto` 文件中定义的 **Pinger** 服务。此类应该扩展从 `.proto` 文件生成的 `GRPC/PingerInterface` 类，并实现 `ping()` 方法：

```php app/src/Endpoint/GRPC/Pinger.php
namespace App\Endpoint\GRPC;

use Spiral\RoadRunner\GRPC;
use GRPC\Pinger\PingerInterface;
use GRPC\Pinger\PingRequest;
use GRPC\Pinger\PingResponse;

final class Pinger implements PingerInterface
{
    public function __construct(
        private readonly HttpClientInterface $httpClient
    ) {
    }
    
    public function ping(GRPC\ContextInterface $ctx, PingRequest $in): PingResponse
    {
        $statusCode = $this->httpClient->get($in->getUrl())->getStatusCode();
    
        return new PingResponse([
            'status_code' => $statusCode
        ]);
    }
}
```

## 启动 gRPC 服务器：

确保在 `.rr.yaml` 中更新 proto 路径：

```yaml .rr.yaml
grpc:
  listen: tcp://0.0.0.0:9001
  proto:
    - "proto/pinger.proto"
```

```terminal
./rr serve
```

### 多个服务

使用 proto 声明的 `import` 指令将多个服务组合在一个应用程序中，或单独存储消息声明。

## 元数据

使用 `Spiral\GRPC\ContextInterface` 访问请求元数据。您可以读取许多系统元数据属性：

```php
use Spiral\RoadRunner\GRPC;
use GRPC\Pinger\PingRequest;
use GRPC\Pinger\PingResponse;

public function ping(GRPC\ContextInterface $ctx, PingRequest $in): PingResponse
{
    dump($ctx->getValue(':authority'));
    dump($ctx->getValue(':peer.address'));
    dump($ctx->getValue(':peer.auth-type'));

    dump($ctx->getValue('user-agent'));
    dump($ctx->getValue('content-type'));
    
    return new PingResponse([
        'status_code' => $this->httpClient->get($in->getUrl())->getStatusCode()
    ]);
}
```

> **了解更多**
> 在[此处](https://grpc.io/docs/guides/auth/)阅读有关身份验证实践的更多信息。

### 响应头

您可以使用特定于上下文的响应头向响应添加任何自定义元数据：

```php
use Spiral\RoadRunner\GRPC;
use GRPC\Pinger\PingRequest;
use GRPC\Pinger\PingResponse;

public function ping(GRPC\ContextInterface $ctx, PingRequest $in): PingResponse
{
    /** @var GRPC\ResponseHeaders $responseHeaders */
    $responseHeaders = $ctx->getValue(GRPC\ResponseHeaders::class);
    $responseHeaders->set('key', 'value');
    
    return new PingResponse([
        'status_code' => $this->httpClient->get($in->getUrl())->getStatusCode()
    ]);
}
```

## 错误

`spiral/roadrunner-grpc` 组件提供了许多异常来指示服务器或请求错误：

| 异常                                                      | 错误代码          |
|----------------------------------------------------------------|---------------------|
| Spiral\RoadRunner\GRPC\Exception\\**GRPCException**            | UNKNOWN(2)          |
| Spiral\RoadRunner\GRPC\Exception\\**InvokeException**          | UNAVAILABLE(14)     |
| Spiral\RoadRunner\GRPC\Exception\\**NotFoundException**        | NOT_FOUND(5)        |
| Spiral\RoadRunner\GRPC\Exception\\**ServiceException**         | INTERNAL(13)        |
| Spiral\RoadRunner\GRPC\Exception\\**UnauthenticatedException** | UNAUTHENTICATED(16) |
| Spiral\RoadRunner\GRPC\Exception\\**UnimplementedException**   | UNIMPLEMENTED(12)   |

> **了解更多**
> 在 `Spiral\RoadRunner\GRPC\StatusCode` 中查看所有状态代码。在[此处](https://github.com/grpc/grpc/blob/master/doc/statuscodes.md)阅读有关 GRPC 错误代码的更多信息。

### 示例：

```php
use Spiral\RoadRunner\GRPC;
use GRPC\Pinger\PingRequest;
use GRPC\Pinger\PingResponse;

public function ping(GRPC\ContextInterface $ctx, PingRequest $in): PingResponse
{
    if ($in->getUrl() === '') {
        throw new GRPC\ServiceException('URL is empty');
    }
    
    if (!\filter_var($url, FILTER_VALIDATE_URL)) {
        throw new GRPC\ServiceException(\sprintf('URL "%s" is invalid', $url));
    }

    return new PingResponse([
        'status_code' => $this->httpClient->get($in->getUrl())->getStatusCode(),
    ]);
}
```

## 最佳实践

为 spiral 应用程序设计 GRPC API 的推荐方法是在单独的存储库中生成服务代码接口、消息和客户端代码。

通用：

- *Image-SDK* - v1.2.0

服务：

- **Image-Service** - 实现 *Image-SDK* v1.2.0
- **Account-Service** - 需要 *Image-SDK* v1.1.0
