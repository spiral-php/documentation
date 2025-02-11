# 基础知识 — Scaffolding（脚手架）

Spiral 框架提供了 `spiral/scaffolder` 组件。这个强大的工具让开发者能够使用一组控制台命令，快速、轻松地为各种类生成应用程序代码，例如：

- 应用程序引导程序 (bootloaders)
- 控制台命令
- 应用程序配置
- HTTP 控制器、中间件、请求过滤器
- 队列任务处理器

等等...

## 安装

要启用该组件，你只需要将 `Spiral\Scaffolder\Bootloader\ScaffolderBootloader` 类添加到引导程序列表中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Scaffolder\Bootloader\ScaffolderBootloader::class,
        // ...
    ];
}
```

在 [框架 — 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Scaffolder\Bootloader\ScaffolderBootloader::class,
    // ...
];
```

在 [框架 — 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的信息。
:::

::::

## 配置

添加引导程序后，你可以通过使用 `scaffolder` 配置文件替换声明生成器及其选项来定制组件以满足你的需求。

以下是可用声明的默认配置：

```php
use Spiral\Scaffolder\Declaration;

return [
    'declarations' => [
        Declaration\BootloaderDeclaration::TYPE => [
            'namespace' => 'Bootloader',
            'postfix' => 'Bootloader',
            'class' => Declaration\BootloaderDeclaration::class,
        ],
        Declaration\ConfigDeclaration::TYPE => [
            'namespace' => 'Config',
            'postfix' => 'Config',
            'class' => Declaration\ConfigDeclaration::class,
            'options' => [
                'directory' => directory('config'),
            ],
        ],
        Declaration\ControllerDeclaration::TYPE => [
            'namespace' => 'Controller',
            'postfix' => 'Controller',
            'class' => Declaration\ControllerDeclaration::class,
        ],
        Declaration\FilterDeclaration::TYPE => [
            'namespace' => 'Filter',
            'postfix' => 'Filter',
            'class' => Declaration\FilterDeclaration::class,
        ],
        Declaration\MiddlewareDeclaration::TYPE => [
            'namespace' => 'Middleware',
            'postfix' => '',
            'class' => Declaration\MiddlewareDeclaration::class,
        ],
        Declaration\CommandDeclaration::TYPE => [
            'namespace' => 'Command',
            'postfix' => 'Command',
            'class' => Declaration\CommandDeclaration::class,
        ],
        Declaration\JobHandlerDeclaration::TYPE => [
            'namespace' => 'Job',
            'postfix' => 'Job',
            'class' => Declaration\JobHandlerDeclaration::class,
        ],
    ],
];
```

你可以为每个可用的类声明类型自定义类的命名空间、后缀和声明类型。没有必要覆盖整个声明配置。相反，你可以仅自定义你需要的特定声明类型。

以下是一个你可以如何自定义配置的示例：

```php app/config/scaffolder.php
use Spiral\Scaffolder\Declaration;

return [
    // ...
    'declarations' => [
        Declaration\MiddlewareDeclaration::TYPE => [
            'class' => Declaration\MiddlewareDeclaration::class,
        ],
        Declaration\CommandDeclaration::TYPE => [
            'namespace' => 'Endpoint\Console',
        ],
        Declaration\JobHandlerDeclaration::TYPE => [
            'namespace' => 'Endpoint\Queue',
            'postfix' => 'Job',
        ],
    ],
];
```

> **注意**
> 这种方法在可能使用许多不同声明类型的大型应用程序中特别有用。通过仅自定义必要的内容，可以简化配置过程并最大限度地减少错误。

### 更改生成类的目录

默认情况下，scaffolder 将在 `app/src` 目录中生成类。在某些情况下，你可能希望更改 scaffolder 生成类的目录。

你可以通过使用 `directory` 选项来实现此目的。

```php app/config/scaffolder.php
return [
    'directory' => directpry('app') . '/Generated' // <=============
];
```

你还可以更改特定声明的目录。例如，你可能希望在 `app/src/Endpoint/Console` 目录中生成控制台命令。为此，你可以使用 `directory` 选项：

```php app/config/scaffolder.php
return [
    'declarations' => [
        Declaration\CommandDeclaration::TYPE => [
            // ...
            'directory' => directpry('app') . '/Endpoint/Console' // <=============
        ],
    ],
];
```

## 通过 ScaffolderBootloader 添加自定义声明

可以注册自定义声明。你可以通过使用 `ScaffolderBootloader` 注册自定义声明来实现此目的：

```php app/src/Application/Bootloader/ScaffolderBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Scaffolder\Bootloader\ScaffolderBootloader as BaseScaffolderBootloader;

final class ScaffolderBootloader extends Bootloader
{
    public function init(BaseScaffolderBootloader $scaffolder): void
    {
        $scaffolder->addDeclaration('Repository', [
            'namespace' => 'App\\Repository',
            'postfix'   => 'Repository',
            'class'     => RepositoryDeclaration::class,
            'options'   => [
                'orm' => 'cycle',
                // some custom options
            ],
        ]);
    }
}
```

通过注册自定义声明，你可以扩展该组件的功能，以满足应用程序的特定需求。

## 可用命令

| 命令                | 描述                        |
|---------------------|-----------------------------|
| create:bootloader   | 创建 Bootloader 声明        |
| create:command      | 创建 Command 声明           |
| create:config       | 创建 Config 声明            |
| create:controller   | 创建 Controller 声明        |
| create:middleware   | 创建 Middleware 声明        |
| create:filter       | 创建 Request 过滤器声明     |
| create:jobHandler   | 创建 Job Handler 声明       |

某些软件包可能提供自己的命令。例如，`Cycle Bridge` 软件包（如果已安装）提供以下命令：

| 命令                | 描述                           |
|---------------------|-------------------------------|
| create:migration    | 创建 Migration 声明          |
| create:repository   | 创建 Entity Repository 声明  |
| create:entity       | 创建 Entity 声明             |

> **了解更多**
> 在 [此处](https://spiral.dev/docs/packages-cycle-bridge) 阅读更多关于 `Cycle Bridge` 软件包和可用控制台命令的信息。

### Bootloader（引导程序）

此命令创建一个 Bootloader 类。引导程序负责在应用程序启动期间初始化和配置组件。使用此命令，你可以快速生成新 Bootloader 的代码，然后可以自定义它以满足你的特定需求。

> **了解更多**
> 在 [框架 — 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的信息。

```terminal
php app.php create:bootloader <name>
```

将创建 `<Name>Bootloader` 类。

#### 配置

在我们的示例中，我们使用了以下声明配置：

```php
Spiral\Scaffolder\Declaration\BootloaderDeclaration::TYPE => [
    'namespace' => 'Application\Bootloader',
],
```

#### 示例

```terminal
php app.php create:bootloader App
```

输出是：

```php app/src/Application/Bootloader/AppBootloader.php
declare(strict_types=1);

namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

final class AppBootloader extends Bootloader
{
    protected const BINDINGS = [];
    protected const SINGLETONS = [];
    protected const DEPENDENCIES = [];

    public function init(): void
    {
    }

    public function boot(): void
    {
    }
}
```

通过使用 `-d` 选项，你可以生成一个 Domain（领域）特定的 Bootloader。

#### 示例

```terminal
php app.php create:bootloader App -d
```

输出是：

```php app/src/Application/Bootloader/AppBootloader.php
declare(strict_types=1);

namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

final class AppBootloader extends DomainBootloader
{
    protected const BINDINGS = [];
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];
    protected const DEPENDENCIES = [];
    protected const INTERCEPTORS = [
        // Put your interceptors here
    ];

    public function init(): void
    {
    }

    public function boot(): void
    {
    }
}
```

### Console Command（控制台命令）

此命令创建一个 Console Command 类。命令提供了一种通过控制台执行应用程序功能的方法。使用此命令，你可以生成一个新的 Command 声明的代码，然后可以自定义它以实现所需的控制台功能。

> **了解更多**
> 在 [控制台 — 入门](../console/configuration.md) 部分阅读更多关于控制台命令的信息。

```terminal
php app.php create:command <name> [alias]
```

将生成 `<Name>Command` 类。命令名称将等于 `name` 或 `alias`（如果设置了此值）。

如果未提供别名，则将从名称自动生成一个。例如，如果名称是 `CreateUser`，则别名将是 `create:user`。

#### 配置

在我们的示例中，我们使用了以下声明配置：

```php
Spiral\Scaffolder\Declaration\CommandDeclaration::TYPE => [
    'namespace' => 'Endpoint\Console',
],
```

#### 示例，没有 `alias`

```terminal
php app.php create:command UserRegister
```

输出是：

```php app/src/Endpoint/Console/UserRegisterCommand.php
declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'user:register')]
final class UserRegisterCommand extends Command
{
    public function __invoke(): int
    {
        // Put your command logic here
        $this->info('Command logic is not implemented yet');

        return self::SUCCESS;
    }
}
```

#### 示例，带别名

```terminal
php app.php create:command UserRegister create:user
```

输出是：

```php app/src/Endpoint/Console/UserRegisterCommand.php
#[AsCommand(name: 'create:user')]
final class UserRegisterCommand extends Command
```

#### 选项和参数

你还可以使用 `-a` 和 `-o` 选项向命令添加参数和选项。

```terminal
php app.php create:command UserRegister -a username -a password -o isAdmin
```

输出是：

```php app/src/Endpoint/Console/UserRegisterCommand.php
declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'user:register')]
final class UserRegisterCommand extends Command
{
    #[Argument(description: 'Argument description')]
    #[Question(question: 'What would you like to name the username argument?')]
    private string $username;

    #[Argument(description: 'Argument description')]
    #[Question(question: 'What would you like to name the password argument?')]
    private string $password;

    #[Option(description: 'Argument description')]
    private bool $isAdmin;

    public function __invoke(): int
    {
        // Put your command logic here
        $this->info('Command logic is not implemented yet');

        return self::SUCCESS;
    }
}
```

#### 命令描述

你还可以使用 `-d` 选项向命令添加描述。

```terminal
php app.php create:command UserRegister -d "Register a new user"
```

输出是：

```php app/src/Endpoint/Console/UserRegisterCommand.php
#[AsCommand(name: 'create:user', description: 'Register a new user')]
final class UserRegisterCommand extends Command
```

### Application Config（应用程序配置）

此命令创建一个 Config 类。Config 是一种管理应用程序配置设置的方法。使用此命令，你可以生成新的 Config 的代码，然后可以自定义它来管理应用程序的配置设置。

> **了解更多**
> 在 [框架 — 配置对象](../framework/config.md) 部分阅读更多关于应用程序配置的信息。

```terminal
php app.php create:config <name>
```

如果不存在，将创建 `<Name>Config` 类和 `<app directory>/config/<name>.php` 文件。

#### 可用选项：

`reverse (r)` - 使用此标志，scaffolder 将查找 `<app directory>/config/<name>.php` 文件并根据其内容创建一个 Config 类。生成的类将包括默认值和 getter，并且在某些情况下，它还将包括基于 key 的数组值 getter。如果一个数组值由多个子值组成，并且 key 和子值的类型相同，则 scaffolder 将尝试创建一个基于 key 的 getter 方法。如果生成的 key 与现有方法冲突，则将省略基于 key 的 getter。

#### 带有空配置文件示例

```terminal
php app.php create:config app
```

输出配置文件：

```php app/config/app.php
return [];
```

输出配置类：

```php app/src/Application/Config/AppConfig.php
declare(strict_types=1);

namespace App\Application\Config;

use Spiral\Core\InjectableConfig;

final class AppConfig extends InjectableConfig
{
    public const CONFIG = 'app';

    /**
     * Default values for the config.
     * Will be merged with application config in runtime.
     */
    protected array $config = [];
}
```

#### 带有反向生成示例

```php app/config/app.php
return [
    //will create "getParam()" by-key-getter (successfully singularized name)
    'params' => [
        'one' => 'param',
        'two' => 'another param',
    ],
    //will create "getParameterBy()" by-key-getter (unsuccessfully singularized name)
    'parameter' => [
        'one' => 'parameter',
        'two' => 'another parameter',
    ],
    //will create "getValueBy()" by-key-getter (because "getValue()" conflicts with the next "value" field)
    'values' => [
        1 => 'value',
        2 => 'another value',
    ],
    'value' => 'third value',
    //won't create by-key-getter due to only 1 sub-value
    'few' => [
        'one' => 'value',
    ],
    //won't create by-key-getter due to mixed values' types
    'mixedValues' => [
        'one' => 'value',
        'two' => 2,
    ],
    //won't create by-key-getter due to mixed keys' types
    'mixedKeys' => [
        'one' => 'value',
        2 => 'another value',
    ],
    //won't create by-key-getter to name conflicts
    //(because "getConflict()" and "getConflictBy()" conflicts with the next "conflict" and "conflictBy" fields)
    'conflicts' => [
        'one' => 'conflict',
        'two' => 'another conflict',
    ],
    'conflict' => 'third conflic',
    'conflictBy' => 'fourth conflic',
];
```

```terminal
php app.php create:config my -r
```

输出是：

```php app/src/Application/Config/AppConfig.php
declare(strict_types=1);

namespace App\Application\Config;

use Spiral\Core\InjectableConfig;

final class AppConfig extends InjectableConfig
{
    public const CONFIG = 'app';

    /**
     * Default values for the config.
     * Will be merged with application config in runtime.
     */
    protected array $config = [
        'params' => [],
        'parameter' => [],
        'values' => [],
        'value' => '',
        'few' => [],
        'mixedValues' => [],
        'mixedKeys' => [],
        'conflicts' => [],
        'conflict' => '',
        'conflictBy' => '',
    ];

    public function getParams(): array
    {
        return $this->config['params'];
    }

    public function getParameter(): array
    {
        return $this->config['parameter'];
    }

    public function getValues(): array
    {
        return $this->config['values'];
    }

    public function getValue(): string
    {
        return $this->config['value'];
    }

    public function getFew(): array
    {
        return $this->config['few'];
    }

    public function getMixedValues(): array
    {
        return $this->config['mixedValues'];
    }

    public function getMixedKeys(): array
    {
        return $this->config['mixedKeys'];
    }

    public function getConflicts(): array
    {
        return $this->config['conflicts'];
    }

    public function getConflict(): string
    {
        return $this->config['conflict'];
    }

    public function getConflictBy(): string
    {
        return $this->config['conflictBy'];
    }

    public function getParam(string $param): string
    {
        return $this->config['params'][$param];
    }

    public function getParameterBy(string $parameter): string
    {
        return $this->config['parameter'][$parameter];
    }

    public function getValueBy(int $value): string
    {
        return $this->config['values'][$value];
    }
}
```

### Http Controller（HTTP 控制器）

此命令创建一个 Controller 类。控制器处理应用程序中特定端点的 HTTP 请求和响应。使用此命令，你可以生成一个新的 Controller 类的代码，然后你可以自定义它以根据需要处理 HTTP 请求和响应。

> **了解更多**
> 在 [HTTP — 入门](../http/configuration.md) 部分阅读更多关于 HTTP 控制器的信息。

```terminal
php app.php create:controller <name>
```

将创建 `<Name>Controller` 类。可用选项：

* `action (a)` (允许多个值) - 你可以使用此选项添加 actions
* `prototype (p)` - 如果设置，将添加 `PrototypeTrait`

#### 配置

在我们的示例中，我们使用了以下声明配置：

```php
Spiral\Scaffolder\Declaration\ControllerDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web',
],
```

#### 带有空 action 列表的示例

```terminal
php app.php create:controller User
```

输出是：

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Router\Annotation\Route;

class UserController
{
}

```

#### 带有 `prototype` 选项的示例

```terminal
php app.php create:controller User -p
```

输出是：

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

class UserController
{
    use PrototypeTrait;
}
```

#### 带有 actions 列表的示例

```bash
php app.php create:controller User \
      -a index \
      -a show \
      -a create \
      -a update \
      -a delete
```

输出是：

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Router\Annotation\Route;

class UserController
{
    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function index(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function show(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function create(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function update(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function delete(): ResponseInterface
    {
    }
}
```

### Request Filter（请求过滤器）

此命令创建一个 Request Filter 类。请求过滤器提供了一种在控制器处理 HTTP 请求之前映射和验证它们的方法。使用此命令，你可以生成一个新的 Request Filter 声明的代码，然后可以自定义它以根据需要修改 HTTP 请求。

> **了解更多**
> 在 [过滤器 — 入门](../filters/configuration.md) 部分阅读更多关于请求过滤器的信息。

```terminal
php app.php create:filter <name>
```

将生成 `<Name>Filter` 类。

#### 配置

在我们的示例中，我们使用了以下声明配置：

```php
Spiral\Scaffolder\Declaration\FilterDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web\Filter',
],
```

#### 示例

```terminal
php app.php create:filter CreateUser
```

> **警告**
> 确保你在你的应用程序中启用了 `Spiral\Validation\Bootloader\ValidationBootloader` 引导程序。

输出是：

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Model\Filter;

final class CreateUserFilter extends Filter
{
}
```

#### 使用属性创建过滤器

`property (p)` - 选项用于为过滤器类定义属性。每个属性都使用 `<name>:<source>:<type>` 格式定义，其中：

- `<name>` 是属性的名称，
- `<source>` 是输入数据的来源（例如，`post`，`get`，`header`，`cookie`，`server` 等），
- `<type>` 是属性的类型（例如，`string`，`int`，`bool` 或 `array`）。

```terminal
php app.php create:filter CreateUser -p username:post -p tags:post:array -p ip:ip -p token:header -p status:query:int
```

输出是：

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Header;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Attribute\Input\RemoteAddress;
use Spiral\Filters\Model\Filter;

final class CreateUserFilter extends Filter
{
    #[Post(key: 'username')]
    public string $username;

    #[Post(key: 'tags')]
    public array $tags;

    #[RemoteAddress(key: 'ip')]
    public string $ip;

    #[Header(key: 'token')]
    public string $token;

    #[Query(key: 'status')]
    public int $status;
}
```

> **注意**
> 在 [此处](../filters/filter.md) 阅读更多关于可用属性的信息。

#### 使用验证规则创建过滤器

要生成带有验证规则的过滤器，只需将 `-s` 选项添加到命令中：

```terminal
php app.php create:filter CreateUser -p ... -s
```

输出是：

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Header;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Attribute\Input\RemoteAddress;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

final class CreateUserFilter extends Filter implements HasFilterDefinition
{
    // ...

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(validationRules: [
            // Put your validation rules here
        ]);
    }
}
```

> **警告**
> 你的应用程序中应该安装一个验证库。在 [此处](../validation/factory.md) 阅读更多关于可用的验证库的信息。

### HTTP Middleware（HTTP 中间件）

此命令创建一个 Middleware 类。中间件提供了一种在 HTTP 请求和响应通过应用程序的中间件堆栈时修改它们的方法。使用此命令，你可以生成一个新的 Middleware 声明的代码，然后可以自定义它以根据需要修改 HTTP 请求和响应。

> **了解更多**
> 在 [HTTP — 中间件](../http/middleware.md) 部分阅读更多关于中间件的信息。

```terminal
php app.php create:middleware <name>
```

将生成 `<Name>` 类。

#### 配置

在我们的示例中，我们使用了以下声明配置：

```php
Spiral\Scaffolder\Declaration\MiddlewareDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web\Middleware',
    'postfix' => 'Middleware',
],
```

#### 示例

```terminal
php app.php create:middleware Logger
```

输出是：

```php app/src/Endpoint/Web/Middleware/LoggerMiddleware.php
declare(strict_types=1);

namespace App\Endpoint\Web\Middleware;

class LoggerMiddleware implements \Psr\Http\Server\MiddlewareInterface
{
    public function process(
        \Psr\Http\Message\ServerRequestInterface $request,
        \Psr\Http\Server\RequestHandlerInterface $handler,
    ): \Psr\Http\Message\ResponseInterface
    {
        return $handler->handle($request);
    }
}
```

### Job Handler（任务处理器）

此命令创建一个 Job Handler 类。Job Handler 处理用于处理排队任务的作业。使用此命令，你可以生成一个新的 Job Handler 声明的代码，然后可以自定义它以根据需要处理作业。

> **了解更多**
> 在 [队列 — 入门](../queue/configuration.md) 部分阅读更多关于作业和队列的信息。

```terminal
php app.php create:jobHandler <name>
```

将创建 `<Name>Job` 类。

#### 配置

在我们的示例中，我们使用了以下声明配置：

```php
Spiral\Scaffolder\Declaration\JobHandlerDeclaration::TYPE => [
    'namespace' => 'Endpoint\Job',
],
```

#### 示例

```terminal
php app.php create:jobHandler UserRegisteredNotification
```

输出是：

```php app/src/Endpoint/Job/UserRegisteredNotificationJob.php
declare(strict_types=1);

namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

final class UserRegisteredNotificationJob extends JobHandler
{
    public function invoke(string $id, array $payload, array $headers): void
    {
    }
}
```
