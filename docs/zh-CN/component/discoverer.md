# 组件 — 发现器

`spiral-packages/discoverer` 包是 Spiral 框架中一个有用的工具。它增强了框架的功能，允许从应用程序核心之外的来源发现引导程序和词法分析器目录。此功能简化了在 Spiral 应用程序中管理和集成各种包的过程。

**特性**

1.  **自动发现引导程序：** 自动发现和注册来自已安装包的引导程序，显著简化了 Spiral 应用程序中的集成和设置过程。

2.  **Composer.json 集成：** 该包利用其他包的 `composer.json` 文件来定义引导程序和词法分析器目录。这种集成简化了配置过程，使开发人员更容易管理包设置。

3.  **自定义注册表支持：** 该包允许创建自定义引导程序和词法分析器注册表。这种灵活性使开发人员能够根据其特定需求定制发现过程，从而增强了 Spiral 应用程序的自定义性和可扩展性。

## 要求

确保您的服务器配置了以下 PHP 版本和扩展：

-   PHP 8.1+
-   Spiral 框架 3.10 或更高版本

## 安装

使用以下命令通过 Composer 安装该包：

```terminal
composer require spiral-packages/discoverer
```

安装后，您需要注册该包的引导程序。将 `DiscovererBootloader` 添加到您的系统的引导程序数组中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineSystemBootloaders(): array
{
    return [
        // ...
        \Spiral\Discoverer\DiscovererBootloader::class,
    ];
}
```

在 [框架 — 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的内容。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const SYSTEM = [
    // ...
    \Spiral\Discoverer\DiscovererBootloader::class,
];
```

在 [框架 — 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的内容。
:::

::::

就是这样。

## 注册表

发现器可以在不同类型的注册表中搜索引导程序或词法分析器目录：

### Composer.json 注册表

当 Spiral 应用程序开发人员安装一个附加包时，他们通常需要注册该包的引导程序。为了简化这个过程，包可以在其 `composer.json` 文件的 `extra` 部分定义它们的引导程序。

**像这样：**

```json vendor/spiral/dotenv/composer.json
{
  "extra": {
    "spiral": {
      "bootloaders": [
        "Spiral\\DotEnv\\Bootloader\\MonologBootloader"
      ],
      "directories": [
        "src/Entities"
      ]
    }
  }
}
```

在某些情况下，您可能希望禁用特定包的包发现。您可以通过在应用程序的 `composer.json` 文件的 `extra` 部分中列出包名称来执行此操作：

```json composer.json
{
  "extra": {
    "spiral": {
      "dont-discover": [
        "spiral/dotenv",
        "spiral-packages/bar"
      ]
    }
  }
}
```

安装后，发现器将自动注册该包的引导程序和目录。

### 自定义引导程序注册表

您可以通过实现 `Spiral\Discoverer\Bootloader\BootloaderRegistryInterface` 来创建引导程序的自定义源。

这是一个例子：

```php
use Spiral\Discoverer\Bootloader\BootloaderRegistryInterface;
use Spiral\Core\Container;
use Spiral\Files\FilesInterface;

final class JsonRegistry implements BootloaderRegistryInterface
{
    private array $bootloaders = [];

    public function __construct(private string $jsonPath) {}

    public function init(Container $container): void {
        $files = $container->get(FilesInterface::class);
        $data = json_decode($files->read($this->jsonPath), true);

        $this->bootloaders = $data['bootloaders'] ?? [];
    }

    public function getBootloaders(): array {
        return $this->bootloaders;
    }

    public function getIgnoredBootloaders(): array {
        return [];
    }
}
```

在 `discoverer.php` 配置文件中注册您的自定义注册表：

```php app/config/discoverer.php
<?php

use Spiral\Discoverer\Bootloader as BootloaderRegistry;
use Spiral\Discoverer\Tokenizer as TokenizerRegistry;

return [
    'registries' => [
        'bootloaders' => [
            BootloaderRegistry\ComposerRegistry::class,
            BootloaderRegistry\ConfigRegistry::class,
            JsonRegistry::class,
        ],
        'directories' => [
            TokenizerRegistry\ComposerRegistry::class,
        ],
    ],
];
```

### 自定义词法分析器注册表

类似地，您可以通过实现 `Spiral\Discoverer\Tokenizer\DirectoryRegistryInterface` 来创建词法分析器目录的自定义源。

这是一个例子：

```php
use Spiral\Discoverer\Tokenizer\DirectoryRegistryInterface;
use Spiral\Core\Container;
use Spiral\Files\FilesInterface;

final class JsonRegistry implements DirectoryRegistryInterface
{
    private array $directories = [];

    public function __construct(private string $jsonPath) {}

    public function init(Container $container): void {
        $files = $container->get(FilesInterface::class);
        $data = json_decode($files->read($this->jsonPath), true);

        $this->directories = $data['directories'] ?? [];
    }

    public function getDirectories(): array {
        return $this->directories;
    }
}
```

在 `discoverer.php` 配置文件中注册您的自定义注册表：

```php app/config/discoverer.php
<?php

use Spiral\Discoverer\Bootloader as BootloaderRegistry;
use Spiral\Discoverer\Tokenizer as TokenizerRegistry;

return [
    'registries' => [
        'bootloaders' => [
            BootloaderRegistry\ComposerRegistry::class,
            BootloaderRegistry\ConfigRegistry::class,
        ],
        'directories' => [
            TokenizerRegistry\ComposerRegistry::class,
            JsonRegistry::class,
        ],
    ],
];
```

如您所见，这些功能不仅节省了时间，而且引入了一定程度的灵活性和可扩展性，这在现代 Web 应用程序开发中是无价的。
