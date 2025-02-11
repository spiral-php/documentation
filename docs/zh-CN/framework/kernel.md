# 框架 — 内核和环境

Spiral 使用一个包含一组应用程序特定服务的内核对象。 与 Symfony 不同，Spiral 只需要一个内核来处理所有调度方法，例如 HTTP、队列、GRPC、控制台等。内核会根据连接的 [调度器](../framework/dispatcher.md) 自动选择合适的调度方法。

> **注意**
> 基础内核实现位于 `spiral/boot` 仓库中。

## 内核职责

`Spiral\Boot\AbstractKernel` 类负责应用程序的以下方面：

- 通过一组应用程序特定的启动加载器初始化容器
- 初始化启动加载器
- 初始化环境和目录结构
- 初始化异常处理程序（如果需要）
- 选择合适的调度器

要创建一个应用程序内核，必须继承 `Spiral\Boot\AbstractKernel` 类。以下代码片段展示了一个例子：

```php app/src/Application/MyApp.php
namespace App\Application;

use Spiral\Boot\AbstractKernel;
use Spiral\Boot\Exception\BootException;

final class MyApp extends AbstractKernel
{
    protected const LOAD = [
        // 要初始化的启动加载器
    ];

    protected function bootstrap(): void
    {
        // 自定义初始化代码
        // 在所有启动加载器加载后调用
    }

    protected function mapDirectories(array $directories): array
    {
        if (!isset($directories['root'])) {
            throw new BootException('Missing required directory `root`');
        }

        if (!isset($directories['app'])) {
            $directories['app'] = $directories['root'] . '/app/';
        }

        return \array_merge(
            [
                // public 根目录
                'public'    => $directories['root'] . '/public/',

                // vendor 库
                'vendor'    => $directories['root'] . '/vendor/',

                // 数据目录
                'runtime'   => $directories['root'] . '/runtime/',
                'cache'     => $directories['root'] . '/runtime/cache/',

                // 应用程序目录
                'config'    => $directories['app'] . '/config/',
                'resources' => $directories['app'] . '/resources/',
            ],
            $directories
        );
    }
}
```

> **注意**
> `Spiral\Framework\Kernel` 定义了默认的目录映射。

## 内核初始化

要初始化内核，应调用静态方法 `create`。以下代码片段展示了一个例子：

```php app.php
$myapp = MyApp::create(
    directories: [
        'root' => __DIR__,
    ],
    handleErrors: false // 不要挂载错误处理程序
);

$myapp->run(environment: null); // 使用默认环境

\dump($myapp->get(\Spiral\Boot\DirectoriesInterface::class)->getAll());
```

> **注意**
> 在初始化期间，`MyApp` 将在容器中作为单例绑定到 `Spiral\Boot\KernelInterface`。

### 回调

`Spiral\Boot\AbstractKernel` 类提供了几个回调，这些回调在应用程序初始化的不同阶段执行。这些回调包括 `running`、`booting`、`booted` 和 `bootstrapped`。 扩展自 `AbstractKernel` 的 `Spiral\Framework\Kernel` 类添加了额外的回调，`appBooting` 和 `appBooted`。这允许开发人员在应用程序初始化过程的特定阶段执行自定义操作。

> **注意**
> 在应用程序包中，默认的 `App\Application\Kernel` 类继承了 `Spiral\Framework\Kernel` 类，并使用这些回调。

#### Running（运行中）

`running` 回调是应用程序初始化过程中第一个执行的回调。当调用 `run` 方法时执行，紧接着在应用程序容器中绑定 `EnvironmentInterface` 之后执行。

以下是 `running` 回调的一个例子：

```php app.php
$app = MyApp::create(directories: ['root' => __DIR__]);

$app->running(static function (): void {
    // 做一些事情
});

$app->run();
```

> **注意**
> 回调可以被多次调用以注册多个回调，它们将按照注册的顺序被调用。

#### Booting（启动中）

`booting` 回调在 `LOAD` 部分中的所有框架启动加载器启动之前执行。

有两种方法可以为 `booting` 阶段注册回调：

:::: tabs

::: tab Kernel initialization（内核初始化）

可以在创建应用程序实例后调用 `booting` 方法。

```php app.php
$app = MyApp::create(
    directories: ['root' => __DIR__]
);

$app->booting(function () {
    // ...
});

$app->run();
```

:::

::: tab Kernel bootloaders（内核启动加载器）

启动加载器的 `init` 方法也可以用于注册一个 booting 回调。此方法在回调触发之前执行。

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\KernelInterface;

class AppBootloader extends Bootloader
{
    public function init(KernelInterface $app): void
    {
        $app->booting(function () {
            // ...
        });
    }
}
```

:::

::::

#### Booted（已启动）

`booted` 回调在 `LOAD` 部分中的所有框架启动加载器完成其初始化过程后执行。

```php
$app->booted(function () {
    // ...
});
```

#### AppBooting（应用启动中）

`appBooting` 回调在 `APP` 部分中的所有应用程序启动加载器启动之前执行。

```php
$app->appBooting(function () {
    // ...
});
```

#### AppBooted（应用已启动）

`appBooted` 回调在 `APP` 部分中的所有应用程序启动加载器完成其初始化过程后执行。

```php
$app->appBooted(function () {
    // ...
});
```

## Environment（环境）

Spiral 通过 `Spiral\DotEnv\Bootloader\DotenvBootloader` 类与 [Dotenv](https://github.com/vlucas/phpdotenv) 集成。此启动加载器负责从 `.env` 文件加载环境变量，并使它们可供应用程序使用。

### 环境变量

`Spiral\Boot\EnvironmentInterface` 用于访问环境变量列表（ENV 变量）。默认情况下，框架依赖于系统级别的环境变量。但是，可以通过在初始化内核时向 `run` 方法传递自定义的 `Spiral\Boot\Environment` 对象来重新定义这些值。

> **另请参阅**
> 在 [入门 — 配置](../start/configuration.md) 部分阅读更多关于应用程序环境的信息。

以下代码片段展示了一个例子：

```php app.php
use \Spiral\Boot\Environment;

// 创建一个应用程序实例 ...

$app->run(new Environment(['DEBUG' => true]));

\dump($app->get(\Spiral\Boot\EnvironmentInterface::class)->getAll());
```

> **注意**
> 这种方法可用于为测试目的启动应用程序。

### .env 文件位置

默认情况下，启动加载器在项目的根目录中查找 `.env` 文件，但是您可以通过在运行内核时定义 `DOTENV_PATH` 环境变量来更改其位置：

```php app.php
use Spiral\Boot\Environment;

$app = App\Application\Kernel::create(...);

$app->run(new Environment(['DOTENV_PATH' => __DIR__ . '/.env.production']));
```

> **注意**
> 此外，您还可以创建您自己的 `DotenvBootloader` 类的实现。这允许您自定义加载环境变量的行为，例如更改搜索 .env 文件的位置，或添加附加功能。这在默认启动加载器无法满足应用程序的特定要求的情况下非常有用。

### 覆盖变量

默认情况下，Spiral 在从 `.env` 文件加载新环境变量时不会覆盖先前设置的环境变量。但是，可以通过在初始化 `Environment` 类时将 `overwrite` 参数设置为 `true` 来更改此行为。

```php app.php
use Spiral\Boot\Environment;

$app = App\Application\Kernel::create(...);

$app->run(new Environment([
    'APP_ENV' => 'production'
], overwrite: true));
```

## 事件

| 事件                                | 描述                                                                                                         |
|--------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Spiral\Boot\Event\Bootstrapped       | 该事件将在来自 `SYSTEM`、`LOAD` 和 `APP` 部分的所有启动加载器初始化 `后` 触发。                                  |
| Spiral\Boot\Event\Serving            | 该事件将在为当前环境处理传入请求查找调度器 `之前` 触发。                                                            |
| Spiral\Boot\Event\DispatcherFound    | 当找到用于处理当前环境中传入请求的调度器时，将触发该事件。                                                          |
| Spiral\Boot\Event\DispatcherNotFound | 当未找到应用程序调度器时，将触发该事件。                                                                             |
| Spiral\Boot\Event\Finalizing         | 当执行最终器`之前`运行最终器时，将触发该事件。                                                                  |

> **注意**
> 要了解更多关于调度事件的信息，请参阅我们文档中的 [事件](../advanced/events.md) 部分。
