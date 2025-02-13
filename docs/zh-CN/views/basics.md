# 视图 — 基础

Spiral 提供了 `spiral/views` 组件，它充当了处理视图的抽象层。它提供了一组用于注册模板引擎并使用它们渲染模板的接口。

## 视图

`Spiral\Views\ViewsInterface` 接口提供了渲染模板、编译视图的缓存版本、重置视图缓存以及获取与给定路径关联的视图实例的方法。

> **注意**
> 该组件也可以通过原型化属性 `views` 访问。 更多关于原型化 trait 的信息，请阅读 [基础 — 原型化](../basics/prototype.md) 章节。

### 渲染

要在控制器或其他服务中渲染视图，只需调用 `render` 方法即可。视图名称不需要包含扩展名或命名空间（将使用默认值）。

:::: tabs

::: tab ViewsInterface

你可以使用依赖注入将 `Spiral\Views\ViewsInterface` 接口注入到你的控制器中。

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Views\ViewsInterface;

class HomeController
{
    public function __construct(
      private readonly ViewsInterface $views
    ) {
    }
    
    public function index(): string
    {
        return $this->views->render('home');
    }
}
```

:::

::: tab Prototype

你可以使用原型化 trait 来访问 `views` 属性以加快开发速度。

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;
    
    public function index(): string
    {
        return $this->views->render('home');
    }
}
```

:::

::::

要使用传递的数据渲染视图，请使用第二个数组参数：

```php app/src/Endpoint/Web/HomeController.php
public function index(): string
{
    return $this->views->render('home', [
        'key' => 'value'
    ]);
}
```

### 获取视图实例

在某些情况下，将视图缓存在无状态组件中可能更有效率。要获取视图的实例，你可以使用 `get` 方法。

```php app/src/Endpoint/Web/HomeController.php
use Spiral\Views\ViewsInterface;

public function index(): string
{
    /** @var \Spiral\Views\ViewInterface $view */
    $view = $this->views->get('profile-card');
    
    $card1 = $view->render('name' => 'John']);
    $card2 = $view->render(['name' => 'Jane']);
    
    return "<html><body>{$card1} {$card2}</body></html>";
}
```

### 编译缓存

`compile` 方法可用于编译视图的一个或多个缓存版本。此方法将视图路径作为参数，并编译视图的缓存版本。

```php app/src/Endpoint/Console/CompileViewsCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Command;
use Spiral\Views\LoaderInterface;
use Spiral\Views\ViewsInterface;

class WarmupViewsCommand extends Command
{
    protected const NAME = 'views:compile';
    protected const DESCRIPTION = 'Compile all views in default namespace';

    public function __invoke(LoaderInterface $loader, ViewsInterface $views): void
    {
        $templates = $loader->withExtension('dark.php')->list();
        // or
        $templates = $loader->withExtension('twig')->list(namespace: 'my-package');

        foreach ($templates as $template) {
            $views->compile($template);
        }
    }
}
```

### 重置视图缓存

`reset` 方法可用于重置视图的缓存。此方法将视图路径作为参数，并重置视图的缓存。

```php
use Spiral\Views\ViewsInterface;

$views->reset('home');
```

## 视图资源加载器

`Spiral\Views\LoaderInterface` 负责加载视图的资源。 视图加载器可用于检查视图是否存在、加载其资源以及列出所有可用的视图。

> **注意**
> 默认情况下，LoaderInterface 由 `Spiral\Views\ViewLoader` 类实现，该类提供基于文件的视图加载。

为了使用在 `LoaderInterface` 中定义的方法，你首先需要为加载器指定一个扩展名。 这可以通过调用 `withExtension` 方法并传入所需的扩展名来完成。

> **警告**
> `withExtension` 方法是不可变的。 它将返回一个新的 `LoaderInterface` 实例。

```php
use Spiral\Views\LoaderInterface;

public function index(LoaderInterface $loader): string
{
    $loader = $loader->withExtension('dark.php');
    
    // ...
}
```

### 检查视图是否存在

`exists` 方法可用于检查视图是否可用于渲染。

```php
if (!$loader->exists('home')) {
    throw new \RuntimeException('View not found');
}

// ...
```

你也可以检查视图是否存在于特定的命名空间中。

```php
$loader->exists('my-package:home')
````

### 加载视图资源

`load` 方法可用于检索视图的资源。它将返回视图的资源作为 `Spiral\Views\ViewSource` 实例。

```php
/** @var \Spiral\Views\ViewSource $source */
$source = $loader->load('home');

$source->getNamespace(); // default
$source->getName(); // home
$source->getFilename(); // /app/views/home.dark.php
$source->getCode(); // <html>...</html>
```

在某些情况下，你可能希望替换视图的源代码。为此，你可以使用 `withCode` 方法。

```php
/** @var \Spiral\Views\ViewSource $source */
$source = $loader->load('home');

$source = $source->withCode('<div>...</div>');
$source->getCode(); // <div>...</div>
```

> **警告**
> `withCode` 方法是不可变的。 它将返回一个新的 `ViewSource` 实例。

### 列出可用视图

`list` 方法可用于列出所有可用的视图。它将返回一个视图路径的数组。

```php
$views = $loader->list();
```

你也可以列出特定命名空间中所有可用的视图。

```php
$views = $loader->list(namespace: 'my-package');
```

### 自定义视图加载器

你可以通过实现 `Spiral\Views\LoaderInterface` 接口来创建你自己的加载器。

这是一个从数据库加载视图的加载器的示例。

> **危险**
> 这只是一个展示如何实现自定义加载器的例子。它未经测试，不应使用。

```php app/src/Integration/Database/DatabaseLoader.php
namespace App\Integration\Database;

use Spiral\Views\LoaderInterface;
use Spiral\Views\Loader\PathParser;
use Spiral\Views\ViewSource;

final class DatabaseLoader implements LoaderInterface
{
    private ?PathParser $parser = null;
    
    public function __construct(
        private readonly DatabaseInterface $database,
        private readonly string $defaultNamespace = self::DEFAULT_NAMESPACE,
    ) {}
    
    public function withExtension(string $extension): LoaderInterface
    {
        $loader = clone $this;
        $loader->parser = new PathParser($this->defaultNamespace, $extension);

        return $loader;
    }
    
    public function getExtension(): ?string
    {
        return $this->parser?->getExtension();
    }
    
    public function exists(string $path, string &$filename = null, ViewPath &$parsed = null): bool
    {
        if ($this->parser === null) {
            throw new LoaderException(
                'Unable to locate view source, no extension has been associated.'
            );
        }
        
        $parsed = $this->parser->parse($path);
        if ($parsed === null) {
            return false;
        }
        
        if (!isset($this->namespaces[$parsed->getNamespace()])) {
            return false;
        }
        
        // Iterate over all registered namespaces and check if the view exists in 
        // any of them.
        foreach ((array)$this->namespaces[$parsed->getNamespace()] as $namespace) {
            $isExists = $this->getTemplate($namespace, $parsed->getBasename()) !== null;
            if ($isExists) {
                $filename = $parsed->getBasename();
                return true;
            }
        }
        
        return false;
    }
    
    public function load(string $path): ViewSource
    {
        if (!$this->exists($path, $filename, $parsed)) {
            throw new LoaderException(
                \sprintf('Unable to load view `%s`, file does not exist.', $path)
            );
        }
        
        // View variable contains an array of record from the database with the 
        // following structure:
        // ['html' => '<div>...</div>', 'path' => '...', 'namespace' => '...']
        $view = $this->getTemplate($parsed->getNamespace(), $parsed->getBasename());
        
        return (new ViewSource(
            $filename,
            $parsed->getNamespace(),
            $parsed->getName()
        ))->withCode($view['html']);
    }
    
    public function list(string $namespace = null): array
    {
        $views = [];
        foreach ($this->namespaces as $namespace) {
            $templates = $this->database->select()
                ->from('views')
                ->where('namespace', $namespace)
                ->fetchAll();
            
            foreach ($templates as $template) {
                $views[] = $namespace . ':' . $template['path'];
            }
        }
        
        return $views;
    }
    
    private function getTemplate(string $namespace, string $path): ?array
    {
        return $this->database->select()
            ->from('views')
            ->where('namespace', $namespace)
            ->where('path', $path)
            ->fetchOne();
    }
}
```

要使用此加载器，你需要替换容器中的默认加载器。

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Views\LoaderInterface;
use App\Integration\Database\DatabaseLoader;

class AppBootloader extends Bootloader 
{
    protected const SINGLETONS = [
        LoaderInterface::class => [self::class, 'initLoader'],
    ];
    
    protected function initLoader(DatabaseInterface $database): LoaderInterface
    {
        return new DatabaseLoader($database);
    }
}
```

## 全局变量

你可以添加将在所有视图中可用的全局变量。

有几种方法可以添加全局变量。

:::: tabs

::: tab Config
你可以使用配置文件 `app/config/views.php`。

```php app/config/views.php
return [
    // ...
    'globalVariables' => [
        'some_var' => env('SOME_VALUE'),
        'other_var' => 'other_value'
    ]
];
```

:::

::: tab GlobalVariablesInterface
你可以使用 `Spiral\Views\GlobalVariablesInterface`。

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Views\GlobalVariablesInterface;
use Spiral\Boot\EnvironmentInterface;

class AppBootloader extends Bootloader 
{
    public function boot(GlobalVariablesInterface $vars, EnvironmentInterface $env): void
    {
         $vars->set('some_var', $env->get('SOME_VALUE'));
         $vars->set('other_var', 'other_value');
    }
}
```

:::

::::

之后，你可以在任何视图模板中使用这些变量，包括继承的模板。

```php
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <?=$some_var?>

    <?=$other_var?>
</body>
</html>
```

> **了解更多**
> 例如，你可以使用全局变量将 CSRF token 传递给视图。
> 更多细节请参见 [HTTP — CSRF 保护](../http/csrf.md#注册视图全局变量)。

## 命名空间

在视图组件中，命名空间用于组织和分类视图。视图可以根据其用途或相关功能分隔到不同的命名空间中。这使得管理视图、避免命名冲突和重用通用模板变得更加容易。

### 注册命名空间

要注册命名空间，你可以使用 `ViewsBootloader` 类的 `addDirectory` 方法。此方法接受两个参数：第一个是命名空间名称，第二个是与该命名空间关联的目录路径。

例如，如果你在应用程序的 `my-package` 目录下有一个名为 views 的目录，你可以使用 `my-package` 命名空间注册它，如下所示：

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Views\Bootloader\ViewsBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ViewsBootloader $views): void
    {
        $views->addDirectory('my-package', directory('root') . 'my-package/views');
    }
}
```

### 访问命名空间

注册此命名空间后，你现在可以使用 `my-package` 命名空间引用 views 目录中的视图。 例如，如果你的 `my-package/views` 目录中有一个名为 `home.dark.php` 的视图文件，你可以按如下方式渲染它：

```php
$views->render('my-package:home');
```

## 模板引擎

### 可用

默认情况下，视图组件仅提供一个模板引擎 - [纯 PHP 模板](./templates.md)。但是，也有其他可用的模板引擎：

- [Stempler](./stempler.md)
- [Twig](./twig.md)

### 自定义模板引擎

该组件提供了注册自定义模板引擎的功能。要使用自定义模板引擎，你需要实现 `Spiral\Views\EngineInterface`。

以下是 `EngineInterface` 方法的简要说明：

#### 加载器

`withLoader` 方法用于为引擎设置和配置视图加载器。你可以在这里定义引擎将使用的视图文件的扩展名。

```php app/src/Integration/Views/BladeEngine.php
use Spiral\Views\EngineInterface;
use Spiral\Views\LoaderInterface;

final class BladeEngine implements EngineInterface
{
    private ?LoaderInterface $loader = null;
    
    public function withLoader(LoaderInterface $loader): EngineInterface
    {
        $engine = clone $this;
        $engine->loader = $loader->withExtension('blade.php');
    
        return $engine;
    }
    
    public function getLoader(): LoaderInterface
    {
        return $this->loader;
    }
}
```

#### 编译和重置缓存

`compile` 和 `reset` 方法编译（和重置缓存）给定上下文中的给定视图路径。 每次需要重新编译视图时都会调用这些方法。

```php app/src/Integration/Views/BladeEngine.php
use Spiral\Views\ContextInterface;
use Illuminate\View\Compilers\CompilerEngine;

private readonly CompilerEngine $blade;

public function compile(string $path, ContextInterface $context): mixed
{
    $filepath = $this->getLoader()->load($path)->getFilename();
    
    // Run the compilation process for the given view path
    $this->blade->getCompiler()->compile($filepath);
}

public function reset(string $path, ContextInterface $context): void
{
    $filepath = $this->getLoader()->load($path)->getFilename();
    
    // Reset the cache for the given view path
    \unlink($this->blade->getCompiler()->getCompiledPath($filepath));
}
```

#### 获取视图实例

`get` 方法返回与视图路径关联的视图类的实例。

```php app/src/Integration/Views/BladeEngine.php
use Spiral\Views\ViewInterface;
use Illuminate\View\Engines\CompilerEngine;

private readonly CompilerEngine $blade;

public function get(string $path, ContextInterface $context): ViewInterface
{
    $filepath = $this->getLoader()->load($path)->getFilename();
    
    return new BladeView($this->blade, $filepath);
}
```

```php app/src/Integration/Views/BladeView.php
use Spiral\Views\ViewInterface;
use Illuminate\View\Engines\CompilerEngine;

class BladeView implements ViewInterface
{
    public function __construct(
        private readonly CompilerEngine $blade, 
        private readonly string $filepath
    ) { 
    }
    
    public function render(array $data = []): string
    {
        return $this->blade->get($this->filepath, $data);
    }
}
```

### 注册自定义模板引擎

`Spiral\Views\Bootloader\ViewsBootloader` 的 `addEngine` 方法允许你将自定义模板引擎添加到视图组件，并使其可用于你的应用程序。

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Views\Bootloader\ViewsBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ViewsBootloader $views): void
    {
        $views->addEngine(MyEngine::class);
    }
}
```

## 事件

| 事件                          | 描述                                    |
|---------------------------------|-----------------------------------------|
| Spiral\Views\Event\ViewNotFound | 当找不到视图时，将触发此事件。        |

> **注意**
> 要了解有关分发事件的更多信息，请参阅我们文档中的 [事件](../advanced/events.md) 章节。
