# 开始使用 — 长时间运行

Spiral 框架旨在促进长时间运行的应用程序的开发，同时确保高效的内存管理和防止内存泄漏。这通过利用先进的 [内存管理技术](../container/scopes.md) 来实现。此外，它还与 RoadRunner 配对使用，进一步增强了应用程序的整体性能和可扩展性。

RoadRunner 是一个高性能的 PHP 应用程序服务器和进程管理器，它为 PHP 应用程序提供了长时间运行的能力。它提供了高效的资源管理，例如 CPU 和内存使用，这有助于应用程序长时间平稳高效地运行。

它被设计用来处理各种请求类型，包括 [HTTP](../http/lifecycle.md)、[gRPC](../grpc/configuration.md)、TCP、[队列任务](../queue/roadrunner.md) 消费和 Temporal。它的运行方式是在启动时只运行一次 worker，然后根据请求的类型将请求定向到 [调度器](../framework/dispatcher.md)。这意味着每个 worker 都是隔离的，并且独立工作，遵循“无共享”的方法，其中资源不会在 worker 之间共享。

使用 RoadRunner 可以显著提高速度和效率，因为消除了应用程序反复进行引导启动的需要。这可以节省 CPU 和内存资源，并减少响应时间。

> **另请参阅**
> 在 [Framework — Application Lifecycle](../framework/lifecycle.md) 部分阅读有关框架和应用程序服务器共生的更多信息。

## 安装

使用 RoadRunner 相对简单。一旦你下载了二进制文件，就可以用它来运行你的 PHP 应用程序。

有几种下载方式：

:::: tabs

::: tab Composer

最好的方法是使用 composer 包 `spiral/roadrunner-cli`。它将帮助你自动下载服务器。

只需在你的项目中安装该软件包并运行以下命令：

```terminal
composer require spiral/roadrunner-cli
```

并运行以下命令以下载最新版本的 RoadRunner：

```terminal
./vendor/bin/rr get
```

> **警告**
> 下载 RoadRunner 需要 PHP 的扩展 `php-curl` 和 `php-zip`。
:::

::: tab cURL

使用 cURL 下载 RoadRunner 的最新稳定版本。

```bash
curl --proto '=https' --tlsv1.2 -sSf  https://raw.githubusercontent.com/roadrunner-server/roadrunner/master/download-latest.sh | sh
```
:::

::: tab Docker

RoadRunner 在 Docker 镜像中提供了预编译的 RoadRunner 服务器二进制文件。

```docker Dockerfile
FROM spiralscout/roadrunner as roadrunner
# OR
# FROM ghcr.io/roadrunner-server/roadrunner as roadrunner

FROM php:8.1-cli

# 将 RoadRunner 二进制文件从 roadrunner 镜像复制到本地 bin 目录
COPY --from=roadrunner /usr/bin/rr /usr/local/bin/rr

# 运行 RoadRunner 服务器命令
CMD ["rr", "serve"]
```

**该镜像可在以下位置获取：**

- **Github** — [ghcr.io/roadrunner-server/roadrunner](https://github.com/roadrunner-server/roadrunner/pkgs/container/roadrunner)
- **Docker Hub** — [spiralscout/roadrunner](https://hub.docker.com/r/spiralscout/roadrunner)

:::

::: tab Linux
适用于 Debian 衍生版本（Ubuntu, Mint, MX 等）的安装选项

```bash
wget https://github.com/roadrunner-server/roadrunner/releases/download/v2.X.X/roadrunner-2.X.X-linux-amd64.deb
sudo dpkg -i roadrunner-2.X.X-linux-amd64.deb
```
:::

::: tab Github

如果其他安装选项都不适合你，你可以随时直接在 GitHub 上下载 RoadRunner 二进制文件。

转到 [最新的 RoadRunner 版本](https://github.com/roadrunner-server/roadrunner/releases/latest)，向下滚动到 "Assets"，然后选择与你的操作系统相对应的二进制文件。

:::

::::

## 配置

你可以使用 `.rr.yaml` 文件配置 worker 的数量、内存限制和其他插件：

```yaml .rr.yaml
rpc:
  listen: tcp://127.0.0.1:6001

server:
  command: "php app.php"
  relay: pipes

# HTTP plugin settings
http:
  address: 0.0.0.0:8080
  middleware: [ "gzip", "static" ]
  static:
    dir: "public"
    forbid: [ ".php", ".htaccess" ]
  pool:
    num_workers: 2
    supervisor:
      max_worker_memory: 100
```

设置 HTTP 的 worker 数量：

```yaml .rr.yaml
http:
  pool:
    num_workers: 4
```

> **另请参阅**
> 在官方 [文档](https://roadrunner.dev/docs) 中阅读有关应用程序服务器配置的更多信息。

## 运行服务器

:::: tabs

::: tab Linux

在 **Linux** 上使用以下命令启动应用程序服务器

```terminal
./rr serve
```

> **警告**
> 确保 `rr` 二进制文件是可执行的。

:::

::: tab Windows
在 **Windows** 上使用以下命令启动应用程序服务器

```terminal
./rr.exe serve
```

:::

::::

> **另请参阅**
> 在 [RoadRunner 文档](https://roadrunner.dev/docs/app-server-cli) 中阅读有关服务器命令的更多信息。

## RoadRunner 桥接器

[spiral/roadrunner-bridge](https://github.com/spiral/roadrunner-bridge) 软件包提供了 Spiral 和 RoadRunner 之间的完全集成。此软件包允许开发人员使用 RoadRunner 的各种插件，包括 `http`、`grpc`、`jobs`、`tcp`、`kv`、`locks`、`centrifugo`、`app-logger` 和 `metrics`。

> **注意**
> 默认情况下，该组件在 [应用程序包](https://github.com/spiral/app) 中可用。

### 安装

要安装该软件包，请运行以下命令：

```terminal
composer require spiral/roadrunner-bridge
```

安装后，你需要在 `Kernel` 中将该软件包的 bootloader 添加到你的应用程序中，选择与你想要使用的插件相对应的特定 bootloader：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
use Spiral\RoadRunnerBridge\Bootloader as RoadRunnerBridge;

public function defineBootloaders(): array
{
    return [
        RoadRunnerBridge\HttpBootloader::class, // 可选，如果需要与 http 插件一起使用
        RoadRunnerBridge\QueueBootloader::class, // 可选，如果需要与 jobs 插件一起使用
        RoadRunnerBridge\CacheBootloader::class, // 可选，如果需要与 KV 插件一起使用
        RoadRunnerBridge\GRPCBootloader::class, // 可选，如果需要与 GRPC 插件一起使用
        RoadRunnerBridge\CommandBootloader::class,
        RoadRunnerBridge\TcpBootloader::class, // 可选，如果需要与 TCP 插件一起使用
        RoadRunnerBridge\MetricsBootloader::class, // 可选，如果需要与 metrics 插件一起使用
        RoadRunnerBridge\LoggerBootloader::class, // 可选，如果需要与 app-logger 插件一起使用
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关 bootloader 的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
use Spiral\RoadRunnerBridge\Bootloader as RoadRunnerBridge;

protected const LOAD = [
    RoadRunnerBridge\HttpBootloader::class, // 可选，如果需要与 http 插件一起使用
    RoadRunnerBridge\QueueBootloader::class, // 可选，如果需要与 jobs 插件一起使用
    RoadRunnerBridge\CacheBootloader::class, // 可选，如果需要与 KV 插件一起使用
    RoadRunnerBridge\GRPCBootloader::class, // 可选，如果需要与 GRPC 插件一起使用
    RoadRunnerBridge\CommandBootloader::class,
    RoadRunnerBridge\TcpBootloader::class, // 可选，如果需要与 TCP 插件一起使用
    RoadRunnerBridge\MetricsBootloader::class, // 可选，如果需要与 metrics 插件一起使用
    RoadRunnerBridge\LoggerBootloader::class, // 可选，如果需要与 app-logger 插件一起使用
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关 bootloader 的更多信息。
:::

::::

## 注意水下暗礁

在使用长时间运行的应用程序时，需要注意一些限制。

### 应用程序状态

当使用 RoadRunner 服务器运行你的应用程序时，文件中的任何更改都不会影响你的应用程序，因为启动后它会被加载到内存中。要查看更改，你需要重启服务器。这可能会导致一些不便。

要强制在每个请求后重新加载 worker（完全调试模式）并将处理限制为单个 worker，请添加 `debug` 选项。这对于调试和开发目的非常有用。

```yaml .rr.yaml
http:
  pool:
    debug: true
```

> **警告**
> 重要的是要注意，此功能会影响应用程序服务器的性能，因此最好仅在开发模式下使用它。

### 内存泄漏

由于应用程序长时间驻留在内存中，即使是小的内存泄漏也可能导致进程重启。RoadRunner 监视内存消耗并执行软重置，但最好避免在你的应用程序源代码中出现内存泄漏。

> **注意**
> Framework 包含一组工具，可以简化开发过程并避免内存/状态泄漏，例如 IoC Scopes、Cycle ORM、Immutable Configs、Domain Cores、Routes 和 Middleware。

<hr>

## 接下来做什么？

现在，通过阅读一些文章更深入地了解基础知识：

* [应用程序生命周期](../framework/lifecycle.md)
* [调度器](../framework/dispatcher.md)
* [终结器](../framework/finalizers.md)
* [静态内存](../advanced/memory.md)
* [自定义调度器](../cookbook/custom-dispatcher.md)
