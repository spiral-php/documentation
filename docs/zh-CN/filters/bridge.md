# 过滤器 — 从 Spiral Framework 2.x 迁移

如果您是从 Spiral Framework 2.x 迁移过来，并且想继续使用旧的过滤器，在这种情况下，您可以使用 [spiral/filters-bridge](https://github.com/spiral/filters-bridge) 包。

`spiral/filters-bridge` 包提供了对请求验证、组合验证、错误消息映射和位置等的支持。

## 要求

确保您的服务器配置了以下 PHP 版本和扩展：

- PHP 8.1+
- Spiral framework 3.0+

## 安装

要安装这个包：

```terminal
composer require spiral/filters-bridge
```

> **注意**
> 该软件包将自动安装和配置 `spiral/validator` 包。

该软件包不需要任何配置，并且可以通过使用 Bootloader `Spiral\Filters\Bootloader\FiltersBootloader` 来激活：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Filters\Bootloader\FiltersBootloader::class,
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
    \Spiral\Filters\Bootloader\FiltersBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 Bootloader 的信息。
:::

::::

使用 `spiral/filters` 组件与 `spiral/filters-bridge` 包之间没有区别，因此你可以使用 [文档](https://spiral.dev/docs/filters-configuration/2.14/en#create-filter) 来了解如何迁移。
