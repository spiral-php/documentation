# HTTP 入门

`spiral/app` 软件包附带了一个预配置的 HTTP 组件。你需要在替代构建中启用它。

## 安装

要安装该扩展：

```terminal
composer require spiral/nyholm-bridge
```

通过添加两个 Bootloader 来激活组件：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        // 快速的 PSR-7 实现
        \Spiral\Nyholm\Bootloader\NyholmBootloader::class,
    
        // HTTP 核心
        \Spiral\Bootloader\Http\HttpBootloader::class,
    
        // PSR-15 handler
        \Spiral\Bootloader\Http\RouterBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 Bootloader 的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    // 快速的 PSR-7 实现
    \Spiral\Nyholm\Bootloader\NyholmBootloader::class,

    // HTTP 核心
    \Spiral\Bootloader\Http\HttpBootloader::class,

    // PSR-15 handler
    \Spiral\Bootloader\Http\RouterBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 Bootloader 的信息。
:::

::::

> **注意**
> 在 [cookbook/psr-15.md](../cookbook/psr-15.md) 中了解如何使用自定义 PSR-15 处理程序。

确保配置 [路由](../http/routing.md)。

## 配置

可以通过 `app/config/http.php` 文件配置 HTTP 扩展：

```php app/config/http.php
return [
    // 默认的 base path
    'basePath'   => '/',
    
    // 默认的 headers
    'headers'    => [
        'Content-Type' => 'text/html; charset=UTF-8'
    ],

    // 应用级别的 middleware
    'middleware' => [
        // middleware 类名
    ],
];
```

> **注意**
> 如果该文件不存在，将使用默认配置。
