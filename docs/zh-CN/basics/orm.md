# 基础 — 数据库和 ORM

为了在你的应用中使用 ORM 和数据库功能，Spiral 提供了 [spiral/cycle-bridge](https://github.com/spiral/cycle-bridge) 组件。

## 安装

该组件会自动包含在 `spiral/app` 中，也可以通过 Composer 执行以下命令安装到现有项目中：

```terminal
composer require spiral/cycle-bridge
```

成功安装该包后，有必要将 `Spiral\Cycle\Bootloader\BridgeBootloader` 引导程序添加到 Kernel 中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Cycle\Bootloader\BridgeBootloader::class,
        // ...
    ];
}
```

阅读更多关于引导程序的内容，请参考 [Framework — Bootloaders](../framework/bootloaders.md) 章节。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Cycle\Bootloader\BridgeBootloader::class,
    // ...
];
```

阅读更多关于引导程序的内容，请参考 [Framework — Bootloaders](../framework/bootloaders.md) 章节。
:::

::::

或者，为了进行更细粒度的控制，可以排除 `BridgeBootloader`，并选择仅需要的引导程序。

在 Kernel 中，相关的代码如下所示：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
use Spiral\Cycle\Bootloader as CycleBridge;

public function defineBootloaders(): array
{
    return [
        // ...
    
        // 数据库
        CycleBridge\DatabaseBootloader::class,
        CycleBridge\MigrationsBootloader::class,
    
        // 自动在每次请求后关闭数据库连接 (可选)
        // CycleBridge\DisconnectsBootloader::class,
    
        // ORM
        CycleBridge\SchemaBootloader::class,
        CycleBridge\CycleOrmBootloader::class,
        CycleBridge\AnnotatedBootloader::class,
        CycleBridge\CommandBootloader::class,
    
        // 验证 (可选)
        // CycleBridge\ValidationBootloader::class,
    
        // DataGrid (可选)
        // CycleBridge\DataGridBootloader::class,
    
        // 数据库 Token 存储 (可选)
        CycleBridge\AuthTokensBootloader::class,
    
        // 迁移和 Cycle Scaffolder (可选)
        CycleBridge\ScaffolderBootloader::class,
        
        // 原型开发 (可选)
        CycleBridge\PrototypeBootloader::class,
    ];
}
```

阅读更多关于引导程序的内容，请参考 [Framework — Bootloaders](../framework/bootloaders.md) 章节。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
use Spiral\Cycle\Bootloader as CycleBridge;

protected const LOAD = [
    // ...

    // 数据库
    CycleBridge\DatabaseBootloader::class,
    CycleBridge\MigrationsBootloader::class,

    // 自动在每次请求后关闭数据库连接 (可选)
    // CycleBridge\DisconnectsBootloader::class,

    // ORM
    CycleBridge\SchemaBootloader::class,
    CycleBridge\CycleOrmBootloader::class,
    CycleBridge\AnnotatedBootloader::class,
    CycleBridge\CommandBootloader::class,

    // 验证 (可选)
    // CycleBridge\ValidationBootloader::class,

    // DataGrid (可选)
    // CycleBridge\DataGridBootloader::class,

    // 数据库 Token 存储 (可选)
    CycleBridge\AuthTokensBootloader::class,

    // 迁移和 Cycle Scaffolder (可选)
    CycleBridge\ScaffolderBootloader::class,
    
    // 原型开发 (可选)
    CycleBridge\PrototypeBootloader::class,
];
```

阅读更多关于引导程序的内容，请参考 [Framework — Bootloaders](../framework/bootloaders.md) 章节。
:::

::::

#### Disconnects 引导程序

该引导程序旨在自动在每次请求后关闭长时间运行的应用程序中的数据库连接。这是一个可选的引导程序，可以根据特定应用程序的需求进行包含或排除。

## 配置

### 数据库

数据库服务的配置位于 `app/config/database.php` 配置文件中。在此文件中，你可以定义所有数据库连接，并指定应该默认使用哪个连接。此文件中大多数配置选项都由你应用程序的环境变量的值驱动。

这是一个定义数据库连接的示例配置文件：

```php app/config/database.php
use Cycle\Database\Config;

return [
    'logger' => [
        'default' => env('DB_LOGGER_DEFAULT'),
        'drivers' => [
            // 'runtime' => 'stdout'
        ],
    ],

    'default' => env('DB_DEFAULT', 'default'),

    /**
     * Spiral/Database 模块提供支持在单个应用程序中管理多个数据库，使用读/写连接，并使用前缀在单个连接中逻辑地分隔多个数据库。
     *
     * 要注册一个新的数据库，只需在下面的 "databases" 部分中添加一个新数据库。
     */
    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],

    /**
     * 每个数据库实例必须有一个关联的连接对象。
     * 连接用于提供低级功能并封装不同的数据库驱动程序。要注册一个新的连接，你必须指定驱动程序类及其连接选项。
     */
    'drivers' => [
        'runtime' => new Config\MySQLDriverConfig(
            connection: new Config\MySQL\TcpConnectionConfig(
                database: 'homestead',
                host: '127.0.0.1',
                port: 3307,
                user: 'root',
                password: 'secret',
            ),
            queryCache: true
        ),
        // ...
    ],
];
```

> **警告**
> 请注意，将 SQLite 用作数据库后端与多个 worker 一起使用会带来重大挑战和限制。务必了解这些限制，以确保你的应用程序的正常功能和性能。在 [SQLite 限制](./#sqlite-limitations) 部分阅读更多关于这方面的内容。

> **另请参阅**
> 在官方网站上的 [数据库 - 安装和配置](https://cycle-orm.dev/docs/database-configuration) 阅读更多关于数据库配置的信息。

### ORM

Spiral 框架的 ORM 服务的配置位于你应用程序的 `app/config/cycle.php`

```php app/config/cycle.php
use Cycle\ORM\SchemaInterface;

return [
    'schema' => [
        /**
         * true (默认) - 编译后，Schema 将存储在缓存中。
         * 在实体修改后，它不会被更改。使用 `php app.php cycle` 更新 schema。
         *
         * false - 编译后，Schema 不会存储在缓存中。
         * 在实体修改后，它将自动更改。（开发模式）
         */
        'cache' => false,

        /**
         * CycleORM 提供了管理每个 schema 未定义部分的默认设置的能力
         */
        'defaults' => [
            SchemaInterface::MAPPER => \Cycle\ORM\Mapper\Mapper::class,
            SchemaInterface::REPOSITORY => \Cycle\ORM\Select\Repository::class,
            SchemaInterface::SCOPE => null,
            SchemaInterface::TYPECAST_HANDLER => [
                \Cycle\ORM\Parser\Typecast::class
            ],
        ],

        'collections' => [
            'default' => 'array',
            'factories' => [
                'array' => new \Cycle\ORM\Collection\ArrayCollectionFactory(),
                // 'doctrine' => new \Cycle\ORM\Collection\DoctrineCollectionFactory(),
                // 'illuminate' => new \Cycle\ORM\Collection\IlluminateCollectionFactory(),
            ],
        ],

        /**
         * Schema 生成器 (可选)
         * null (默认) - 将使用引导程序中定义的 schema 生成器
         */
        'generators' => null,

        // 'generators' => [
        //        \Cycle\Schema\Generator\ResetTables::class,
        //        \Cycle\Annotated\Embeddings::class,
        //        \Cycle\Annotated\Entities::class,
        //        \Cycle\Annotated\TableInheritance::class,
        //        \Cycle\Annotated\MergeColumns::class,
        //        \Cycle\Schema\Generator\GenerateRelations::class,
        //        \Cycle\Schema\Generator\GenerateModifiers::class,
        //        \Cycle\Schema\Generator\ValidateEntities::class,
        //        \Cycle\Schema\Generator\RenderTables::class,
        //        \Cycle\Schema\Generator\RenderRelations::class,
        //        \Cycle\Schema\Generator\RenderModifiers::class,
        //        \Cycle\Annotated\MergeIndexes::class,
        //        \Cycle\Schema\Generator\GenerateTypecast::class,
        // ],
    ],

    /**
     * 准备所有内部 ORM 服务（映射器，存储库，类型转换器...）
     */
    'warmup' => false,
];
```

## Cycle ORM

CycleORM 是一个强大且灵活的对象关系映射 (ORM) 工具，适用于 PHP，它允许开发人员以面向对象的方式与数据库进行交互。它提供了许多功能，使处理数据变得容易，包括灵活的配置选项、强大的查询构建器以及对 schema 动态映射的支持。

它支持各种流行的关系型数据库，例如 MySQL、MariaDB、PostgresSQL、SQLServer 和 SQLite。

> **注意**
> 完整的文档可在官方网站 [CycleORM](https://cycle-orm.dev/docs) 上找到。

### ORM 实例

你可以通过使用 `Cycle\ORM\ORMInterface` 接口从容器中访问 ORM 实例。

### 存储库

让我们假设我们有一个 `User` 实体

```php
use Cycle\Annotated\Annotation as Cycle;

#[Cycle\Entity(repository: UserRepository::class)]
class User
{
    // ...
}
```

以及一个 `UserRepository` 存储库。

```php 
class UserRepository extends \Cycle\ORM\Select\Repository
{
    public function findByEmail(string $email): ?User
    {
        return $this->findOne(['email' => $email]);
    }
}
```

你可以通过提供实体或角色名称来自己从 ORM 实例请求存储库。

```php
use Cycle\ORM\ORMInterface;
use Cycle\ORM\RepositoryInterface;

class UserService
{   
    private readonly RepositoryInterface $repository;

    public function __construct(
        Cycle\ORM\ORMInterface $orm
    ) {
        $this->repository = $orm->getRepository(User::class);
    }
    
    public function getProfile(string $email): User
    {
        $user = $this->repository->findOne(['email' => $email]);
        // ...
    }
}
```

你也可以从容器中请求存储库。框架使用 [IoC 注入](../container/injectors.md) 将存储库注入到实现 `Cycle\ORM\RepositoryInterface` 的代码中。

```php
class UserService
{
    public function __construct(
        private readonly UserRepository $repository
    ) {
    }
    
    public function getProfile(string $email): User
    {
        $user = $this->repository->findByEmail($email);
        // ...
    }
}
```

当你从容器中请求存储库时，Spiral 将自动从 ORM 请求存储库并将其与正确的实体关联起来。

### 事务

为了持久化实体的更改，你的应用程序服务和控制器将需要 `Cycle\ORM\EntityManagerInterface`。

默认情况下，框架将根据需要从容器中自动创建一个事务。考虑到事务总是在操作 `run` 之后清理，你可以将其作为构造函数参数来请求。

> **另请参阅**
> 你可以在 [CycleORM 文档](https://cycle-orm.dev/docs/advanced-entity-manager) 中阅读更多关于事务的信息。

这是一个使用 `EntityManagerInterface` 的服务的示例：

```php
use Cycle\ORM\EntityManagerInterface;

class UserService
{
    public function __construct(
        private readonly EntityManagerInterface $entityManager
    ) {
    }
    
    public function create(string $name, string $email): User
    {
        $user = new User($name, $email);
        
        $this->entityManager->persist($user);
        $this->entityManager->run();
        
        return $user;
    }
}
```

> **注意：**
> 在使用特定于服务的事务时，请确保在同一方法范围内始终调用 `persist/delete` 和 run 方法。

### 实体验证

Cycle bridge 提供了 `CycleBridge\ValidationBootloader` 引导程序，该程序为 [spiral/validator](../validation/spiral.md) 包注册了额外的检查器。此引导程序包括两个额外的验证规则，这些规则增强了验证器的功能，并允许在应用程序中进行更高效和有效的的数据验证。

#### exists

检查是否存在具有给定角色和主键的实体。

默认情况下，规则将通过主键检查实体是否存在。

```php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Setter;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class StoreUser extends Filter implements HasFilterDefinition
{
    #[Post]
    #[Setter(filter: 'intval')]
    public int $id;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::exists', 
                    \App\Entity\User::class // 实体角色
                ] 
            ]       
        ]);
    }
}
```

你也可以指定字段名称和值，这些字段和值将用于检查实体是否存在。

```php
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class UpdateUser extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $username;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::exists', 
                    \App\Entity\User::class, // 实体角色
                    'username', // 字段名称
                ], 
            ],       
        ]);
    }
}
```

#### unique

检查是否具有给定角色的实体是唯一的。

```php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Setter;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class StoreUser extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $username;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::unique', 
                    \App\Entity\User::class, // 实体角色
                    'username', // 字段名称
                ] 
            ]       
        ]);
    }
}
```

### 实体行为

如果你想在你的应用程序中使用 [cycle/entity-behavior](https://cycle-orm.dev/docs/entity-behaviors-install/) 包，你首先需要安装它：

```bash
composer require cycle/entity-behavior
```

之后，你需要在应用程序容器中将 `Cycle\ORM\Transaction\CommandGeneratorInterface`
与 `\Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator` 绑定：

```php app/src/Application/Bootloader/EntityBehaviorBootloader.php
namespace App\Application\Bootloader;

use Cycle\ORM\Transaction\CommandGeneratorInterface;
use Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator;
use Spiral\Boot\Bootloader\Bootloader;

final class EntityBehaviorBootloader extends Bootloader
{
    protected const BINDINGS = [
        CommandGeneratorInterface::class => \Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator::class,
    ];
}
```

最后，你需要在应用程序内核中注册 `App\Application\Bootloader\EntityBehaviorBootloader`：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\EntityBehaviorBootloader::class,
        // ...
    ];
}
```

阅读更多关于引导程序的内容，请参考 [Framework — Bootloaders](../framework/bootloaders.md) 章节。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\EntityBehaviorBootloader::class,
    // ...
];
```

阅读更多关于引导程序的内容，请参考 [Framework — Bootloaders](../framework/bootloaders.md) 章节。
:::

::::

就这样！现在你可以在你的应用程序中使用实体行为。

### 拦截器

#### Cycle 实体解析

Cycle ORM 集成提供了一个 `Spiral\Cycle\Interceptor\CycleInterceptor` 拦截器，它允许你通过主键在控制器方法中自动解析实体。

> **注意：**
> 在 [HTTP — 拦截器](../http/interceptors.md) 章节阅读更多关于使用拦截器的信息。

激活拦截器：

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Cycle\Interceptor\CycleInterceptor;
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        // ...
    ];
}
```

之后，你可以在控制器方法中使用 cycle 实体注入：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use App\Entity\User;
use Spiral\Router\Annotation\Route;

final class HomeController
{
    #[Route('/users/<user>')]
    public function index(User $user)
    {
        dump($user);
    }
}
```

> **注意：**
> 如果找不到实体，将抛出 404 异常。

### 长时间运行

Cycle ORM 旨在简化在守护进程应用程序（例如在 RoadRunner 或 Swoole 下运行的 PHP worker）中使用该库。ORM 提供了多个选项来避免内存泄漏，这些选项也可以应用于批处理操作。这有助于确保应用程序在执行长时间运行的进程时的稳定性和效率。

该包将在每次请求后自动清理堆。如果你需要手动清理堆，你可以使用以下方法：

```php app/src/Domain/User/Service/UserService.php
use Cycle\ORM\ORMInterface;

class UserService
{
    public function __construct(
        private readonly ORMInterface $orm
    ) {
    }
    
    public function create(string $name, string $email): User
    {
        // 创建一个新用户
        
        $this->orm->getHeap()->clean();
    }
}
```

### 控制台命令

Cycle ORM 集成提供了多个命令，以便于控制。你可以使用以下命令获取任何命令的帮助

```bash
php app.php help cycle...
```

> **注意**
> 确保在 cycle 引导程序之后启用 `Spiral\Cycle\Bootloader\CommandBootloader` 以激活帮助程序命令。

#### 数据库

| 命令                | 描述                                                                                                       |
|---------------------|------------------------------------------------------------------------------------------------------------|
| `db:list [db]`      | 获取可用数据库、它们的表和记录数的列表。<br/>`db` 数据库名称。                                                    |
| `db:table <table>` | 描述特定数据库的表 schema。<br/>`table` 表名称（必需）。<br/>`--database` 源数据库。                                 |

#### ORM 和 Schema

| 命令            | 描述                                                                                              |
|-----------------|---------------------------------------------------------------------------------------------------|
| `cycle`         | 从数据库和注释类更新（初始化）cycle schema。                                                               |
| `cycle:migrate` | 生成 ORM schema 迁移。<br/>`--run` 自动运行生成的迁移。                                                      |
| `cycle:render`  | 渲染可用的 CycleORM schema。<br/>`--no-color` 显示没有颜色的输出。                                                    |

> **注意**
> 你可以使用 `-vv` 标志运行任何 cycle 命令，以查看已修改的表列表。

<hr>

## 数据库

### 访问数据库

你可以在控制器和服务中通过几种方式访问你的数据库：

#### 使用 Database provider

```php app/src/Domain/User/Service/UserService.php
use Cycle\Database\DatabaseProviderInterface;

final class UserService 
{
    public function __construct(
        private readonly DatabaseProviderInterface $dbal
    ) {}
    
    public function store(): void
    {
        // 默认数据库
        dump($this->dbal->database());
    
        // 使用别名 default，它指向主数据库
        dump($this->dbal->database('default'));
    
        // 次要数据库
        dump($this->dbal->database('slave'));
    }
}
```

#### 使用方法和构造函数注入

DBAL 组件完全支持 [IoC 注入](../container/injectors.md)，基于数据库名称及其别名：

```php
use Cycle\Database\DatabaseInterface;

public function store(
    DatabaseInterface $database, 
    DatabaseInterface $primary,
    DatabaseInterface $slave
): void {
    // Database 是 "primary" 的别名
    dump($database === $primary);

    dump($primary);
    dump($slave);
}
```

#### 使用原型

使用 `PrototypeTrait` 访问 `Cycle\Database\DatabaseProviderInterface` 和默认数据库实例：

```php app/src/Domain/User/Service/UserService.php
final class UserService 
{
    use PrototypeTrait;
    
    public function store(): void
    {
        dump($this->dbal);
        dump($this->db); // 默认数据库
    }
}
```

### 运行查询

要运行数据库查询，请使用方法 `query`：

```php
dump(
    $db->query('SELECT * FROM users WHERE id > ?', [
        1
    ])->fetchAll()
);
```

要执行更新或删除语句，请使用备用方法 `execute`：

```php
dump(
    $db->execute('DELETE FROM users WHERE id > ?', [
        1,
    ]) // 影响的行数
);
```

> **注意**
> 在 [此处](https://cycle-orm.dev/docs/database-query-builders) 阅读如何使用查询构建器。

### 记录

Spiral 提供了通过使用 `spiral/logger` 组件来记录数据库查询的功能。该组件使用 Monolog 作为其默认的日志驱动程序。

> **另请参阅**
> 在 [基础知识 — 记录](../basics/logging.md) 章节阅读更多关于记录器的信息。

可以在 `app/config/database.php` 配置文件的 `logger` 部分配置数据库日志驱动程序。

```php app/config/database.php
return [
    'logger' => [
        'default' => null,
        'drivers' => [],
    ],

    // ...
];
```

如果未定义任何日志驱动程序，则将使用具有当前数据库驱动程序名称的 channel。如果使用 SQLite 驱动程序执行查询，则 Monolog channel 名称将自动设置为 `Cycle\Database\Driver\SQLite\SQLiteDriver`。

在这种情况下，你可以为 channel `Cycle\Database\Driver\SQLite\SQLiteDriver` 配置一个 Monolog handler，以便记录所有数据库查询。

这可以通过在 `app/config/monolog.php` 文件中添加以下代码来完成：

```php app/config/monolog.php
return [
    'handlers' => [
        // ...

        \Cycle\Database\Driver\SQLite\SQLiteDriver::class => [
            [
                'class' => 'log.rotate',
                'options' => [
                    'filename' => directory('runtime') . 'logs/db.log',
                    'level' => Logger::DEBUG,
                ],
            ],
        ],
    ],
    
    // ...
];
```

`logger` 的 `drivers` 部分用于指定哪个数据库驱动程序应使用由 key 指定的日志 channel。

让我们考虑以下数据库配置：

```php app/config/database.php
return [
    'logger' => [
        'drivers' => [
            'runtime' => 'console'
        ],
    ],
    
    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],
    
    'drivers' => [
        'runtime' => new Config\SQLiteDriverConfig(...),
        // ...
    ],
];
```

我们可以使用日志配置数组将 `runtime` 数据库驱动程序映射到 `console` 日志 channel。

以及以下 Monolog 配置：

```php app/config/monolog.php
return [
    'handlers' => [
        //...
        'console' => [
            \Monolog\Handler\ErrorLogHandler::class,
        ],
    ],
];
```

使用这些配置，每次你使用 `runtime` 数据库驱动程序时，它的日志都将被发送到 `console` channel。

你还可以将特定的数据库驱动程序指向特定的日志 channel。例如，你可以为你的 SQLite 数据库创建一个单独的日志 channel，为你的 MySQL 数据库创建另一个日志 channel。通过这种方式，你可以单独监视每个数据库的日志并解决每个数据库特有的任何问题。这有助于你快速识别和修复问题，而无需筛选大型、复杂的日志文件。

```php app/config/database.php
return [
    'logger' => [
        'drivers' => [
            \Cycle\Database\Driver\MySQL\MySQLDriver::class => 'db_logs',
            \Cycle\Database\Driver\SQLite\SQLiteDriver::class => 'console'
        ],
    ],
];
```

在这种情况下，每次使用 `SQLiteDriver` 数据库驱动程序时，它都会将日志发送到 `console` 日志 channel。

你还可以将 `default` 键设置为特定的日志 channel。这将用作未在 `drivers` 部分中指定的数据库驱动程序执行的所有查询的默认日志 channel。

```php app/config/database.php
return [
    'logger' => [
        'default' => 'console',
    ],
];
```

通过设置默认日志 channel，它可以确保即使未为特定的数据库驱动程序定义特定的日志 channel，仍然存在用于记录的 channel。

例如，如果开发人员将默认日志 channel 设置为 `console`，则任何没有为其分配特定日志 channel 的数据库驱动程序都将其日志定向到 `console` channel。这为未定义日志 channel 的情况提供了后备，确保捕获所有日志。

### 控制台命令

默认的 Web 和 GRPC 捆绑包包含一组控制台命令，用于查看数据库 schema。

在你的应用程序中激活引导程序 `Spiral\Cycle\Bootloader\CommandBootloader`：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Cycle\Bootloader\CommandBootloader::class,
        // ...
    ];
}
```

阅读更多关于引导程序的内容，请参考 [Framework — Bootloaders](../framework/bootloaders.md) 章节。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Cycle\Bootloader\CommandBootloader::class,
    // ...
];
```

阅读更多关于引导程序的内容，请参考 [Framework — Bootloaders](../framework/bootloaders.md) 章节。
:::

::::

#### 查看可用驱动程序和表

要查看可用数据库、驱动程序和表：

```terminal
php app.php db:list
```

输出：

```output
+------------+------------+---------+---------+-----------+---------+----------------+
|  [32mName (ID): [39m |  [32mDatabase: [39m  |  [32mDriver: [39m |  [32mPrefix: [39m |  [32mStatus: [39m   |  [32mTables: [39m |  [32mCount Records: [39m |
+------------+------------+---------+---------+-----------+---------+----------------+
|  [32mdefault [39m    | runtime.db | SQLite  | ---     | connected | users   |  [33m0 [39m              |
|            |            |         |         |           | posts   |  [33m0 [39m              |
+------------+------------+---------+---------+-----------+---------+----------------+
```

#### 查看表 schema

要查看关于特定表的详细信息：

```terminal
php app.php db:table posts
```

输出：

```output
Columns of default.posts:
+---------+----------------+----------------+-----------+----------------+
| Column: | Database Type: | Abstract Type: | PHP Type: | Default Value: |
+---------+----------------+----------------+-----------+----------------+
| id      | int            | primary        | int       | ---            |
| title   | string (255)   | text           | string    | ---            |
| user_id | int            | integer        | int       | ---            |
+---------+----------------+----------------+-----------+----------------+

Indexes of default.posts:
+-----------------------------------+-------+----------+
| Name:                             | Type: | Columns: |
+-----------------------------------+-------+----------+
| posts_index_user_id_5e32b9642a0ff | INDEX | user_id  |
+-----------------------------------+-------+----------+

Foreign Keys of default.posts:
+------------------+---------+----------------+-----------------+------------+------------+
| Name:            | Column: | Foreign Table: | Foreign Column: | On Delete: | On Update: |
+------------------+---------+----------------+-----------------+------------+------------+
| posts_user_id_fk | user_id | users          | id              | CASCADE    | CASCADE    |
+------------------+---------+----------------+-----------------+------------+------------+
```

<hr>

## 迁移

当你使用 Cycle ORM 并且需要使你的数据库适应应用程序中实体的更改时，`cycle/migrations` 包提供了一种方便的方法来生成和管理你的迁移。此过程会将你当前的数据库 schema 与你的实体更改进行比较，并创建必要的迁移文件。

### 生成迁移

在对你的实体进行了更改或创建了新实体之后，你需要生成反映这些更改的迁移文件。为此，请在你的控制台中运行以下命令：

```terminal
php app.php cycle:migrate
```

新的迁移文件将创建在 app/migrations 目录中。

> **警告**
> 在执行上述命令之前，请确保通过运行 `php app.php migrate` 应用任何现有的迁移。这可确保你的数据库 schema 是最新的。

### 应用迁移

在生成迁移文件之后，需要将它们应用于你的数据库以进行实际更改。

```terminal
php app.php migrate
```

此命令应用最新生成的迁移。当你运行它时，Cycle ORM 会根据你的迁移文件中描述的更改来更新你的数据库 schema。

**Cycle ORM 提供了几个命令来管理不同阶段和不同用途的迁移：**

#### 重放

```terminal
php app.php migrate:replay
```

此命令对于重新运行迁移很有用。它首先回滚（或“down”）迁移，然后再次运行它们（或“up”）。当你想快速测试在你的迁移中所做的更改时，它会很方便。

**选项：**

- `--all`：此选项将重放所有迁移，而不仅仅是最后一个。

#### 回滚

```terminal
php app.php migrate:rollback
```

如果你需要撤消迁移，这将是你将使用的命令。它会反转由迁移所做的更改。默认情况下，它会回滚最后的迁移。

**选项：**

- `--all`：此选项将回滚所有迁移，而不仅仅是最后一个。

#### 状态

```terminal
php app.php migrate:status
```

想看看完成了什么以及正在等待什么？此命令向你显示所有迁移的列表及其状态 - 它们是否已被应用。

**这是一个输出示例：**

```output
+-----------------------------------------------------------+---------------------+---------------------+
| Migration                                                 | Created at          | Executed at         |
+-----------------------------------------------------------+---------------------+---------------------+
| 0_default_create_auth_tokens                              | 2023-09-25 16:45:13 | 2023-10-04 10:46:11 |
| 0_default_create_user_role_create_user_roles_create_users | 2023-09-26 21:22:16 | 2023-10-04 10:46:11 |
| 0_default_change_user_roles_add_read_only                 | 2023-09-27 13:53:19 | 2023-10-04 10:46:11 |
+-----------------------------------------------------------+---------------------+---------------------+
```

#### 初始化

```terminal
php app.php migrate:init
```

在你开始使用迁移之前，迁移组件需要一个地方来记录哪些迁移已运行。此命令通过在你的数据库中创建迁移表来设置该跟踪。

> **注意**
> 当你第一次运行 `php app.php migrate` 时，会自动运行此命令。

### 配置

如果你想自定义 Cycle ORM 处理迁移的方式，你可以在 `app/config/migration.php.` 创建一个配置文件。该文件允许你设置各种选项，例如迁移文件的位置或数据库中迁移表的名称。

**以下是你可以在配置文件中设置的内容：**

- **Directory（目录）**：选择保存迁移文件的位置。
- **Table（表）**：命名跟踪迁移状态的表。
- **Strategy（策略）**：确定如何生成迁移文件。
- **Name Generator（名称生成器）**：指定如何创建迁移文件名。
- **Safe（安全）**：在生产环境中运行迁移期间跳过确认请求。

**这是一个配置文件示例**

```php app/config/migration.php
use Cycle\Schema\Generator\Migrations\Strategy\SingleFileStrategy;
use Cycle\Schema\Generator\Migrations\NameBasedOnChangesGenerator;

return [
    /**
     * 存储迁移文件的目录
     */
    'directory' => directory('app').'migrations/',

    /**
     * 存储关于迁移状态信息的表名（每个数据库）
     */
    'table' => 'migrations',
    
    /**
     * 迁移文件生成器策略
     */
    'strategy' => SingleFileStrategy::class,
    
    /**
     * 迁移文件名生成器
     */
    'nameGenerator' => NameBasedOnChangesGenerator::class,

    /**
     * 当设置为 true 时，在迁移运行期间不会请求确认。
     */
    'safe' => env('APP_ENV