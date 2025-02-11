# Cookbook — Spiral 框架升级指南：2.14 到 3.0

Spiral 框架 3.0 版本的发布带来了对 2.x 系列的重大增强和改变，包括对 Symfony 6.x 和 7.x 组件的支持，各种错误修复和可用性改进。

## 关键变化

### 影响：高

#### PHP 8.1

Spiral 框架 3.x 要求最低 PHP 版本为 8.1。

现在，Spiral 中的所有类和接口都包含了类型化的参数、属性和返回类型。如果你的应用程序或包扩展了任何 Spiral 的核心类并重写了这些方法或属性，则需要在你的代码中引入类型声明。

> 最初，请关注 `InjectableConfig::$config` 属性，该属性现在必须被类型化为数组。

#### 只读属性

请注意，许多类属性在最新版本中已被标记为 `readonly`。如果你的代码涉及直接设置 Spiral 核心对象中的值，你需要验证这些属性是否仍然可以被覆盖。此更改要求你调整与 Spiral 核心对象的交互方式，可能需要使用构造函数参数或专用方法来设置这些值。

## 更新依赖项

### 影响：高

更新应用程序的 `composer.json` 文件中的依赖项以确保与 Spiral 框架 3.x 兼容至关重要。请按照以下说明更新并添加必要的包。

### 需要更新

修改你的 `composer.json` 以更新以下包：

- `symfony/console` 更新到 `^6.0`
- `spiral/framework` 更新到 `^3.0`
- `spiral/cycle-bridge` 更新到 `^2.0`
- `spiral/roadrunner-bridge` 更新到 `^3.0`
- `spiral/nyholm-bridge` 更新到 `^1.3`
- `spiral/testing` 更新到 `^2.0`

更新并添加这些依赖项后，运行 `composer update` 以安装新版本，并确保你的项目已准备好用于 Spiral 3.x。

## 环境

### 影响：低

`Environment` 类不再覆盖现有的环境变量（ENV）。

现在，在你的 `.env` 文件中定义的变量可以被外部环境变量覆盖。这意味着在服务器级别或作为系统级环境变量设置的变量将具有优先权。此更改还允许你专门为 PHPUnit 设置环境变量。

要恢复到之前的行为，即 `.env` 文件变量可以覆盖外部环境变量，你需要使用 overwrite 参数设置为 `true` 来实例化 `\Spiral\Boot\Environment` 对象。

#### 示例

```php
App::create(
    directories: ['root' => __DIR__]
)->run(new \Spiral\Boot\Environment(overwrite: true));
```

> 关于如何处理环境变量的[更多信息](../framework/kernel.md#environment)

## Http

### 影响：高

Bootloader 不再自动在应用程序中注册中间件。

#### 以前通过 Bootloader 注册的中间件

以下列表包括以前通过特定 Bootloader 注册的中间件。在更新后，这些中间件需要根据你的应用程序需求和所需的执行顺序手动注册：

- `Spiral\Broadcasting\Bootloader\WebsocketsBootloader` -> `Spiral\Broadcasting\Middleware\AuthorizationMiddleware`
- `Spiral\Bootloader\Auth\HttpAuthBootloader` -> `Spiral\Auth\Middleware\AuthMiddleware`
- `Spiral\Bootloader\Debug\HttpCollectorBootloader` -> `Spiral\Debug\StateCollector\HttpCollector`
- `Spiral\Bootloader\Http\CookiesBootloader` -> `Spiral\Cookies\Middleware\CookiesMiddleware`
- `Spiral\Bootloader\Http\CsrfBootloader` -> `Spiral\Csrf\Middleware\CsrfMiddleware`
- `Spiral\Bootloader\Http\ErrorHandlerBootloader` -> `Spiral\Http\Middleware\ErrorHandlerMiddleware`
- `Spiral\Bootloader\Http\JsonPayloadsBootloader` -> `Spiral\Http\Middleware\JsonPayloadMiddleware`
- `Spiral\Bootloader\Http\SessionBootloader` -> `Spiral\Session\Middleware\SessionMiddleware`

此更改使你能够灵活地选择对你的应用程序至关重要的中间件以及它们应该被集成的顺序。

> 关于 HTTP 中间件的[更多信息](../http/middleware.md)

#### 示例

```php
<?php

declare(strict_types=1);

namespace App\Bootloader;

use App\Controller\AuthController;
use App\Controller\HomeController;
use App\Controller\PageController;
use Spiral\Auth\Middleware\AuthMiddleware;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Cookies\Middleware\CookiesMiddleware;
use Spiral\Csrf\Middleware\CsrfMiddleware;
use Spiral\Debug\StateCollector\HttpCollector;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Http\Middleware\JsonPayloadMiddleware;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;
use Spiral\Session\Middleware\SessionMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            ErrorHandlerMiddleware::class,
            JsonPayloadMiddleware::class,
            HttpCollector::class
        ];
    }

    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                CookiesMiddleware::class,
                SessionMiddleware::class,
                CsrfMiddleware::class,
                AuthMiddleware::class,
            ],
            'api' => [
                //
            ]
        ];
    }
}
```

### 影响：低

Spiral 的 HTTP 组件不再开箱即用默认实现 `Spiral\Http\EmitterInterface`。

要在你的应用程序中使用 HTTP 时包含一个 emitter 实现，例如，如果你想将 Spiral 与 `nginx` 一起使用，你可以通过运行以下命令添加一个特定的包：

```terminal
composer require spiral/sapi-bridge
```

之后，确保在你的应用程序的引导过程中注册必要的 bootloader。这可以通过在你的应用程序的 bootloader 配置中的 LOAD 数组中包含 `Spiral\Sapi\Bootloader\SapiBootloader` 类来完成：

```php app/Application/Bootloader/AppBootloader.php
protected const LOAD = [
    // ...
    \Spiral\Sapi\Bootloader\SapiBootloader::class,
];
```

> 关于如何在[文档](../start/server.md#roadrunner-bridge)中安装和配置 RoadRunner bridge 的[更多信息]。

### 影响：中

Spiral 已停止使用其自己的 PSR-7 接口实现，而是选择集成 `nyholm/psr7` 包。

由于此更改，以下类已从框架中删除：

- `Spiral\Bootloader\Http\DiactorosBootloader`
- `Spiral\Http\Diactoros\ResponseFactory`
- `Spiral\Http\Diactoros\ServerRequestFactory`
- `Spiral\Http\Diactoros\StreamFactory`
- `Spiral\Http\Diactoros\UploadedFileFactory`
- `Spiral\Http\Diactoros\UriFactory`

为了适应此更改，你应该切换到使用 `nyholm/psr7` 用于 PSR-7 HTTP 消息接口。这涉及更新你的应用程序的依赖项，并可能调整先前依赖于现已删除的 Spiral 类的任何自定义实现。这种转变使 Spiral 与广泛采用的且社区支持的 PSR-7 实现保持一致，从而促进了互操作性和标准化。

### 影响：中

Spiral 对用于错误处理的 `RendererInterface` 进行了重大更改。具体来说，`Spiral\Http\ErrorHandler\RendererInterface` 中 `renderException` 方法的第三个参数已被修改。以前接受字符串作为错误消息，现在需要一个 `\Throwable` 作为异常本身。

更新后的方法签名如下：

```php
interface RendererInterface
{
    public function renderException(Request $request, int $code, \Throwable $exception): Response;
}
```

此更改意味着在实现 `RendererInterface` 时，你的方法现在必须处理一个 `\Throwable` 对象，而不是一个简单的字符串消息。此调整允许更详细的错误处理和报告，因为整个异常对象，包括其堆栈跟踪和其他详细信息，可以在 renderException 方法中使用。

### 影响：低

我们不再使用 `laminas/diactoros` 包。将其替换为 `nyholm/psr7`

### 影响：低

Spiral 现在支持自定义输入包的定义，在处理不同类型的输入数据方面提供了更大的灵活性。

#### 示例

以下示例说明了如何定义一个自定义输入包，以使用 Symfony 的 FilesBag 处理文件上传。

```php
<?php

declare(strict_types=1);

namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\HttpBootloader;
use Spiral\Validation\Symfony\Http\Request\FilesBag;

class SampleBootloader extends Bootloader
{
    public function init(HttpBootloader $http): void
    {
        $http->addInputBag('symfonyFiles', [
            'class'  => FilesBag::class,
            'source' => 'getUploadedFiles',
            'alias' => 'symfony-file'
        ]);
    }
}
```

要通过自定义输入包访问文件，你可以使用以下方法：

```php
$container->get(\Spiral\Http\Request\InputManager::class)->symfonyFiles('avatar');
```

此功能增强了 Spiral 内输入处理的模块化和可扩展性，使开发人员能够根据其应用程序的特定需求定制框架的请求处理能力。

> 关于 HTTP 输入管理器的[更多信息](../http/request-response.md#inputmanager)

### 影响：低

`InputBag` 类的默认行为已更改，涉及其如何处理具有 `null` 值的键。此调整会影响你检查 `InputBag` 实例中键的存在及其值的方式。

以前，人们可能隐式地期望 `has` 和 `isset` 的行为保持一致。然而，在更新后的行为中，存在区别：

```php
use Spiral\Http\Request\InputBag;

$bag = new InputBag([4 => null]);

$bag->has(4); // true
isset($bag[4]); // false
```

此更改强调了检查包中键的存在（`has` 方法）与检查键是否存在且具有非空值（`isset`）之间的区别。`has` 方法只是验证键的存在，而不考虑它的值，而 `isset` 检查键，并确保该值不为 null。

### 其他改进

- 添加了 AuthTransportMiddleware。参见 [PR](https://github.com/spiral/framework/pull/717)
- 添加了通过 Autowire 使用中间件的功能。参见 [PR](https://github.com/spiral/framework/pull/726)
- 改进了路由组中间件。参见 [PR](https://github.com/spiral/framework/pull/730)

## Bootloader

### 影响：中

作为 Spiral 不断发展的一部分，几个 bootloader 已被重新定位到它们各自的组件命名空间中。此重组旨在改进模块化，并使管理框架内的特定功能更容易。下面是已移动的 bootloader 及其新位置的列表：

- `Spiral\Bootloader\TokenizerBootloader` -> `Spiral\Tokenizer\Bootloader\TokenizerBootloader`
- `Spiral\Bootloader\AttributesBootloader` -> `Spiral\Attributes\Bootloader\AttributesBootloader`
- `Spiral\Bootloader\Distribution\DistributionBootloader` -> `Spiral\Distribution\Bootloader\DistributionBootloader`
- `Spiral\Bootloader\Distribution\DistributionConfig` -> `Spiral\Distribution\Config\DistributionConfig`
- `Spiral\Bootloader\Storage\StorageBootloader` -> `Spiral\Storage\Bootloader\StorageBootloader`
- `Spiral\Bootloader\Storage\StorageConfig` -> `Spiral\Storage\Config\StorageConfig`
- `Spiral\Bootloader\Security\ValidationBootloader` -> `Spiral\Validation\Bootloader\ValidationBootloader`
- `Spiral\Bootloader\Views\ViewsBootloader` -> `Spiral\Views\Bootloader\ViewsBootloader`
- `Spiral\Bootloader\Http\DiactorosBootloade` -> `Spiral\Http\Bootloader\DiactorosBootloader`

### 影响：高

Spiral 的 bootloading 过程中已经进行了重要的重命名方法，以更好地反映它们的目的和功能：

- `boot` 方法已重命名为 `init`。
- `start` 方法已重命名为 `boot`。

此更改旨在阐明 bootloader 的生命周期阶段。现在，`init` 方法用于初始设置和配置，而 `boot` 方法用于启动或启用 bootloader 提供的功能。

### 影响：低

Spiral 现在自动 bootload 在 bootloader 的 `init` 和 `boot` 方法中声明的所有依赖项。这意味着你不再需要在 `DEPENDENCIES` 常量中显式列出 bootloader，以便框架识别和加载它们。

例如，直接将 bootloader 注入到 boot 或 init 方法中：

```php
public function boot(AttributesBootloader $bootloader)
// or
public function init(AttributesBootloader $bootloader)
```

在功能上等同于声明依赖项的先前方法：

```php
protected const DEPENDENCIES = [
    AttributesBootloader::class,
];
```

此增强通过减少冗余并使过程更直观来简化 bootloader 的配置。

> 关于 bootloader 的[更多信息](../framework/bootloaders.md)

## Kernel

### 影响：中

`Spiral\Boot\AbstractKernel::init` 方法已被删除。为了初始化应用程序，应该使用 `create` 方法作为替代。此更改简化了设置和运行 Spiral 应用程序的过程。

#### 示例

要初始化并运行你的应用程序，现在应该使用以下方法：

```php
App::create(
    directories: ['root' => __DIR__]
)->run();
```

### 影响：中

`Spiral\Boot\AbstractKernel` 的 `create` 方法已被标记为 final。此修改表明该方法不能被覆盖。

### 影响：低

`Spiral\Boot\AbstractKernel` 内的方法命名已更新，以更好地反映其功能：

- `starting` 方法已重命名为 `booting`。
- `started` 方法已重命名为 `booted`。

### 其他改进

- 为 Kernel 类添加了一个新的回调。参见 [PR](https://github.com/spiral/framework/pull/720)

## 可注入的配置

### 影响：中

Spiral 增强了 `InjectableConfig` 类，以自动合并在 `InjectableConfig` 子类的 `$config` 属性中指定的预定义配置，以及从外部配置文件加载的配置。这种改进通过允许预定义的默认值通过外部配置文件轻松覆盖或扩展，从而促进了更灵活和动态的配置管理。

#### 示例

考虑以下示例，其中自定义配置类扩展了 `InjectableConfig` 类。该类中的预定义配置将与外部配置文件的内容合并：

```php
class SomeConfig extends \Spiral\Core\InjectableConfig
{
    // 预定义的默认配置
    protected array $config = [   // <=== 将与 some-config.php 中的数据合并
        'default' => 'sync',
        'aliases' => []
    ];
}
```

假设你有一个外部配置文件 `app/config/some-config.php`，其中包含附加或覆盖设置：

```php
// app/config/some-config.php

return [
    'aliases' => [...]
];
```

当实例化并从容器中获取 `SomeConfig` 类时，生成的配置将是预定义的 `$config` 数组和 `some-config.php` 文件内容的合并版本。

```php
var_dump($container->get(SomeConfig::class)->default); // sync
```

这种机制确保了可以在代码中直接指定默认设置，同时允许通过外部配置文件进行灵活的覆盖或添加，从而结合了两种方法管理应用程序配置的优点。

> 关于可注入配置的[更多信息](../framework/config.md)

## 数据网格

### 影响：低

[spiral/data-grid](https://github.com/spiral/data-grid) 包不再默认与 Spiral 框架捆绑在一起。如果你的应用程序依赖于数据网格组件，你现在需要将其显式包含为依赖项。

要将数据网格组件添加到你的项目中，请执行以下命令：

```bash
composer require spiral/data-grid
```

有关如何将数据网格组件作为独立包安装和使用的详细说明，包括其设置和配置，请参阅[文档](../component/data-grid.md)。

## 队列

### 影响：高

Spiral 对其队列系统进行了重大更改，从 `Spiral\Queue\QueueTrait` 中删除了 `pushCallable` 方法。这意味着默认情况下不再支持将对象和 callable 直接推入队列，因为框架已停止开箱即用地使用 `opis/closure`。

### 其他改进

- 添加了为不同类型的作业配置序列化的功能。
  参见 [PR](https://github.com/spiral/framework/pull/749)
- 添加了为作业设置 defaultSerializer 的功能。参见 [PR](https://github.com/spiral/framework/pull/768)

## 验证

Spiral 框架已经演进其验证方法，不再将其核心包中提供默认的验证器。此决定使开发人员能够选择最适合其项目需求的验证工具，从而确保在应用程序开发中更大的灵活性和适应性。

现在，开发人员有机会在几个验证选项中进行选择，包括：

- [spiral/validatior](../validation/spiral.md)
- [spiral-packages/symfony-validator](../validation/symfony.md)
- 和 [spiral-packages/laravel-validator](../validation/laravel.md)

或创建他们[自己的验证器](../validation/factory.md)

### 影响：高

目前，框架中没有 `Spiral\Validation\ValidationInterface` 的默认实现。要在你的 Spiral 项目中集成验证，你应该从上面提供的选项或其他首选验证器中选择一个验证器，然后通过 Composer 安装它。例如，要使用 Spiral 自己的验证包：

```bash
composer require spiral/validatior
```

选择并安装验证器后，你必须在应用程序的 bootloader 中将 `ValidationInterface` 与你选择的实现绑定。

```php
use Spiral\Validation\ValidationInterface;
use Spiral\Validation\ValidationProviderInterface;
use Spiral\Validator\FilterDefinition;

class AppBootloader extends Bootloader
{
    protected const SINGLETONS = [
        ValidationInterface::class => [self::class, 'initValidation'],
    ];

    // ...

    public function initValidation(ValidationProviderInterface $provider): ValidationInterface
    {
        return $provider->getValidation(FilterDefinition::class);
    }
}
```

## spiral/monolog-bridge

### 影响：高

常量 `Spiral\Monolog\LogFactory::DEFAULT` 已在 Spiral 框架的 2.x 版本中被弃用，并在 3.x 版本中被删除。作为替代方案，请使用 `Spiral\Monolog\Config\MonologConfig::DEFAULT_CHANNEL` 来引用默认的日志记录通道。

此更改需要在你的应用程序的日志记录配置或先前依赖于 `LogFactory::DEFAULT` 的任何代码引用中进行更新。应进行调整以使用 `MonologConfig::DEFAULT_CHANNEL` 以确保与 Spiral 框架 3.x 兼容。

### 影响：低

要配置默认的 Monolog 通道，你可以如下设置一个环境变量：

```dotenv
# ...
MONOLOG_DEFAULT_CHANNEL=stderr
```

此外，你可以在应用程序的配置文件中直接在 Monolog 配置数组中指定此默认通道并配置处理程序和处理器：

```php
[
    'default' => env('MONOLOG_DEFAULT_CHANNEL', 'stderr'),
    'handlers' => [
        'default' => [
            // ...
        ],
        'stderr' => [
            // ...
        ],
        'stdout' => [
            // ...
        ],
    ],

    'processors' => [
        'default' => [
            // ...
        ],
        'stdout' => [
            // ...
        ],
    ],
]
```

> 参见 [pull request](https://github.com/spiral/framework/pull/675)

## 弃用和移除

#### RoadRunner

在 Spiral 框架 3.x 中，内置的 RoadRunner 集成已被删除，以简化核心框架并在选择服务器解决方案时提供更大的灵活性。对于需要 RoadRunner 集成的项目：

- 使用 [spiral/roadrunner-bridge](https://github.com/spiral/roadrunner-bridge) 包将 RoadRunner 与 Spiral 框架无缝集成。

以下组件也已被删除并替换为其各自的替代方案：

- `spiral/jobs` 不再是 Spiral 框架的一部分。相反，请使用 [spiral/queue](https://spiral.dev/docs/queue-configuration/3.0/en) 组件进行作业队列管理。
- `spiral/broadcast` 已被 [spiral/broadcasting](https://spiral.dev/docs/component-broadcasting/3.0/en) 组件替换，提供增强的事件广播功能。

#### CycleORM、Database、Migrations

Spiral 框架 3.x 已将其与 `cycle/orm`、`cycle/database` 和 `cycle/migrations` 的直接集成解耦，从而允许更模块化的架构。要将 CycleORM 集成到你的 Spiral 项目中：

- 使用 [spiral/cycle-bridge](https://github.com/spiral/cycle-bridge) 包，该包促进了 CycleORM 和相关组件的集成。

## 新的异常处理程序

Spiral 框架现在通过 `Spiral\Exceptions\ExceptionHandler` 集中了所有异常处理，消除了在代码库中进行异常管理时对额外的 snapshotter 和 logger 组件的需求。

更新后的 `ExceptionHandler` 支持为特定环境注册自定义异常渲染器和报告器，从而增强了对异常处理和报告的灵活性和控制。

> 关于异常处理的[更多信息](../basics/errors.md)

## 过滤器

### 影响：低

添加了一个新方法 `Spiral\Filters\InputInterface::hasValue`

### 影响：高

Filters 组件经历了完全的重新设计。要在 3.0 环境中使用 Spiral 版本 2.x 中的过滤器，请包含 [spiral/filters-bridge](https://github.com/spiral/filters-bridge) 包。

### 重新设计的过滤器

1. 不再需要依赖于 `spiral/validation` 组件的验证器。
2. 过滤器现在利用 PHP 属性来映射输入数据。
3. 启用容器解析期间的自动验证，如需要。
4. 现在支持与第三方验证器的兼容性，包括：
    - [spiral/validator](https://spiral.dev/docs/security-validator/3.0/en) (从 `validatior` 修正)
    - [spiral-packages/symfony-validator](https://github.com/spiral-packages/symfony-validator)
    - [spiral-packages/laravel-validator](https://github.com/spiral-packages/laravel-validator)

#### 过滤器示例

以下示例演示了重新设计的过滤器的使用：

```php
use Spiral\Filters\Attribute\Input;
use Spiral\Filters\Attribute\NestedArray;
use Spiral\Filters\Attribute\NestedFilter;
use Spiral\Filters\Attribute\Setter;

class CreateUserFilter extends SymfonyFilter 
{
    #[Asset\NotBlank]
    #[Asset\Length(['min' => 30])]
    #[Input\Post]
    public string $username;

    #[Asset\NotBlank]
    #[Asset\Length(['min' => 30])]
    #[Input\Post(key: 'first_name')]
    public string $name;

    #[Asset\NotBlank]
    #[Asset\Length(['min' => 30])]
    #[Input\Query(key: 'last_name')]
    public string $lastName;

    #[Input\Query(key: 'page_num')]
    public int $page;

    #[Asset\NotBlank]
    #[Input\Attribute]
    private string $wsPath;

    #[Asset\NotBlank]
    #[Input\BearerToken]
    public string $token;

    #[Input\Cookie(key: 'utm_source')]
    public string $utmSource;

    #[Input\Cookie]
    public string $utmId;

    #[NestedArray(
        class: UserTagsFilter::class,
        input: new Inout\Post('tags')
    )]
    public array $tags = [];

    #[NestedFilter(class: UtmFilter::class)]
    public UtmFilter $utm;

    #[Input\Post]
    #[Setter(filter: 'md5')]
    public string $password;
}
```

```php
class UserController
{
    public function createUser(CreateUserFilter $filter): array
    {
        $user = new User(
            $filter->username,
            $filter->firstName,
            $filter->lastName
        );

        $user->setUtmData(
            $filter->utm->id,
            $filter->utm->source,
            ...
        );

        foreach($filter->tags as $tag) {
            $user->addTag($tag->role);
        }

        $this->em->persist($user);
    }
}
```

> 关于过滤器的[更多信息](../filters/configuration.md)

## 可注入的枚举

假设你需要确定你的应用程序是否正在生产模式下运行。一个简单的方法是检查 `$env->get('APP_ENV') === 'production'`。但是，枚举提供了更方便和类型安全的选择。

```php
enum AppEnvironment: string
{
    case Production = 'prod';
    case Stage = 'stage';
    case Testing = 'testing';
    case Local = 'local';

    public function isProduction(): bool
    {
        return $this === self::Production;
    }
}
```

当容器解析枚举类时，它将提供一个枚举对象，其中包含在环境中定义的值。

```php
class MigrateCommand extends Command 
{
    const NAME = '...';

    // 解析并检测当前环境
    public function perform(AppEnvironment $appEnv): int
    {
        if ($appEnv->isProduction()) {
            // 拒绝
            return 0;
        }

        // ...
        return 1;
    }
}
```

要使用此功能，请实现 `Spiral\Boot\Injector\InjectableEnumInterface` 并使用 `Spiral\Boot\Injector\ProvideFrom` 属性来指定一个返回特定枚举对象的方法。

```php
<?php

declare(strict_types=1);

namespace Spiral\Boot\Environment;

use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\Injector\ProvideFrom;
use Spiral\Boot\Injector\InjectableEnumInterface;

#[ProvideFrom(method: 'detect')]
enum DebugMode implements InjectableEnumInterface
{
    case Enabled;
    case Disabled;

    public function isEnabled(): bool
    {
        return $this === self::Enabled;
    }

    public static function detect(EnvironmentInterface $environment): self
    {
        return \filter_var($environment->get('DEBUG'), \FILTER_VALIDATE_BOOL) ? self::Enabled : self::Disabled;
    }
}
}
```

这种方法通过利用枚举进行特定于环境的逻辑来增强类型安全性和代码清晰度。

> 关于可注入枚举的[更多信息](../container/injectors.md#enum-injectors)

## 新的序列化程序组件

Spiral 框架引入了一个新组件 `spiral/serializer`，用于有效载荷的序列化和反序列化。此组件具有灵活性，支持用于各种序列化格式的自定义驱动程序。

### 配置

要配置序列化程序组件，你需要在 `app/config/serializer.php` 文件中定义你喜欢的序列化程序：

```php
// app/config/serializer.php
use Spiral\Core\Container\Autowire;

return [
    'default' => env('DEFAULT_SERIALIZER_FORMAT', 'json'),
    'serializers' => [
        'json' => JsonSerializer::class,
        'yaml' => new YamlSerializer(),
        'custom' => new Autowire(CustomSerializer::class),
    ],
];
```

下面是一个使用序列化程序组件在将有效负载推送到管道之前对其进行序列化的 `Queue` 类的示例：

```php
class Queue 
{
    public function __construct(
        private readonly PipelineInterface $pipeline,
        private readonly \Spiral\Serializer\SerializerInterface $serializer,
    ) {}

    public function push(array $payload): void
    {
        $this->pipeline->push(
            $this-serializer->serialize($payload),
        );
    }
}
```

在需要动态选择序列化格式的情况下，你可以使用 `SerializerManager` 根据运行时提供的选项检索合适的序列化程序：

```php
class Queue 
{
    public function __construct(
        private readonly PipelineInterface $pipeline,
        private readonly \Spiral\Serializer\SerializerManager $manager
    ) {}

    public function push(array $payload, Options $options): void
    {
        // $options->format = 'json'
        // $options->format = 'xml'
        // ...

        $this->pipeline->push(
            $this-manager
                ->getSerializer($options->format)
                ->serialize($payload)
        );
    }
}
```

> 关于序列化程序组件的[更多信息](../advanced/serializer.md)

## 控制台改进

#### 命令签名

自 3.6 版本发布以来，Spiral 提供了使用 PHP 属性定义控制台命令的功能。这允许使用更直观和简化的方法定义具有明确关注点分离的命令。

以下是使用属性定义控制台命令的示例：

```php
namespace App\Api\Cli\Command;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;
use Symfony\Component\Console\Input\InputOption;

#[AsCommand(
    name: 'app:create:user', 
    description: 'Creates a user with the given data')
]
final class CreateUser extends Command
{
    #[Argument]
    private string $email;

    #[Argument(description: 'User password')]
    private string $password;

    #[Argument(name: 'username', description: 'The user name')]
    private string $userName;

    #[Option(shortcut: 'a', name: 'admin', description: 'Set the user as admin')]
    private bool $isAdmin = false;

    public function __invoke(): int
    {
        $user = new User(
            email: $this->email,
            password: $this->password,
        );
        
        $user->setIsAdmin($this->isAdmin);
        
        // Save user to database...

        return self::SUCCESS;
    }
}
```

#### 命名序列

Spiral 现在支持创建命名序列，允许组织和重用命令序列。

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Console\Bootloader\ConsoleBootloader;

class MyPackageAssetsBootloader extends Bootloader
{
    public function init(ConsoleBootloader $console): void
    {
        $console->addSequence('publish:assets', PublishCssFilesCommand::class);
        $console->addSequence('publish:assets', PublishJsFilesCommand::class);
    }
}
```

使用命名序列：

```php
use Spiral\Console\Command\SequenceCommand;
use Psr\Container\ContainerInterface;
use Spiral\Console\Config\ConsoleConfig;

class PublishAssetsCommand extends SequenceCommand
{
    public function perform(ConsoleConfig $config, ContainerInterface $container): int
    {
        $this->info("Publishing assets:");
        $this->newLine();

        return $this->runSequence(
            $config->getSequence('publish:assets'), 
            $container
        );
    }
}
```

#### `__invoke` 方法

命令现在可以使用 `__invoke` 方法作为 `perform` 方法的替代方案，从而简化了命令定义。

```php
class CheckHttpCommand extends Command 
{
    // ...

    public function __invoke(): int
    {
        // ...
    }
}
```

#### 生产环境中确认应用程序

一项新功能确保在生产环境中尝试执行敏感操作时显示确认提示。

```php
use Spiral\Console\Confirmation\ApplicationInProduction;

final class MigrateCommand extends Command
{
    protected const NAME = 'db:migrate';
    protected const DESCRIPTION = '...';

    public function perform(ApplicationInProduction $confirmation): int
    {
        if (!$confirmation->confirmToProceed()) {
            return self::FAILURE;
        }
        
        // run migrations...
    }
}
```

#### 新的辅助方法

引入了一套辅助方法，以促进各种命令行交互：

```php
class SomeCommand extends Command 
{
    // ...

    public function perform(): int
    {
        // 确定是否存在输入选项。
        $this->hasOption(...);

        // 确定是否存在输入参数。
        $this->hasArgument(...);

        // 询问确认。
        $status = $this->confirm('Are you sure?', default: false): bool;

        // 提问。
        $status = $this->ask('Are you sure?', default: 'no'): mixed;

        // 提示用户输入，但将答案隐藏在控制台中。
        $status = $this->secret('User password'): string;

        // 将消息写入信息输出。
        $this->info(...);

        // 将消息写入注释输出。
        $this->comment(...);

        // 将消息写入问题输出。
        $this->question(...);

        // 将消息写入错误输出。
        $this->error(...);

        // 将消息写入警告输出。
        $this->warning(...);

        // 将消息写入警报输出。
        $this->alert(...);

        // 将消息写入标准输出。
        $this->line(..., 'error');

        // 写入一个空白行。
        $this->newLine();

        $this->newLine(count: 5);
    }
}
```

#### 控制台命令拦截器

开发人员现在可以为控制台命令注册自定义拦截器，从而增强对命令执行和生命周期的控制。

> 关于控制台拦截器的[更多信息](../console/interceptors.md)

#### 输出对象替换

现在可以在控制台命令中替换输出对象，从而允许自定义输出处理。

```php
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

abstract class MyCommand extends \Spiral\Console\Command 
{
    protected function prepareOutput(InputInterface $input, OutputInterface $output): OutputInterface
    {
        return new \App\Console\MyOutput($input, $output);
    }
}
```

## 容器重构

Spiral 框架的容器系统经历了重大的重构，增强了其灵活性和功能。下面是一个详细说明这些改进的综合指南。

> 关于容器的[更多信息](../container/overview.md)

### 模块化容器设计

容器现在在模块化的基础上运行，允许在容器初始化之前用自定义实现替换各个模块。此更改引入了更大的灵活性，以使容器适应特定的项目需求。

```php
// 初始化共享容器，绑定，目录等。
$app = App::create(
    directories: ['root' => __DIR__],
    container: new Spiral\Core\Container(
        config: new Spiral\Core\Config(
            resolver: App\CustomResolver::class
        )
    )
)->run();
```

### PHP 8.0 联合类型支持

已添加对 PHP 8.0 联合类型的支持，从而允许在应用程序的代码库中进行更具表现力的类型声明。

### 可变参数支持

容器现在支持可变参数，从而在将参数传递给方法和函数方面提供了更大的灵活性：

- 可以通过参数名称传递参数，包括数组中的命名参数和位置参数。
- 也可以通过参数名称直接传递值。
- 支持位置尾随