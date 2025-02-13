# 组件 — 静态分析

Tokenizer 是一个关键组件，它提供了丰富的功能，极大地增强了 Spiral 应用的开发体验。其主要作用是扫描指定的目录，使开发者能够轻松地管理和组织他们的代码库。该工具特别擅长根据特定的接口或属性识别和使用类，从而促进各种实际应用。

#### 关键用例

1. **自动路由注册**: 一个经典的应用程序是识别控制器动作上的路由属性。通过这样做，它自动化了在应用程序中注册路由的过程，利用在控制器中定义的属性。此功能显著减少了手动开销并简化了路由机制。

2. **模块化结构支持**: 在模块化架构至关重要的项目中，Tokenizer 表现出色。它可以自动检测和注册实现特定接口的模块（例如，作为 composer 包添加的模块）。此功能确保了应用程序内各种模块的无缝集成和管理。

3. **基于属性的配置**: Tokenizer 可以扫描代码库中的特定属性，从而实现基于属性的配置和行为定义。这种方法符合现代编码实践，其中属性用于定义依赖注入、配置设置等各个方面。

## 定位器

定位器就像你代码的探照灯，它们帮助你找到特定的代码片段。

### 类定位器

如果你想定位类，请使用 `Spiral\Tokenizer\ClassesInterface`。使用它，你可以根据类的名称、它们实现的接口或它们包含的 traits 来搜索类。

这里有一个简单的例子。假设你想定位所有实现 `\Psr\Http\Server\MiddlewareInterface` 接口的类：

```php
use Spiral\Tokenizer\ClassesInterfac;

public function findClasses(ClassesInterface $classes): void
{
    foreach ($classes->getClasses(\Psr\Http\Server\MiddlewareInterface::class) as $middleware) {
        dump($middleware->getFileName());
    }
}
```

`getClasses` 方法将返回一个表示找到的类的 `ReflectionClass` 对象数组。

### 枚举定位器

如果你正在寻找枚举，那么 `Spiral\Tokenizer\EnumsInterface` 就是你所需要的。它带有一个 `getEnums` 方法来帮助你进行搜索：

```php
use Spiral\Tokenizer\EnumsInterface;

public function findEnums(EnumsInterface $enums): void
{
    foreach ($enums->getEnums() as $enum) {
        dump($enum->getFileName());
    }
}
```

### 接口定位器

如果你想找到特定的接口，`Spiral\Tokenizer\InterfacesInterface` 可以满足你的需求。像这样使用它的 `getInterfaces` 方法：

```php
use Spiral\Tokenizer\InterfacesInterface;

public function findEnums(InterfacesInterface $interfaces): void
{
    foreach ($interfaces->getInterfaces() as $interface) {
        dump($interface->getFileName());
    }
}
```

> **警告**
> Tokenizer 将忽略所有包含 `include` 或 `require` 语句的文件。这是因为要求这样的反射是不安全的。请不要在你的代码中使用它们。

### 自定义搜索目录

默认情况下，Tokenizer 在 app 目录中进行搜索。但是，你可能经常希望它也考虑其他目录。幸运的是，使用 `Spiral\Bootloader\TokenizerBootloader` 可以轻松地自定义这一点。

:::: tabs

::: tab 引导程序

以下是指定其他目录的方法：

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Bootloader\TokenizerBootloader;
use Spiral\Boot\DirectoriesInterface;

class AppBootloader extends Bootloader
{
    public function init(DirectoriesInterface $directories, TokenizerBootloader $tokenizer): void
    {
        $tokenizer->addDirectory($directories->get('vendor') . 'spiral/validator/src');
    }
}
```

:::

::: tab 配置

或者，你可以在 `app/config/tokenizer.php` 配置文件中直接添加目录：

```php app/config/tokenizer.php
return [
    'directories' => [
        directory('app'),
        directory('vendor') . 'spiral/validator/src',
    ],
];
```

:::

::::

### 排除特定目录

你可能还想从 Tokenizer 的搜索中排除特定目录。方法如下：

```php app/config/tokenizer.php
return [
    'directories' => [
        //...
    ],
    'exclude' => [
        directory('resources'),
        directory('config'),
        'tests',
        'migrations',
    ],
];
```

> **注意**
> 请记住，扩展用于类搜索的目录会减慢进程。建议仅添加对你来说必不可少的目录。

### 作用域类定位器

当处理庞大的目录时，Tokenizer 可能会稍微变慢。但是，你可以通过使用作用域类定位器来提高它的速度。此工具允许你通过设置特定的搜索区域（我们称之为“作用域”）来分而治之。

如果你需要在大量目录中搜索类，Tokenizer 组件的性能可能会很差。在这种情况下，你可以使用作用域类定位器来提高性能。

使用作用域类定位器，你可以定义要在命名的作用域内搜索的目录。这允许你选择性地仅搜索与你当前任务相关的目录。

#### 设置作用域

要使用作用域类定位器，你需要定义你的作用域：

:::: tabs

::: tab 配置

作用域可以在 `app\config\tokenizer.php` 配置文件中定义。

以下是如何定义名为 `scopeName` 的作用域，该作用域搜索 `app/Directory` 目录的示例：

```php app/config/tokenizer.php
return [
    'scopes' => [
        'scopeName' => [
            'directories' => [
                directory('app') . 'Directory',
            ],
            'exclude' => [
                directory('app') . 'Directory/Excluded',
            ]
        ],
    ]
];
```

> **注意**
> `exclude` 参数是有原因的。如果你知道目录的某些部分你不需要，只需告诉 Tokenizer 跳过它们即可。它会使事情变得更快！

:::

::: tab 引导程序

作用域可以使用 `Spiral\Bootloader\TokenizerBootloader` 引导程序定义：

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Bootloader\TokenizerBootloader;
use Spiral\Boot\DirectoriesInterface;

class AppBootloader extends Bootloader
{
    public function init(DirectoriesInterface $directories, TokenizerBootloader $tokenizer): void
    {
        $tokenizer->addScopedDirectory('scopeName', $directories->get('app') . 'Directory');
    }
}
```

:::

::::

准备好作用域后，你就可以指示 Tokenizer 仅在选定的作用域内搜索。这就像告诉它去哪个部门一样！

:::: tabs

::: tab 类

`Spiral\Tokenizer\ScopedClassesInterface` 允许你使用其 `getScopedClasses` 方法来完成此操作。只需将作用域的名称传递给它，它将返回在该区域中找到的所有类。

要使用此方法，你需要将 `scope` 的名称作为参数传递。然后该方法将返回一个 `ReflectionClass` 对象数组，表示在该作用域内找到的类。

```php
use Spiral\Tokenizer\ScopedClassesInterface;

final class SomeLocator
{
    public function __construct(
        private readonly ScopedClassesInterface $locator
    ) {
    }

    public function findDeclarations(): array
    {
        foreach ($this->locator->getScopedClasses('scopeName') as $class) {
            // ...
        }
    }
}
```

:::

::: tab 枚举

就像类一样，你可以为搜索枚举设置特定的作用域。在 `app\config\tokenizer.php` 文件中配置好你的作用域后，你可以使用 `Spiral\Tokenizer\ScopedEnumsInterface`。

下面是一个示例，说明如何在特定作用域中搜索枚举：

```php
use Spiral\Tokenizer\ScopedEnumsInterface;

final class EnumSearcher
{
    public function __construct(
        private readonly ScopedEnumsInterface $locator
    ) {
    }

    public function pinpointEnums(): array
    {
        $foundEnums = [];

        foreach ($this->locator->getScopedEnums('scopeName') as $enum) {
            $foundEnums[] = $enum;
            // or any other operations you want...
        }

        return $foundEnums;
    }
}
```

:::

::: tab 接口

与上述类似，一旦你在配置文件中设置好作用域，`Spiral\Tokenizer\ScopedInterfacesInterface` 将帮助你搜索那些指定区域内的接口。

以下是如何在指定作用域中搜索接口：

```php
use Spiral\Tokenizer\ScopedInterfacesInterface;

final class InterfaceSearcher
{
    public function __construct(
        private readonly ScopedInterfacesInterface $locator
    ) {
    }

    public function findInterfaces(): array
    {
        $identifiedInterfaces = [];

        foreach ($this->locator->getScopedInterfaces('scopeName') as $interface) {
            $identifiedInterfaces[] = $interface;
            // or add your desired actions...
        }

        return $identifiedInterfaces;
    }
}
```

:::

::::

## 使用类监听器的有效扫描

对于大型代码库，使用 Tokenizer 的定位器进行常规扫描可能会减慢速度，尤其是在你经常搜索类、枚举或接口时。类监听器提供了一种更智能的方法，允许你在不重复扫描目录的情况下监听和响应类发现。

### 为什么要使用类监听器？

想象一下，每次想要一本书时都必须搜索一个非常大的图书馆。这是很多工作。现在，想象一下，有人每次图书馆里有新书到达时都会告诉你。这类似于类监听器的工作方式。你不会每次都搜索图书馆，而是在新书（类）到达时得到提醒。这在应用程序引导期间特别有用，在应用程序引导期间会进行初始扫描，之后监听器会保持在循环中。

### 配置监听器

默认情况下，监听器侧重于类，但你可以轻松地将它们配置为扩大它们的范围，以涵盖枚举和接口。为此，你需要在 `app\config\tokenizer.php` 文件中添加以下配置：

```php app/config/tokenizer.php
return [
    'load' => [
        'classes' => true, // 搜索类
        'enums' => true, // 搜索枚举
        'interfaces' => true, // 搜索接口
    ],
];
```

> **注意**
> 请记住，你不必同时启用这三个。根据你项目的需要进行调整。你启用的越多，该过程就会越慢。

### 使用方法

要使用此功能，你需要在引导程序列表的顶部包含 `Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader` 引导程序：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineSystemBootloaders(): array
{
    return [
        \Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const SYSTEM = [
    \Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::::

### 创建监听器

下一步是创建一个监听器类。这个类应该实现 `Spiral\Tokenizer\TokenizationListenerInterface`，它要求两个方法：

- `listen(\ReflectionClass $class)`: 每次发现一个类时都会调用此方法。你可以添加逻辑来处理或存储来自此类的信息。
- `finalize()`: 把它想象成闭幕式。一旦扫描和处理了所有类，就会调用此方法。它是根据发现的类包装事物或完成操作的完美位置。

**这是一个监听器的示例：**

```php
use Spiral\Attributes\ReaderInterface;

final class RouteAttributeListener implements TokenizationListenerInterface
{
    private array $attributes = [];

    public function __construct(
        private readonly ReaderInterface $reader,
        private readonly RouterInterface $router
    ) {
    }
    
    public function listen(\ReflectionClass $class): void
    {
        foreach ($class->getMethods() as $method) {
            $route = $this->reader->firstFunctionMetadata($method, Route::class);

            if ($route === null) {
                continue;
            }

            $this->attributes[] = [$method, $route];
        }
    }

    public function finalize(): void
    {
        foreach ($this->attributes as [$method, $route]) {
            $this->router->addRoute(...);
        }
    }
}
```

例如，此监听器侦听具有特定路由属性的类，并在扫描完成后将它们添加到路由器。

### 缓存监听器目标

为了提高应用程序的性能，你可以使用 `Spiral\Tokenizer\Attribute\TargetAttribute` 和 `Spiral\Tokenizer\Attribute\TargetClass` 属性来过滤由监听器处理的类和属性。这允许你通过过滤由监听器处理的类和属性来提高代码的性能。

当你使用属性来过滤由监听器处理的类时，组件会在应用程序的第一次引导后将过滤后的类缓存在 `runtime/cache/listeners` 目录中。

缓存过滤后的类为你的应用程序提供了几个好处。它大大减少了处理代码库所需的时间，因为类定位器可以从缓存中加载过滤后的类，而不是在每次应用程序启动时重新扫描你的代码库。这有助于提高应用程序的性能并减少应用程序引导所需的时间。

默认情况下，禁用过滤类的缓存。如果要启用缓存，可以将 `TOKENIZER_CACHE_TARGETS` 环境变量设置为 `true`。

```dotenv .env
TOKENIZER_CACHE_TARGETS=true
```

#### TargetAttribute

它允许你根据类的属性过滤类。
当指定目标属性时，类定位器将仅处理具有该属性的类。
如果你有一个只需要分析特定类型类（例如具有特定路由属性的控制器类）的监听器，这可能很有用。

**这是一个如何使用它的示例：**

```php
use Spiral\Tokenizer\Attribute\TargetAttribute;

#[TargetAttribute(Route::class, useAnnotations: true)]
final class RouteLocatorListener implements TokenizationListenerInterface
{
    // ...
}
```

在此示例中，`RouteAttributeListener` 将仅处理具有 `Route` 属性的类。
这意味着如果类定位器找到了一个没有此属性的类，
它不会调用此监听器的 `listen` 方法。

你可以向你的监听器类添加多个属性：

```php
use Spiral\Tokenizer\Attribute\TargetAttribute;
use Spiral\Tokenizer\TokenizationListenerInterface;

#[TargetAttribute(Route::class)]
#[TargetAttribute(SymfonyRoute::class)]
class RouteLocatorListener implements TokenizationListenerInterface
{
    public function listen(\ReflectionClass $class): void
    {
        // 对具有 Route 或 SymfonyRoute 属性的类执行某些操作
    }
}
```

你还可以将第二个参数 `useAnnotations: true` 传递给该属性，以指定 Tokenizer 也应该在类注释中查找目标属性。

使用 `scanParents: true` 来定位在其父类或接口中具有目标属性的类。

#### TargetClass

它的工作方式与 `TargetAttribute` 类似，但它不是根据属性过滤类，而是根据其类型过滤类。如果你有一个只需要分析特定类型类（例如控制器、实现特定接口或扩展特定类的类）的监听器，这很有用。

**这是一个如何使用的示例**

```php
use Spiral\Tokenizer\Attribute\TargetClass;

#[TargetClass(SymfonyCommand::class)]
final class CommandLocatorListener implements TokenizationListenerInterface
{
    // ...
}
```

在此示例中，监听器将处理所有扩展 `SymfonyCommand` 的类。这意味着如果类定位器找到一个扩展它的类，它将调用此监听器的 `listen` 方法。

> **注意**
> 你可以向你的监听器类添加多个属性。

### 监听器注册

要注册你的监听器，你需要使用 `Spiral\Tokenizer\TokenizerListenerRegistryInterface`。

这是一个如何注册监听器的示例：

```php
use Spiral\Tokenizer\TokenizerListenerRegistryInterface;

class AppBootloader extends Bootloader
{
    public function init(
        TokenizerListenerRegistryInterface $listenerRegistry,
        RouteAttributeListener $listener
    ): void {
        $listenerRegistry->addListener($listener);
    }
}
```

> **警告**
> 为了确保正确调用你的监听器，请确保在应用程序 Kernel 的 `LOAD` 部分内的引导程序中注册它们。如果你在 `APP` kernel 部分内注册它们，则不会调用监听器。

## 命令行命令

### 信息

想知道 Tokenizer 是如何设置的吗？ 使用 `tokenizer:info` 命令。

**只需运行以下命令：**

```terminal
php app.php tokenizer:info
```

#### 你将看到什么

1.  **包含的目录：** 显示 Tokenizer 在哪些文件夹中查找。
2.  **排除的目录：** Tokenizer 忽略的文件夹。
3.  **加载器：** 告诉你 Tokenizer 正在寻找哪种 PHP 内容（例如类或接口）。它还会显示如何打开或关闭这些。
4.  **Tokenizer 缓存：** 显示是否使用快捷方式（缓存）来加速事情。你也可以打开或关闭此选项。

#### 示例输出

```terminal
包含的目录：
+------------------------------------+-------+
| 目录                              | 范围  |
+------------------------------------+-------+
| /vendor/intruforce/grpc-shared/src |       |
| app/                               |       |
+------------------------------------+-------+

排除的目录：
+-----------+-------+
| 目录      | 范围  |
+-----------+-------+

加载器：
+------------+------------------------------------------------------------------------------+
| 加载器     | 状态                                                                       |
+------------+------------------------------------------------------------------------------+
| 类         | 已启用                                                                      |
| 枚举       | 已禁用。要启用，请将 "TOKENIZER_LOAD_ENUMS=true" 添加到你的 .env 文件。      |
| 接口       | 已禁用。要启用，请将 "TOKENIZER_LOAD_INTERFACES=true" 添加到你的 .env 文件。 |
+------------+------------------------------------------------------------------------------+

Tokenizer 缓存：已禁用
要启用缓存，请将 "TOKENIZER_CACHE_TARGETS=true" 添加到你的 .env 文件。
```
