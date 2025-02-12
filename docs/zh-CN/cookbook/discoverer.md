# 烹饪书 — Bootloader 发现器

The `BootloadersDiscoverer` class is responsible for loading all bootloaders in the application. Bootloaders are classes that implement the `Spiral\Bootloader\BootloaderInterface` interface. They are used to register services and configure the application during the bootstrapping process.

`BootloadersDiscoverer` 类负责加载应用程序中的所有 bootloader。 Bootloader 是实现了 `Spiral\Bootloader\BootloaderInterface` 接口的类。 它们用于在引导过程中注册服务和配置应用程序。

The `BootloadersDiscoverer` uses the `Spiral\Bootloader\BootloaderRegistryInterface` to find all bootloaders. By default, it will search in the following directories:

`BootloadersDiscoverer` 使用 `Spiral\Bootloader\BootloaderRegistryInterface` 来查找所有 bootloader。 默认情况下，它将在以下目录中搜索：

*   `app/Bootloader`
*   `config/bootloaders` (if enabled)

The discoverer will also look for all bootloaders declared in the `bootloaders` configuration. You can define a custom array of directories or classes:

发现器还将查找在 `bootloaders` 配置中声明的所有 bootloader。 您可以定义自定义的目录或类数组：

```php
<?php

declare(strict_types=1);

use Spiral\Bootloader\BootloaderInterface;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Core\Container\Autowire;
use Spiral\Core\FactoryInterface;
use Spiral\DotEnv\Bootloader\DotEnvBootloader;
use Spiral\Environment\EnvBootloader;
use Spiral\Files\FilesBootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;
use Spiral\RoadRunnerBridge\Bootloader\RoadRunnerBootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;
use Spiral\Tokenizer\Bootloader\TokenizerBootloader;

return [
    'directories' => [
        'app/Bootloader' // Custom location for your bootloaders
    ],
    'classes' => [
        //  String
        EnvBootloader::class,
        //  Autowire with dependencies
        new Autowire(MonologBootloader::class),
        //  Array with arguments
        [DotEnvBootloader::class, ['path' => directory('root').'.env']],
        //  Callable that returns BootloaderInterface instance
        static function (FactoryInterface $factory, ConfiguratorInterface $configurator): BootloaderInterface {
            return $factory->make(RoadRunnerBootloader::class);
        },
        // ... your bootloaders
    ],
];
```

### Discovering with Namespace

You can define a custom namespace to be used for bootloaders:

### 使用命名空间发现

您可以定义一个用于 bootloader 的自定义命名空间：

```php
<?php

declare(strict_types=1);

use Spiral\Bootloader\BootloaderInterface;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Core\Container\Autowire;
use Spiral\Core\FactoryInterface;
use Spiral\DotEnv\Bootloader\DotEnvBootloader;
use Spiral\Environment\EnvBootloader;
use Spiral\Files\FilesBootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;
use Spiral\RoadRunnerBridge\Bootloader\RoadRunnerBootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;
use Spiral\Tokenizer\Bootloader\TokenizerBootloader;

return [
    'namespaces' => [
        'MyBootloaders' => 'app/Bootloader',
    ],
    'classes' => [
        //  String
        EnvBootloader::class,
        //  Autowire with dependencies
        new Autowire(MonologBootloader::class),
        //  Array with arguments
        [DotEnvBootloader::class, ['path' => directory('root').'.env']],
        //  Callable that returns BootloaderInterface instance
        static function (FactoryInterface $factory, ConfiguratorInterface $configurator): BootloaderInterface {
            return $factory->make(RoadRunnerBootloader::class);
        },
        // ... your bootloaders
    ],
];
```

In this example, the `BootloadersDiscoverer` will look for all classes that are in the `MyBootloaders` namespace and are located in the `app/Bootloader` directory and all bootloaders from the `classes` option.

在这个例子中，`BootloadersDiscoverer` 将查找位于 `MyBootloaders` 命名空间下，且位于 `app/Bootloader` 目录中的所有类，以及 `classes` 选项中的所有 bootloader。

### Disable Directory Discovering

You can disable directory discovery:

### 禁用目录发现

您可以禁用目录发现：

```php
<?php

declare(strict_types=1);

use Spiral\Bootloader\BootloaderInterface;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Core\Container\Autowire;
use Spiral\Core\FactoryInterface;
use Spiral\DotEnv\Bootloader\DotEnvBootloader;
use Spiral\Environment\EnvBootloader;
use Spiral\Files\FilesBootloader;
use Spiral\Monolog\Bootloader\MonologBootloader;
use Spiral\RoadRunnerBridge\Bootloader\RoadRunnerBootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;
use Spiral\Tokenizer\Bootloader\TokenizerBootloader;

return [
    'directories' => false,
    'classes' => [
        //  String
        EnvBootloader::class,
        //  Autowire with dependencies
        new Autowire(MonologBootloader::class),
        //  Array with arguments
        [DotEnvBootloader::class, ['path' => directory('root').'.env']],
        //  Callable that returns BootloaderInterface instance
        static function (FactoryInterface $factory, ConfiguratorInterface $configurator): BootloaderInterface {
            return $factory->make(RoadRunnerBootloader::class);
        },
        // ... your bootloaders
    ],
];
```

In this case, the `BootloadersDiscoverer` will only load bootloaders from the `classes` option.

在这种情况下，`BootloadersDiscoverer` 将仅从 `classes` 选项加载 bootloader。
