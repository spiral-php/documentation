# 视图 — Twig 模板引擎

该框架提供了与 [Twig 模板](https://twig.symfony.com/)引擎的深度集成，包括对 IoC 范围、i18n 集成和缓存的访问。

## 安装和配置

要安装 Twig 桥接组件，请运行以下命令：

```terminal
composer require spiral/twig-bridge
```

通过将 `Spiral\Twig\Bootloader\TwigBootloader` 引导加载器添加到您的 `Kernel` 来激活该组件：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Views\Bootloader\ViewsBootloader::class,
        \Spiral\Bootloader\Views\TranslatedCacheBootloader::class, // 将本地化视图保存在单独的缓存文件中
        \Spiral\Twig\Bootloader\TwigBootloader::class,
        // ...
    ];
}
```

在 [框架 — 引导加载器](../framework/bootloaders.md) 部分阅读有关引导加载器的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Views\Bootloader\ViewsBootloader::class,
    \Spiral\Bootloader\Views\TranslatedCacheBootloader::class, // 将本地化视图保存在单独的缓存文件中
    \Spiral\Twig\Bootloader\TwigBootloader::class,
    // ...
];
```

在 [框架 — 引导加载器](../framework/bootloaders.md) 部分阅读有关引导加载器的更多信息。
:::

::::

您可以通过 `TwigBootloader` 的 `addExtension` 方法将任何自定义扩展添加到 Twig：

```php app/src/Application/Bootloader/TwigExtensionBootloader.php
class TwigExtensionBootloader extends Bootloader
{
    public function boot(TwigBootloader $twig): void
    {
        $twig->addExtension(MyExtension::class);
    
        // 自定义选项
        $twig->setOption('name', 'value');
    }
}
```

## 使用

您可以立即使用 twig 视图。在 `app/views` 目录中创建一个带有 `.twig` 扩展名的视图。

```twig app/views/filename.twig
Hello, {{ name }}!
```

您可以在控制器中不使用扩展名来使用此视图：

```php
public function index(): string
{
    return $this->views->render('filename', ['name' => 'User']);
}
```

> **注意**
> 您可以自由使用 twig 的 `include` 和 `extends` 指令。

要从 IoC 范围访问值：

```twig app/views/filename.twig
Hello, {{ name }}!

{{ get("Spiral\\Http\\Request\\InputManager").attribute('csrfToken') }}
```

## 调试

在 Twig 中，您可以使用 `dump` 函数来显示有关变量的信息，包括其类型和值。这对于在您的模板中进行调试非常有用。

要启用该功能，请创建一个新的引导加载器：`TwigDebugBootloader`：

```php app/src/Application/Bootloader/TwigDebugBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Twig\Bootloader\TwigBootloader;
use Twig\Extension\DebugExtension;

final class TwigDebugBootloader extends Bootloader
{
    public function boot(TwigBootloader $twig)
    {
        $twig->addExtension(new DebugExtension());
        $twig->setOption('debug', true);
    }
}
```

通过将 `TwigDebugBootloader` 引导加载器添加到您的 `Kernel` 来激活该组件：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\TwigDebugBootloader::class,
        // ...
    ];
}
```

在 [框架 — 引导加载器](../framework/bootloaders.md) 部分阅读有关引导加载器的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\TwigDebugBootloader::class,
    // ...
];
```

在 [框架 — 引导加载器](../framework/bootloaders.md) 部分阅读有关引导加载器的更多信息。
:::

::::

之后，您可以在您的模板中使用 `{{ dump() }}` 函数。

```twig
<pre>
    {{ dump(cats) }}
</pr>
```

在这种情况下，`cats` 是我们想要调试的变量。

您可以通过将它们作为参数传递来分配更多变量：

```twig
{{ dump(cats, dogs, birds) }}
```

如果我们没有向该函数传递任何值，则将转储来自当前上下文的所有变量。

> **查看更多**
> 有关更多信息，请访问官方 [twig 文档](https://twig.symfony.com/doc/3.x/functions/dump.html)。
