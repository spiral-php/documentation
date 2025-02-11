# GRPC — 客户端 SDK

在 [这篇文章](./service.md) 的前一部分，我们展示了如何使用 Spiral 和 `spiral/roadrunner-bridge` 软件包在 PHP 中创建一个 gRPC 服务 `Pinger`。在这一部分，我们将展示如何创建一个客户端服务 `Monitor`，用于与 `Pinger` 服务通信。

## PHP 客户端

通过遵循这些步骤，你将能够为 Pinger 服务创建一个客户端 SDK，该 SDK 可以轻松地在你的 PHP 应用程序中使用。这将使与服务通信并将其集成到你的代码库中变得更容易。

> **注意**
> 你可以在这里找到 `grpc` PHP 扩展、`protoc` 编译器和 `protoc-gen-php-grpc` 插件的 [安装说明](./configuration.md)。

### 1. 从 `.proto` 文件生成 PHP 类

为了创建客户端 SDK，我们将使用从 [前一部分](./service.md) 中的 `.proto` 文件生成的 PHP 类。你可以按照前一部分的说明轻松生成这些类。

### 2. 创建一个客户端类

我们将创建一个实现 `generated/GRPC/PingerInterface` 接口的类，并提供一个用于调用服务方法的简单接口。

```php
namespace App\Service;

use GRPC\Pinger;
use Spiral\Core\CoreInterface;
use Spiral\RoadRunner\GRPC;

final class PingerClient implements Pinger\PingerInterface extends \Grpc\BaseStub
{
    public function ping(GRPC\ContextInterface $ctx, Pinger\PingRequest $in): Pinger\PingResponse
    {
        [$response, $status] = $this->_simpleRequest(
            '/' . self::NAME . '/ping',
            $in,
            [Pinger\PingResponse::class, 'decode'],
            (array) $ctx->getValue('metadata'),
            (array) $ctx->getValue('options')
        )->wait();

        return $response;
    }
}
```

### 3. 在容器中注册客户端类

要在你的应用程序中使用客户端类，你需要将其注册到容器中。你可以在 Application bootloader 中执行此操作：

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Service\PingerClient;
use GRPC\Pinger\PingerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;

final class AppBootloader extends Bootloader
{
    protected const SINGLETONS = [
        PingerInterface::class => [self::class, 'initPingService'],
    ];

    private function initPingService(
        EnvironmentInterface $env
    ): PingerInterface {
        return new PingerClient(
            $env->get('PING_SERVICE_HOST', '127.0.0.1:9001'),
            ['credentials' => \Grpc\ChannelCredentials::createInsecure()],
        );
    }
}
```

并将 bootloader 添加到 bootloaders 列表中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\AppBootloader::class,
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
    \App\Application\Bootloader\AppBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloaders 的信息。
:::

::::

现在，客户端类已在容器中注册为单例。

### 4. 客户端用法

最后，你可以将客户端类注入到你的代码中，并使用它来调用 Pinger 服务。

这是一个如何使用 `PingerClient` 的例子：

```php app/src/Endpoint/Console/PingServiceCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Question;
use GRPC\Pinger\PingerInterface;
use Spiral\Console\Command;
use GRPC\Pinger\PingRequest;
use Spiral\RoadRunner\GRPC\Context;
use Spiral\RoadRunner\GRPC\Exception\GRPCException;

#[AsCommand(name: 'ping')]
final class PingCommand extends Command
{
    #[Argument(description: 'URL to ping')]
    #[Question(question: 'Provide URL to ping')]
    private string $url;

    public function __invoke(
        PingerInterface $client
    ): int {
        try {

            $this->writeln(\sprintf('Sending ping request [%s]...', $this->url));

            $response = $client->ping(
                new Context([]),
                new PingRequest(['url' => $this->url])
            );

            $this->writeln(\sprintf(
                'Response: code - %d',
                $response->getStatusCode()
            ));

        } catch (GRPCException $e) {

            $this->writeln(\sprintf(
                'Error: code - %d, message - %s',
                $e->getCode(),
                $e->getMessage()
            ));

        }

        return self::SUCCESS;
    }
}
```

要使用此命令，请从命令行运行它：

```terminal
php app.php ping https://google.com
```

这将调用 Pinger 服务并打印响应的 HTTP 状态码。

## Golang 客户端

GRPC 允许你使用任何受支持的语言创建客户端 SDK。要生成 Golang 客户端，请首先安装 GRPC 工具包：

### 1. 安装必要的依赖项

要在 Go 中使用 gRPC，你需要安装必要的依赖项。你可以使用 Go 包管理器执行此操作：

```bash
go get -u google.golang.org/grpc
go get -u github.com/golang/protobuf/protoc-gen-go
```

> **了解更多**
> [在这里](https://grpc.io/docs/tutorials/basic/go/) 阅读更多关于如何创建 Golang GRPC 客户端和服务器的信息。

### 2. 编译 `.proto` 文件

接下来，你需要将 `.proto` 文件编译成 Go 代码。你可以使用 protoc 编译器和 Go 插件执行此操作：

```bash
protoc -I proto/ proto/pinger.proto --go_out=plugins=grpc:pinger
```

这将生成一个 `pinger.pb.go` 文件，其中包含在 `.proto` 文件中定义的服务和消息的 Go 类。

> **注意**
> 注意 `pinger.proto` 中的 `package` 名称。

### 3. 创建客户端

现在你可以在 Go 中为 Pinger 服务创建一个客户端。下面是一个客户端的示例，它调用 Pinger 服务的 `ping()` 方法并将 HTTP 状态码打印到控制台：

```golang
package main

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"pinger"
)

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial("127.0.0.1:9001", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	
	defer conn.Close()

	client := pinger.NewPingerClient(conn)

	// Call the ping method.
	response, err := client.Ping(context.Background(), &pinger.PingRequest{
		Url: "https://google.com",
	})

	if err != nil {
		log.Fatalf("error calling ping: %v", err)
	}

	// Print the HTTP status code.
	fmt.Println(response.StatusCode)
}
```

你可以使用 Go 命令运行此客户端：

```bash
go run main.go
```

### 传递元数据

要传递服务器和客户端之间的元数据：

```golang
package main

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"google.golang.org/grpc/metadata"
	"pinger"
)

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial("127.0.0.1:9001", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	
	defer conn.Close()

	client := pinger.NewPingerClient(conn)
	
	// attach value to the server
	ctx := metadata.AppendToOutgoingContext(context.Background(), "client-key", "client-value")

	var header metadata.MD
	
	// Call the ping method.
	response, err := client.Ping(ctx, &pinger.PingRequest{
		Url: "https://google.com",
	}, grpc.Header(&header))

	if err != nil {
		log.Fatalf("error calling ping: %v", err)
	}

	// Print the HTTP status code.
	fmt.Println(response.StatusCode)
}
```

> **了解更多**
> [在这里](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md) 阅读更多关于在 Golang 中使用元数据的信息。
