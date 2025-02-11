# Views — 入门

`spiral/views` 组件默认在 Web bundle 中可用。要在其他构建中使用它，请运行：

```terminal
composer require spiral/views
```

要激活该组件，请使用 Bootloader `\Spiral\Views\Bootloader\ViewsBootloader`：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Views\Bootloader\ViewsBootloader::class,
        // ...
    ];
}
```

关于 bootloaders 更多信息，请阅读[框架 — Bootloaders](../framework/bootloaders.md) 章节。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Views\Bootloader\ViewsBootloader::class,
    // ...
];
```

关于 bootloaders 更多信息，请阅读[框架 — Bootloaders](../framework/bootloaders.md) 章节。
:::

::::

该组件可以处理多种渲染引擎（**Twig**，**Stempler** 或 **native PHP** 模板），并在多个命名空间中存储视图文件。

## 配置

要更改默认组件配置，请创建并编辑文件 `app/config/views.php`：

```php app/config/views.php
use Spiral\Views\Engine\Native\NativeEngine;

return [
    'cache' => [
        'enabled' => !env('DEBUG', false),
        'directory' => directory('cache') . 'views'
    ],
    'namespaces' => [
        'default' => [
            directory('views')
        ]
    ],
    'dependencies' => [],
    'engines' => [
        NativeEngine::class
    ],
    'globalVariables' => [
        'some_var' => env('SOME_VALUE')
    ]
];
```

## 缓存

默认情况下，如果设置了 `DEBUG` 环境变量，视图缓存将被禁用。缓存文件存储在 `runtime/cache/views` 中。

## 清除

运行控制台命令 `views:reset`  来删除视图缓存：

```terminal
php app.php views:reset
```

## 预热

为了预热缓存，运行 `views:compile` 或 `configure`:

```terminal
php app.php views:compile
```

> **警告**
> 在负载下切换工作者池之前，务必预热视图缓存。
