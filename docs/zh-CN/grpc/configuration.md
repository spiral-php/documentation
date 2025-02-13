# GRPC — 快速入门

长期以来，使用 REST APIs 在微服务之间进行通信是一种常见的方法。然而，REST APIs 存在一些限制和挑战，这些限制和挑战会影响系统的性能、可扩展性和可靠性。

从 REST APIs 切换到 gRPC 可以帮助解决这些问题。[gRPC](https://grpc.io/) 协议为分布式应用程序提供了极其高效的跨服务通信方式。公共工具包包括为许多语言生成客户端和服务器代码库的工具，允许开发人员为他们的任务使用最优化语言。

> **了解更多**
> 你可以在[这里](https://developers.google.com/protocol-buffers/docs/overview)阅读更多关于 protobuf 的信息。

## 工具包安装

无需任何依赖即可开箱即用地运行 PHP 应用程序。但是，为了开发、调试和扩展 GRPC 项目，你将需要几个工具。

### 安装 Protoc

为了将 `.proto` 文件编译成目标语言，你必须安装 `protoc` 编译器。

> **注意**
> 你可以从[这里](https://github.com/protocolbuffers/protobuf/releases)下载最新的 `protoc` 二进制文件。

### 安装 Protobuf 扩展（可选）

为了使用更大的消息实现更好的性能，请确保安装 PHP 的 `protobuf` 扩展。

你可以手动编译该扩展，或者通过 [PECL](https://pecl.php.net/package/protobuf) 安装它。

```bash
sudo pecl install protobuf
```

**如果出现 `Segmentation Fault` 错误，请尝试安装另一个 `protobuf` 库。我们建议一开始使用 `3.10.0`。**

```bash
sudo pecl install protobuf-3.10.0
```

## 包安装

`spiral/roadrunner-bridge` 包允许你将 RoadRunner 的
[gRPC 插件](https://roadrunner.dev/docs/app-server-grpc) 与 Spiral 结合使用。该包提供了生成 proto 文件、客户端代码和应用程序引导器的工具。

首先，你需要安装 [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) 包。

安装该包后，使用引导器 `Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader` 激活该组件：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader::class,
        \Spiral\RoadRunnerBridge\Bootloader\CommandBootloader::class,
        // ...
    ];
}
```

在[框架 — 引导器](../framework/bootloaders.md) 部分阅读更多关于引导器的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader::class,
    \Spiral\RoadRunnerBridge\Bootloader\CommandBootloader::class,
    // ...
];
```

在[框架 — 引导器](../framework/bootloaders.md) 部分阅读更多关于引导器的信息。
:::

::::

安装包后，你需要下载 `protoc-gen-php-grpc`。 它是 `protoc` 工具的插件，用于为 gRPC 服务生成 PHP 代码。

> **注意**
> 要下载二进制文件，你可以使用 `spiral/roadrunner-cli` 包提供的 `download-protoc-binary` 命令行工具。此命令将下载最新版本的 `protoc-gen-php-grpc` 二进制文件，并将其保存到你项目的根目录。

```terminal
./vendor/bin/rr download-protoc-binary
```

## 配置

如果要配置生成服务类，请创建配置文件 `app/config/grpc.php`：

```php app/config/grpc.php
return [
    /**
     * 存放生成的 DTO（数据传输对象）文件的路径。
     */
    'generatedPath' => directory('root') . '/GRPC',

    /**
     * 所有 proto 文件的根目录，将在其中搜索导入文件。
     */
    'servicesBasePath' => directory('root') . '/proto',

    /**
     * protoc-gen-php-grpc 库的路径。
     */
    'binaryPath' => directory('root').'/bin/protoc-gen-php-grpc',

    /**
     * 一个 proto 文件路径的数组，这些文件应该通过 grpc:generate 命令行编译成 PHP。
     */
    'services' => [
        //directory('root').'proto/calculator.proto',
    ],
];
```

## 应用服务器

要在 RoadRunner 应用服务器中启用该组件，请添加以下配置节：

```yaml .rr.yaml
server:
  command: "php app.php"

grpc:
  # 要监听的 GRPC 地址
  listen: "tcp://0.0.0.0:9001"
  proto:
    - "proto/calculator.proto"
```

> **注意**
> 关于配置 RoadRunner gRPC 插件的完整文档[在这里](https://roadrunner.dev/docs/app-server-grpc)。

## 示例应用程序

有一个很好的示例 [**Demo ticket booking system（演示票务预订系统）**](https://github.com/spiral/ticket-booking) 应用程序，它基于 Spiral 构建，遵循微服务原则，并允许开发人员创建可重用、独立且易于维护的组件。

在这个演示应用程序中，你可以找到一个使用 RoadRunner 的 gRPC 插件创建和使用 gRPC 服务的示例。

总的来说，我们的演示票务预订系统是 Spiral 和其他工具如何用于构建现代化和高效应用程序的一个很好的例子。 我们希望你使用它并了解更多关于其功能的知识。

**祝你（假）购票愉快！**

## 生成证书

无需任何加密层即可运行 GRPC。但是，为了保护我们的应用程序，我们必须颁发适当的服务器密钥和证书。 你可以使用任何常规的 SSL 证书（例如，由 [https://letsencrypt.org/](https://letsencrypt.org/) 颁发的证书）或通过 [OpenSSL](https://www.openssl.org/) 手动颁发它。

要颁发服务器密钥和证书：

```bash
openssl req -newkey rsa:2048 -nodes -keyout app.key -x509 -days 365 -out app.crt
```

> **注意**
> 确保使用适当的域名或 `localhost`，这将是你的客户端正确连接所必需的。
