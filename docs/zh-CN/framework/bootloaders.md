# 框架 — Bootloader

Spiral 在引导过程中使用 bootloader 类来处理应用程序配置。其中一项关键特性是 bootloader 仅在应用程序引导期间执行一次，确保向其中添加额外代码不会对运行时性能产生负面影响。

Bootloader 类在配置应用程序的各个方面发挥着至关重要的作用。

**以下是一些示例：**

- **容器配置：** Bootloader 可用于配置依赖注入容器。这包括注册服务和将接口绑定到它们各自的实现等任务。
- **应用程序配置：** Bootloader 负责设置各种应用程序级别的设置，包括环境、调试模式和错误处理。
- **数据库配置：** Bootloader 参与设置和配置应用程序的数据库连接。这包括指定数据库驱动程序、连接详细信息以及任何必要的迁移。
- **路由配置：** Bootloader 可用于设置应用程序的路由规则，例如定义哪些控制器应该处理哪些 URL。
- **服务初始化：** Bootloader 负责初始化应用程序所需的任何服务，例如缓存、日志记录和事件系统。

![Application Control Phases](https://user-images.githubusercontent.com/773481/180768689-c711e6f0-3523-4330-a496-f78088504b29.png)

## 简单的 Bootloader

要轻松创建一个简单的 bootloader，请使用 scaffolding 命令：

```terminal
php app.php create:bootloader GithubClient
```

> **注意**
> 更多关于 scaffolding 的内容请阅读 [基础知识 — Scaffolding](../basics/scaffolding.md#bootloader) 章节。

执行此命令后，将通过以下输出确认创建成功：

```output
Declaration of ' [32mGithubClientBootloader [39m' has been successfully written into ' [33mapp/src/Application/Bootloader/GithubClientBootloader.php [39m'.
```

现在您可以在 `app/src/Application/Bootloader` 目录中找到 `GithubClientBootloader` 类。

目前，新的 Bootloader 不会执行任何操作。稍后，我们将为其添加一些功能。

## 注册 bootloader

每个 bootloader 都必须在您的应用程序内核 `app/src/Application/Kernel.php` 中激活。

在 Spiral 中，您可以使用两种不同的方法在内核中注册 bootloader：常量和方法。

1.  **常量：** 内核提供了诸如 `Kernel::SYSTEM`、`Kernel::LOAD` 和 `Kernel::APP` 之类的常量。这些常量允许您指定应在应用程序初始化过程的不同阶段执行哪些 bootloader。通过使用这些常量，您可以实现清晰的关注点分离，并轻松理解哪些 bootloader 处理特定的任务。
2.  **方法：** 内核还提供了诸如 `defineBootloaders`、`defineAppBootloaders` 和 `defineSystemBootloaders` 之类的方法。这些方法允许更复杂的逻辑和对象、匿名类或其他高级用例的注册。使用这些方法，您可以更好地控制和灵活地控制应用程序的初始化过程。

> **警告**
> 您不能同时使用方法和常量。如果您选择使用方法，则通过常量进行的注册将不会生效。

:::: tabs

::: tab 使用方法

将类引用添加到您的 `App\Application\Kernel` 类的 `defineBootloaders` 或 `defineAppBootloaders` 方法中：

```php app/src/Application/Kernel.php
namespace App\Application;

use App\Application\Bootloader\RoutesBootloader;
use App\Application\Bootloader\LoggingBootloader;
use App\Application\Bootloader\MyBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // ...
           RoutesBootloader::class,
        ];
    }

    public function defineAppBootloaders(): array
    {
        return [
           LoggingBootloader::class,
           MyBootloader::class,
           
           // anonymous bootloader via object instance
           new class extends Bootloader {
               // ...
           },
       ];
    }
}
```

> **注意**
>  `defineAppBootloaders` 方法中的 Bootloader 始终在 `defineBootloaders` 方法中的 bootloader 之后加载。将特定于域的 bootloader 保留在其中。

:::

::: tab 使用常量

将类引用添加到您的 `App\Application\Kernel` 类的 `LOAD` 或 `APP` 常量中：

```php app/src/Application/Kernel.php
namespace App\Application;

use App\Application\Bootloader\RoutesBootloader;
use App\Application\Bootloader\LoggingBootloader;
use App\Application\Bootloader\MyBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    protected const LOAD = [
        // ...
        RoutesBootloader::class,
    ];

    protected const APP = [
        // ...
        LoggingBootloader::class,
        MyBootloader::class,
    ];
}
```

> **注意**
> `APP` 常量中的 bootloader 始终在 `LOAD` 常量中的 bootloader 之后加载。将特定于域的 bootloader 保留在其中。

:::

::::

### Bootloader 加载控制

还有一个特性可以让开发人员管理 bootloader 的设置和使用方式。此功能对于使应用程序适应各种环境（例如 HTTP、命令行界面或任何其他上下文）特别有利。它允许根据环境的特定需求选择性地激活或停用 bootloader，从而提高效率和性能。

有一个新的 DTO 类 `Spiral\Boot\Attribute\BootloadConfig`，它支持包含或排除 bootloader，传递将转发到 bootloader 的 init 和 boot 方法的参数，并根据环境变量动态调整 bootloader 加载。

要在 Kernel 中使用它，您应该使用 bootloader 的完整类名作为 bootloader 数组中的键，相应的值是 `BootloadConfig` 对象。

```php app/src/Application/Kernel.php
namespace App\Application;

use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Prototype\Bootloader\PrototypeBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // ...
            PrototypeBootloader::class => new BootloadConfig(allowEnv: ['APP_ENV' => ['local', 'dev']]),
            // ...
        ];
    }
}
```

在此示例中，我们指定仅当定义了环境变量 `APP_ENV` 且其值为 `local` 或 `dev` 时，才应加载 `PrototypeBootloader`。

您无需直接创建 `BootloadConfig` 对象，而是可以定义一个返回 `BootloadConfig` 对象的函数。此函数可以接受参数，这些参数可能从容器中获取。

```php app/src/Application/Kernel.php
namespace App\Application;

use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Environment\AppEnvironment;
use Spiral\Prototype\Bootloader\PrototypeBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // ...
            PrototypeBootloader::class => static fn (AppEnvironment $env) => new BootloadConfig(enabled: $env->isLocal()),
            // ...
        ];
    }
}
```

您还可以使用 `BootloadConfig` 类作为属性来控制 bootloader 的行为。此方法特别有用，因为它允许您直接在 bootloader 的类中设置配置，使其更简单、更容易理解。

**以下是您如何使用属性配置 bootloader 的简单示例：**

```php app/src/Application/Bootloader/SomeBootloader.php
use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Bootloader\Bootloader;

#[BootloadConfig(allowEnv: ['APP_ENV' => 'local'])]
final class SomeBootloader extends Bootloader
{
}
```

当您希望将配置与 bootloader 的代码保持紧密联系时，属性是一个绝佳的选择。这是一种更直观的设置 bootloader 的方式，尤其是在配置简单且不需要复杂逻辑的情况下。

#### 扩展以自定义前提条件

通过扩展 `BootloadConfig`，您可以创建自定义类，这些类封装了 bootloader 应该运行的特定条件。这种方法通过将配置详细信息抽象到这些自定义类中来简化 bootloader 的使用。

```php app/src/Application/Bootloader/TargetRRWorker.php
namespace App\Application\Bootloader;

use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Bootloader\Bootloader;

class TargetRRWorker extends BootloadConfig 
{
    public function __construct(array $modes)
    {
        parent::__construct(
            env: ['RR_MODE' => $modes],
        );
    }
}
```

现在您可以在您的 bootloader 中使用它

```php app/src/Application/Bootloader/SomeBootloader.php
use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Bootloader\Bootloader;

#[TargetRRWorker(modes: ['http', 'grpc'])]
final class SomeBootloader extends Bootloader
{
}
```

或者在 Kernel 中使用：

```php app/src/Application/Kernel.php
namespace App\Application;

use Spiral\Framework\Kernel;

class Kernel extends Kernel
{
    public function defineBootloaders(): array
    {
        return [
            HttpBootloader::class => new TargetRRWorker(['http']),
            RoutesBootloader::class => new TargetRRWorker(['http']),

            GrpcBootloader::class => new TargetRRWorker(['grpc']),

            TemporalBootloader::class => new TargetRRWorker(['temporal']),
            
            // Other bootloaders...
        ];
    }
}
```

扩展 `BootloadConfig` 的能力为自定义 bootloader 的行为开辟了无限的可能性。

## 可用方法

Bootloader 提供了两个方法 `init` 和 `boot`，它们在应用程序初始化时执行。

### Init 方法

此方法将**首先**运行。

在继续之前为配置文件设置默认值是个好主意。您可以使用特殊的 bootloader 方法根据需要修改配置文件。之后，您可以继续运行任何不需要读取配置文件并且不依赖于其他 bootloader 的 `init` 和 `boot` 方法中的代码执行的逻辑。

这也是添加初始化回调和配置容器绑定的好时机，只要它不需要访问应用程序配置即可。

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Config\ConfiguratorInterface;
use App\Service\Github\GithubConfig;

final class GithubClientBootloader extends Bootloader
{
    public function __construct(
        private readonly ConfiguratorInterface $config
    ) {
    }

    public function init(EnvironmentInterface $env): void 
    {
        $this->config->setDefaults(
            GithubConfig::CONFIG,
            [
                'access_token' => $env->get('GITHUB_ACCESS_TOKEN'),
                'secret' => $env->get('GITHUB_SECRET'),
            ]
        );
    }
}
```

> **注意**
> 1.  在 [配置](../start/configuration.md) 章节中了解更多关于 `Spiral\Boot\EnvironmentInterface` 的信息。
> 2.  在 [内核和环境](../framework/kernel.md) 章节中了解更多关于 `Spiral\Boot\AbstractKernel` 类（也称为“内核”）的信息。
> 3.  在 [配置对象](../framework/config.md) 章节中了解更多关于 `Spiral\Config\ConfiguratorInterface` 类的信息。

### Boot 方法

此方法将在所有 bootloader 中的 `init` 方法执行后运行。这样做的原因是您可能需要 bootloader 初始化结果才能继续进行。例如，已编译的配置文件。

请记住，它应该在所有这些 `init` 方法完成后运行。

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use App\Service\Github\GithubConfig;

final class GithubClientBootloader extends Bootloader
{
    // See code above ...
    
    public function boot(GithubConfig $config): void 
    {
        $token = $config->getAccessToken();
        // ...
    }
}
```

## 配置容器

Bootloader 通常用于设置容器，例如，如果我们想将多个实现链接到它们的接口或创建一些服务。我们可以使用 `init` 或 `boot` 方法来做到这一点，这使我们能够使用方法注入来请求我们需要的任何服务。

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use Spiral\Core\BinderInterface;
use App\Service\Github\GithubConfig;
use App\Service\Github\ClinetInterface;
use App\Service\Github\Clinet;

final class GithubClientBootloader extends Bootloader
{
    // See code above ...
    
    public function boot(BinderInterface $binder): void 
    {
        $binder->bindSingleton(
            ClinetInterface::class, 
            static fn (GithubConfig $config) => new Clinet(
                $config->getAccessToken(),
                $config->getSecret(),
            )
        );
    }
}
```

> **注意**
> 依赖注入 (DI) 容器将在需要创建 `MyService` 实例时调用作为参数提供给 `bindSingleton` 方法的闭包。当调用闭包时，DI 容器将自动解析并注入闭包所需的任何依赖项。
>
> 如果您想了解更多关于 DI 的信息，您可以查看文档的 [容器和工厂](../container/overview.md) 部分。它应该包含您需要的所有信息。

Bootloader 还提供了简化容器绑定定义的功能

:::: tabs

::: tab 使用方法

您可以使用 `defineBindings` 和 `defineSingletons` 方法以声明方式定义容器绑定。

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use Spiral\Core\BinderInterface;
use App\Service\Github\GithubConfig;
use App\Service\Github\ClientInterface;
use App\Service\Github\Client;

final class GithubClientBootloader extends Bootloader
{
    public function defineSingletons(): array
    {
        return [
            MyInterface::class => MyClass::class
        ];
    }

    public function defineSingletons(): array
    {
        return [
            ClientInterface::class => [self::class, 'createClient'],
            
            // or
            
            ClientInterface::class => static fn(GithubConfig $config) => new Client(
                $config->getAccessToken(),
                $config->getSecret(),
            );
        ];
    }

    // See code above ...
    
    public function createClient(GithubConfig $config): ClinetInterface 
    {
        return new Clinet(
            $config->getAccessToken(),
            $config->getSecret(),
        );
    }
}
```

:::

::: tab 使用常量

您可以使用 `BINDINGS` 和 `SINGLETONS` 常量以声明方式定义容器绑定。

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use Spiral\Core\BinderInterface;
use App\Service\Github\GithubConfig;
use App\Service\Github\ClientInterface;
use App\Service\Github\Client;

final class GithubClientBootloader extends Bootloader
{
    const BINDINGS = [
        MyInterface::class => MyClass::class
    ];
    
    const SINGLETONS = [
        ClientInterface::class => [self::class, 'createClient'],
    ];
    
    // See code above ...
    
    public function createClient(GithubConfig $config): ClinetInterface 
    {
        return new Clinet(
            $config->getAccessToken(),
            $config->getSecret(),
        );
    }
}
```

:::

::::

## 配置应用程序

Bootloader 的另一个常见用例是在应用程序启动之前配置框架。例如，我们可以为我们的应用程序或模块声明一个新的路由：

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Controller;
use Spiral\Router\Route;

final class RoutesBootloader extends Bootloader 
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'my-route',
            new Route('/<action>', new Controller(MyController::class))
        );
    }
}
```

> **注意**
> 您只能在引导阶段（又名通过另一个 bootloader）使用 bootloader 来配置您的组件。框架不允许您在组件初始化后更改任何配置值。

## 依赖于其他 Bootloader

依赖于其他 bootloader 在某些情况下可能非常有用。例如，如果您想确保在您的 bootloader 之前初始化某个 bootloader，您可以使用两种主要方法之一：将 bootloader 类注入到 init 或 boot 方法中作为参数，或者在您的 bootloader 类中使用 `Bootloader::DEPENDENCIES` 常量。

这可以成为管理应用程序初始化并确保在您需要时所有必要的资源和依赖项都可用的一种好方法。请记住，即使依赖 bootloader 被多个其他 bootloader 依赖，也只会初始化一次。

一些框架 bootloader 可以用作配置应用程序设置的简单方法。例如，我们可以使用 `Spiral\Bootloader\Http\HttpBootloader` 添加全局 PSR-15 中间件：

**定义依赖 bootloader 有两种方法：**

1.  将 bootloader 类注入到您 bootloader 类的 `init` 或 `boot` 方法中作为参数

**例如：**

```php app/src/Application/Bootloader/MyBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\HttpBootloader;
use App\Middleware\MyMiddleware;

class MyBootloader extends Bootloader 
{
    public function boot(HttpBootloader $http): void
    {
        $http->addMiddleware(MyMiddleware::class);
    }
}
```

2.  在您的 bootloader 类中使用 `Bootloader::DEPENDENCIES` 常量。当您不需要直接在 bootloader 类中访问依赖 bootloader 时，这可以成为定义依赖 bootloader 的一种便捷方式。

**例如：**

```php app/src/Application/Bootloader/MyBootloader.php
namespace App\Application\Bootloader;

class MyBootloader extends Bootloader 
{
    protected const DEPENDENCIES = [
        \Spiral\Bootloader\Http\HttpBootloader::class
    ];
    
    public function boot(): void
    {
        // ...
    }
}
```

在初始化依赖的 bootloader 之后，Spiral 将自动解析并初始化依赖的 bootloader。

这两种方法都允许您以声明方式定义依赖 bootloader，这可以使您更容易管理应用程序的初始化，并确保所有必要的资源和依赖项在需要时可用。

> **注意**
> 依赖 bootloader 仅初始化一次，即使有多个其他 bootloader 依赖于它们也是如此。

## 级联 bootloading

您可以使用 Bootloader 本身控制 bootload 过程，只需请求 `Spiral\Boot\BootloadManager`：

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\BootloadManager;
use Spiral\Bootloader\DebugBootloader;
use Spiral\Boot\EnvironmentInterface;

class AppBootloader extends Bootloader
{
    public function boot(BootloadManager $bootloadManager, EnvironmentInterface $env): void
    {
        if ($env->get('DEBUG')) {
            $bootloadManager->bootload([
                DebugBootloader::class
            ]);
        }
    }
}
```

<hr>

## 接下来是什么？

现在，通过阅读一些文章深入了解基础知识：

*   [HTTP — Interceptors](../http/interceptors.md)
*   [Scaffolding](../basics/scaffolding.md)
