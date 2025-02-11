# HTTP — 请求生命周期

与其他大多数 PHP 框架不同，HTTP 请求在应用程序服务器之外开始。

- [RoadRunner](https://roadrunner.dev)

![请求生命周期](https://user-images.githubusercontent.com/773481/181182150-8cc2b6c4-2b50-4e85-afd7-e5c2d1c98b2c.png)

> **注意**
> 响应以相反的方向返回。

## PSR

Spiral 基于一组标准，这使得它与其它框架、路由器、中间件等兼容。您可以在这里了解更多关于使用的 PSR 标准：

- [PSR-7：HTTP 消息接口](https://www.php-fig.org/psr/psr-7/)
- [PSR-15：HTTP 服务器请求处理器](https://www.php-fig.org/psr/psr-15/)
- [PSR-17：HTTP 工厂](https://www.php-fig.org/psr/psr-17/)

## 流程描述

用户请求到达 RoadRunner 应用程序服务器。服务器将通过许多中间件层传递它，其中一些用于启用静态文件服务或实现特定于域的逻辑。

一旦所有中间件处理完成，`net/http` 请求将被转换为 `PSR-7` 格式并传递给第一个可用的 PHP 工作进程。

工作进程将使用 `spiral/http` 组件和 `Spiral\Http\Http` 核心来处理此请求。该核心会将 PSR-7 请求对象传递给一组与 PSR-15 兼容的中间件（`Psr\Http\Message\ServerRequestInterface`）。

一旦所有中间件处理完成，框架将为请求对象创建一个 [IoC 作用域](../advanced/ioc.md)。这种方法允许您使用 PSR-7 请求作为经典的全局对象，而从技术上讲，它仅在用户请求期间存在。

请求被传递到您选择的 PSR-15 处理器 (默认为 `spiral/router`)。 处理器必须生成一个响应，该响应将通过所有中间件层发送回用户。

> **注意**
> Spiral 路由器提供了将自定义中间件集合与每个路由关联的能力。

## 手动调用 HTTP

可以在应用程序内调用 HTTP 核心。这对于测试目的或如果你想从其他框架内部运行 Spiral 框架是很有用的。要做到这一点，获取 `Spiral\Http\Http` 的实例:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Nyholm\Psr7\Uri;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Http\Http;

class HomeController implements SingletonInterface
{
    public function __construct(
        private readonly Http $http
    ) {
    }

    public function index(ServerRequestInterface $request): string
    {
        $response = $this->http->handle(
            $request->withUri(new Uri('/home/other')) // modify Uri of current request
        );

        return (string) $response->getBody(); // "other"
    }

    public function other(): string
    {
        return 'other';
    }
}
```

> **注意**
> IoC 作用域可以嵌套，因此所有功能都将正常工作。但是，请注意，并非所有扩展都允许嵌套 (您尚不允许创建嵌套会话)。

## 事件

| 事件                             | 描述                                                       |
|-----------------------------------|------------------------------------------------------------|
| Spiral\Http\Event\RequestReceived | 收到请求时将触发此事件。                                  |
| Spiral\Http\Event\RequestHandled  | 成功处理请求时将触发此事件。                                |

> **注意**
> 要了解更多关于派发事件的信息，请参阅文档中的 [事件](../advanced/events.md) 部分。
