# 基础知识 — 调试

在使用 Spiral 框架和 RoadRunner 开发长时间运行的应用程序时，调试代码需要考虑一些特定的挑战。以下是您需要了解的内容：

## 常见的调试技术

### 避免使用 `die` 和 `exit` 函数

在传统的 PHP 开发中，您可能习惯使用 `die` 或 `exit` 函数来停止脚本执行，通常与像 `var_dump` 这样的转储函数结合使用。但是，在 Spiral 环境中，使用 `die` 或 `exit` 可能会破坏您的应用程序，因为它会完全停止 RoadRunner worker，而不仅仅是当前请求。

例如，使用 `symfony/var-dumper` 包中的 `dd` 函数可能会导致问题，因为此函数会转储变量的内容，然后调用 `die`，从而破坏您的 RoadRunner worker。

### 处理 `PHP_SAPI` 不匹配

RoadRunner 不使用传统的 PHP SAPI (Server API)，并且一些转储器是建立在它们在 CLI (命令行界面) 环境中工作的假设之上。当您尝试调试应用程序时，这种差异可能会导致意外的行为。

## Spiral Dumper

我们开发了 `spiral/dumper` 包来解决这些问题。此包充当 [symfony/var-dumper](https://symfony.com/doc/current/components/var_dumper.html) 库的包装器，允许您将变量转储直接发送到 HTTP worker 中的浏览器，或者在其他环境中发送到 `STDERR` 输出。它设计为与 RoadRunner 的长时间运行方式良好配合。

通过 `spiral/dumper` 包，开发人员可以在开发过程中轻松地检查和分析变量值。这个包是调试和排除 Web 和 CLI 应用程序故障的宝贵资产。

### 安装

默认情况下，`spiral/dumper` 包已包含在 `spiral/app` skeleton 中。但是，如果您使用不同的 skeleton，您可以使用以下命令轻松安装该包：

```terminal
composer require --dev spiral/dumper
```

安装后，您需要将该包的 bootloader 添加到您的应用程序中。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineSystemBootloaders(): array
{
    return [
        // ...
        \Spiral\Debug\Bootloader\DumperBootloader::class,
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关 bootloader 的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const SYSTEM = [
    // ...
    \Spiral\Debug\Bootloader\DumperBootloader::class,
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关 bootloader 的更多信息。
:::

::::

### 使用

要转储变量，只需使用该包提供的辅助函数 `dump()` 即可。

```php
dump($variable);
```

通过此包，您可以像在传统的 PHP 应用程序中一样使用 `dd` 函数，但没有停止整个 RoadRunner worker 的风险。

```php
dd($variable);
```

<hr />

## Symfony VarDumper

或者，对于更传统的调试方法，您可以选择使用 symfony/var-dumper 包。

此包提供了一个独立的服务器，用于收集所有转储的数据。您使用命令启动服务器，它将侦听由 dump() 函数发送的数据。您发送给此函数的任何变量转储都将显示在单独的控制台窗口中，而不是在您的主要应用程序输出中。

这是一个控制台输出示例：

```terminal
./vendor/bin/var-dump-server

Symfony Var Dumper Server
=========================
 [OK] Server listening on tcp://127.0.0.1:9912
 // Quit the server with CONTROL-C.

$ app.php
---------
 -------- ---------------------------------------------------------
  date     Fri, 18 Aug 2023 11:54:44 +0000
  source   SimpleController.php on line 36
  file     app/src/Interfaces/Http/Controller/SimpleController.php
 -------- ---------------------------------------------------------
null

 -------- ---------------------------------------------------------
  date     Fri, 18 Aug 2023 11:54:44 +0000
  source   SimpleController.php on line 37
  file     app/src/Interfaces/Http/Controller/SimpleController.php
 -------- ---------------------------------------------------------
App\Service\Site\Site^ {#1260
  -theme: "default"
  -docs: App\Service\Site\Docs^ {#1269
    -defaultVersion: "3.5"
    -defaultLanguage: "en"
  }
  -host: "127.0.0.1"
}
```

### 安装

要安装该包，请执行以下命令：

```terminal
composer require --dev symfony/var-dumper
```

### 使用

要启动服务器，请运行以下命令：

```terminal
./vendor/bin/var-dump-server
```

要使用此功能，您还需要在您的 `.env` 文件中定义 `VAR_DUMPER_FORMAT` 环境变量，如下所示：

```dotenv .env
VAR_DUMPER_FORMAT=server
```

### 已知问题

如果一个对象有许多属性，或者这些属性包含大量数据，则控制台输出可能会变得不堪重负。在这些情况下，很难在控制台中筛选大量文本以找到您感兴趣的特定信息。

当您处理复杂的对象（例如具有许多关系的 ORM 实体或大型数组）时，这个问题会变得更加严重，这些对象具有深层和宽泛的结构。控制台作为线性且大小有限的输出，可能难以以可读和可导航的方式呈现此信息。

<hr />

## 使用 Buggregator 进行高级调试

[Buggregator](https://github.com/buggregator/spiral-app) 是一个强大的、容器化的 Web 应用程序和服务器，旨在显着增强您在 PHP 开发中的调试体验。它监听 TCP 和 HTTP 端口，允许它处理各种传入请求，包括变量转储、异常、应用程序日志、SMTP 邮件等。

![var-dumper](https://user-images.githubusercontent.com/773481/208727353-b8201775-c360-410b-b5c8-d83843d388ff.png)

### 主要特点

1.  **捕获和显示 PHP 变量：** 轻松与 [Symfony var-dumper](https://github.com/buggregator/spiral-app#2-symfony-vardumper-server) 等工具集成，以有组织、可读的格式捕获和显示转储的变量。

2.  **异常处理：** 它可以捕获和显示异常，包括由错误跟踪平台（例如 [Sentry](https://github.com/buggregator/spiral-app#4-compatible-with-sentry-reports)）发送的异常，从而为您提供清晰、集中的问题视图。

3.  **SMTP 邮件捕获器：** 它可以充当 [模拟 SMTP 服务器](https://github.com/buggregator/spiral-app#3-fake-smtp-server-for-catching-mail)，捕获和显示您的应用程序在开发过程中发送的电子邮件，以便您可以在不实际发送它们的情况下查看和测试电子邮件。

4.  **用户友好的界面：** 提供一个干净、直观的 Web UI，它以易于导航和理解的方式组织和显示您的调试数据。

5.  **容器化，易于设置：** Buggregator 打包为 Docker 容器，这使得在任何开发环境中启动和运行变得非常简单。

当处理具有众多属性和大量数据的复杂对象时，筛选控制台输出或日志文件可能会让人不堪重负，并且非常耗时。 Buggregator 通过以结构化、可折叠和可搜索的 Web 界面呈现此信息来解决此问题。这样，您就可以快速有效地找到您需要的确切数据，而无需滚动数百行文本。

### 安装

要开始使用 Buggregator，只需拉取 Docker 镜像并运行容器即可：

```bash 最新稳定版本
docker run --pull always ghcr.io/buggregator/server:latest
    -p 8000:8000
    -p 1025:1025
    -p 9912:9912
    -p 9913:9913
```

或者，如果您想将其与 docker-compose 一起使用，请将以下服务添加到您的 `docker-compose.yaml` 文件中：

```yaml docker-compose.yaml
services:
  # ...
  buggregator:
    image: ghcr.io/buggregator/server:latest
    ports:
      - 8000:8000
      - 1025:1025
      - 9912:9912
      - 9913:9913
```

Buggregator 运行后，在您的 Web 浏览器中导航到 http://127.0.0.1:8000 以访问 Buggregator 界面，并开始实时监控您的应用程序的调试数据。

> **注意**
> 关于配置您的应用程序以将数据发送到 Buggregator 的信息，请参阅
> [GitHub 存储库](https://github.com/buggregator/server#features)

### XHProf 集成

Buggregator 不仅可以作为捕获和显示调试数据的全面工具，而且还可以作为应用程序分析的宝贵合作伙伴。它可以用作 [Xhprof](https://github.com/buggregator/spiral-app#1-xhprof-profiler) 配置文件的一个观察者，为开发人员提供了一种直观而有效的方式来分析性能数据，确定瓶颈，并发现其 PHP 应用程序中的内存泄漏。

![xhprof](https://user-images.githubusercontent.com/773481/208724383-3790a3e1-9ebe-4616-8d4d-d1869f8f2b7c.png)

**它可以通过以下方式使用：**

1.  **方便的瓶颈识别：** Buggregator 以一种使识别性能瓶颈变得容易的方式呈现 XHProf 数据。通过可排序的表格和图形表示，开发人员可以快速了解应用程序的哪些部分消耗了最多时间和资源。

2.  **内存泄漏检测：** Buggregator 的详细分析视图帮助开发人员查明内存使用高峰，这使其成为识别和解决内存泄漏的宝贵工具。

#### 安装

首先，您需要安装 Xhprof 扩展。一种方法是使用 PECL 包。

```terminal
pear channel-update pear.php.net
pecl install xhprof
```

接下来，安装分析器包：

```terminal
composer require --dev spiral/profiler:^3.0
```

安装该包后，将该包的 bootloader 添加到您的应用程序中。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Profiler\ProfilerBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关 bootloader 的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Profiler\ProfilerBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关 bootloader 的更多信息。
:::

::::

#### 配置

使用以下环境变量配置分析器，以将数据发送到 [Buggregator](https://github.com/buggregator/spiral-app) 服务器：

```dotenv .env
PROFILER_ENDPOINT=http://127.0.0.1:8000/api/profiler/store
PROFILER_APP_NAME=My super app
```

#### 使用

有两种使用分析器的方法：

- 分析器作为拦截器
- 分析器作为中间件

#### 分析器作为拦截器

如果您想分析应用程序的某些支持使用拦截器的特定部分，则拦截器将非常有用。

-   [控制器](../http/interceptors.md),
-   [GRPC](../grpc/interceptors.md),
-   [队列作业](../queue/interceptors.md).
-   TCP
-   [事件](../advanced/events.md#interceptors).

> **查看更多**
> 在 [Framework — Interceptors](../framework/interceptors.md) 部分阅读有关拦截器的更多信息。

要将分析器用作拦截器，您只需注册 `Spiral\Profiler\ProfilerInterceptor` 类。

这是一个如何在 http 层中使用分析器作为拦截器的示例：

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        \Spiral\Profiler\ProfilerInterceptor::class
    ];
}
```

#### 分析器作为中间件

如果您想分析对应用程序的所有请求，中间件将非常有用。要将分析器用作中间件，您需要将其添加到您的路由器。

> **查看更多**
> 在 [HTTP — Routing](../http/routing.md) 部分阅读有关中间件的更多信息。

##### 全局中间件

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Profiler\ProfilerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            ProfilerMiddleware::class,  // <================
            // ...
        ];
    }
    
    // ...
}
```

##### 路由组中间件

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Profiler\ProfilerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                // ...
            ],
            'profiler' => [                  // <================
                ProfilerMiddleware::class,
                'middleware:web',
            ],
        ];
    }
    
    // ...
}
```

##### 路由中间件

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Router\Annotation\Route;

final class UserController
{
    #[Route(route: '/users', name: 'user.store', methods: ['POST'], middleware: \Spiral\Profiler\ProfilerMiddleware::class)]
    public function store(...): void 
    {
        // ...
    }
}
```

<hr />

## XDebug

在使用 xDebug 扩展时，调试 Spiral 应用程序与调试任何其他经典的 PHP 应用程序一样可行。

### IDE 配置

首先，您需要配置您的 IDE 以与 xDebug 一起工作。

> **查看更多**
> 在官方 [文档](https://roadrunner.dev/docs/php-debugging) 中阅读有关 IDE 配置的更多信息。

### 按需

仅在需要时启用 RoadRunner 和 xDebug 更方便。将以下环境变量添加到 `.rr.yaml` 中以正确配置 xDebug：

```yaml .rr.yaml
env:
  PHP_IDE_CONFIG: serverName=application.loc
  XDEBUG_CONFIG: remote_host=localhost max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
```

> **注意**
> 根据您的环境更改值。

要启用 xDebug，请使用 `-o`（覆盖标志）运行应用程序服务器，用于所需的服务：

```terminal
./rr serve -o "server.command=php -d zend_extension=xdebug app.php"
```

### 在 Docker 中

要在 docker 中更改 worker 配置，请对您的容器使用以下或类似的配置：

```yaml docker-compose.yaml
version: "2"
services:
  ...
  app:
    ...
    command:
      - /usr/local/bin/rr
      - serve
      - -o
      - server.command=php -d zend_extension=xdebug.so app.php
    environment:
      PHP_IDE_CONFIG: serverName=application.loc
      XDEBUG_CONFIG: remote_host=host.docker.internal max_nesting_level=250 remote_enable=1 remote_connect_back=0 var_display_max_depth=5 idekey='PHPSTORM'
```
