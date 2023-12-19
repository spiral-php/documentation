# Views — Twig templating

The framework provides deep integration with the [Twig Template](https://twig.symfony.com/) engine including access to 
IoC scopes, i18n integration, and caching.

## Installation and Configuration

To install Twig bridge component, run the following command:

```terminal
composer require spiral/twig-bridge
```

Activate the component by adding `Spiral\Twig\Bootloader\TwigBootloader` bootloader to your `Kernel`:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Views\Bootloader\ViewsBootloader::class,
        \Spiral\Bootloader\Views\TranslatedCacheBootloader::class, // keep localized views in separate cache files
        \Spiral\Twig\Bootloader\TwigBootloader::class,
        // ...
    ];
}
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Views\Bootloader\ViewsBootloader::class,
    \Spiral\Bootloader\Views\TranslatedCacheBootloader::class, // keep localized views in separate cache files
    \Spiral\Twig\Bootloader\TwigBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

You can add any custom extension to Twig via the `addExtension` method of `TwigBootloader`:

```php app/src/Application/Bootloader/TwigExtensionBootloader.php
class TwigExtensionBootloader extends Bootloader
{
    public function boot(TwigBootloader $twig): void
    {
        $twig->addExtension(MyExtension::class);
    
        // custom options
        $twig->setOption('name', 'value');
    }
}
```

## Usage

You can use twig views immediately. Create a view with `.twig` extension in the `app/views` directory.

```twig app/views/filename.twig
Hello, {{ name }}!
```

You can use this view without an extension in your controllers:

```php
public function index(): string
{
    return $this->views->render('filename', ['name' => 'User']);
}
```

> **Note**
> You can freely use twig's `include` and `extends` directives.

To access the value from the IoC scope:

```twig app/views/filename.twig
Hello, {{ name }}!

{{ get("Spiral\\Http\\Request\\InputManager").attribute('csrfToken') }}
```

## Debug

In Twig you can use the `dump` function to display information about a variable, including its type and value. It's 
useful for debugging purposes in your templates.

To enable the function create a new Bootloader: `TwigDebugBootloader`:

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

Activate the component by adding `TwigDebugBootloader` bootloader to your `Kernel`:

:::: tabs

::: tab Using method

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

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\TwigDebugBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

Afterwards you can use the `{{ dump() }}` function in your template.

```twig
<pre>
    {{ dump(cats) }}
</pr>
```

`cats` in this case is the variable we would like to debug.

You can assign more variables by passing them as arguments:

```twig
{{ dump(cats, dogs, birds) }}
```

If we don't pass any value to the function, all variables from the current context will be dumped.

> **See more**
> For more information, visit the official [twig documentation](https://twig.symfony.com/doc/3.x/functions/dump.html).
