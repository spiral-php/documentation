# 实时聊天应用程序

近年来，实时聊天应用程序越来越受欢迎，而实现 WebSocket 服务器以支持双向通信已成为构建此类应用程序的关键。然而，创建这种类型的应用程序可能是一项具有挑战性的任务。

幸运的是，现在有新的框架和工具可以更容易地设置 WebSocket 服务器。在本教程中，我们将演示如何使用 Spiral 框架、RoadRunner 和 Centrifugo 创建一个实时聊天应用程序，该应用程序具有身份验证和双向通信功能。

Spiral 框架提供了一系列无缝集成的组件，这使其成为构建复杂应用程序的理想选择。在本教程中，我们将指导您使用 Spiral 框架、Centrifugo、RoadRunner 和 ORM 创建一个简单的实时聊天应用程序。

> **注意**
> 本教程涵盖了创建聊天应用程序中使用的组件和方法的基础知识。欲了解更多详细信息，我们建议您参考相关章节。此外，一个演示仓库可在[此处](https://github.com/spiral/simple-chat)获取。

## 安装

### Spiral 应用程序

要开始构建您的实时聊天应用程序，您可以通过运行以下命令轻松地安装包含大多数所需组件的默认 `spiral/app` 软件包：

```terminal
composer create-project spiral/app realtime-chat
```

在安装过程中，您将被提示使用 Spiral 安装程序选择各种选项，例如应用程序预设、是否使用 Cycle ORM、使用哪些集合、使用哪个验证器组件等等。对于本教程，我们建议选择上面显示的选项：

```terminal
✔ 您想安装哪个应用程序预设？ > Web
✔ 创建默认的应用程序结构和演示数据？ > 否
✔ 您想使用 SAPI 吗？ > 否
✔ 您需要 Cycle ORM 吗？ > 是
✔ 您想与 Cycle ORM 使用哪些集合？ > Doctrine Collections
✔ 您想使用哪个验证器组件？ > Spiral Validator
✔ 您想使用队列组件吗？ > 否
✔ 您想使用缓存组件吗？ > 否
✔ 您想使用邮件组件吗？ > 否
✔ 您想使用存储组件吗？ > 否
✔ 您想使用哪个模板引擎？ > Stempler
✔ 您想使用事件调度器吗？ > 否
✔ 您需要 cron 作业调度程序吗？ > 否
✔ 您需要 Temporal 吗？ > 否
✔ 您需要 RoadRunner Metrics 吗？ > 否
✔ 您需要 Sentry 吗？ > 否
```

安装完成后，您可以通过运行以下命令立即启动服务器并打开您的应用程序：

```terminal
cd realtime-chat

./rr serve
```

您的应用程序将默认在 http://127.0.0.1:8080 上可用。

### Centrifugo

Centrifugo 是一个强大的实时消息传递服务器。为了安装它，我们准备了一个简单的 bash 脚本，它会下载最新版本的 Centrifugo 二进制文件并将其安装在您应用程序的 `bin` 目录中。您可以使用以下命令运行该脚本：

```bash
wget --timeout=10 https://github.com/centrifugal/centrifugo/releases/download/v4.1.2/centrifugo_4.1.2_linux_amd64.tar.gz
mkdir -p bin
tar xvfz centrifugo_4.1.2_linux_amd64.tar.gz centrifugo
rm -rf centrifugo_4.1.2_linux_amd64.tar.gz
mv centrifugo bin/
chmod +x ./bin/centrifugo
```

安装 Centrifugo 后，下一步是在您项目的根目录中创建一个 `centrifugo.json` 配置文件，其中将包含必要的配置详细信息：

```json centrifugo.json
{
  "allowed_origins": [
    "*"
  ],
  "proxy_connect": true,
  "address": "127.0.0.1",
  "port": 8081,
  "grpc_api": true,
  "grpc_api_address": "127.0.0.1",
  "grpc_api_port": 10000,
  "proxy_connect_endpoint": "grpc://127.0.0.1:10001",
  "proxy_connect_timeout": "10s",
  "proxy_rpc_endpoint": "grpc://127.0.0.1:10001",
  "proxy_rpc_timeout": "10s"
}
```

为了启用 Centrifugo 和 RoadRunner 之间的通信，您需要在 `.rr.yaml` 文件中配置 RoadRunner，方法是指定 gRPC 服务器的详细信息及其与 Centrifugo 的连接。这允许 RoadRunner 处理 Centrifugo 事件并将其发送到您的应用程序，反之亦然。

```yaml .rr.yaml
#...

service:
  # 创建一个将运行 Centrifugo 服务器的新服务
  cetrifugo:
    service_name_in_log: true
    remain_after_exit: true
    restart_sec: 1
    command: "./bin/centrifugo --config=centrifugo.json"

centrifuge:
  proxy_address: tcp://127.0.0.1:10001
  grpc_api_address: tcp://127.0.0.1:10000
  pool:
    reset_timeout: 10
    num_workers: 5
```

在这些配置到位后，您的应用程序将能够将事件发送到 Centrifugo，而 Centrifugo 将能够将事件发送到您的应用程序，从而实现它们之间的双向通信。

## 配置

Spiral 应用程序的配置是通过位于 `app/config` 目录中的配置文件完成的。您可以使用这些文件中的预定义值，也可以使用 `env` 和 `directory` 函数以编程方式获取这些值。

应用程序依赖项在 `composer.json` 文件中定义，并且它们在 `app/src/Application/Kernel.php` 文件中作为启动器激活。

### 启动器

为了优化我们的应用程序并使其更轻量级，我们需要添加所需的启动器并删除一些默认的启动器。让我们在 `app/src/Application/Kernel.php` 文件中做一些更改：

```diff app/src/Application/Kernel.php
// ...

// RoadRunner
RoadRunnerBridge\LoggerBootloader::class,
RoadRunnerBridge\HttpBootloader::class,
+ RoadRunnerBridge\CentrifugoBootloader::class,

// ...

// Security and validation
Framework\Security\EncrypterBootloader::class,
Framework\Security\FiltersBootloader::class,
- Framework\Security\GuardBootloader::class,

// ...

Framework\Http\CsrfBootloader::class,
- Framework\Http\PaginationBootloader::class,

// ...

// ORM
CycleBridge\SchemaBootloader::class,
CycleBridge\CycleOrmBootloader::class,
CycleBridge\AnnotatedBootloader::class,
+ CycleBridge\AuthTokensBootloader::class,

// Views and view translation
ViewsBootloader::class,
- TranslatedCacheBootloader::class,

// ...

// Fast code prototyping
PrototypeBootloader::class,
+ CycleBridge\PrototypeBootloader::class,
```

> **注意**
> 在[此处](../framework/bootloaders.md)阅读更多关于启动器的信息。

默认情况下，路由规则位于 `app/src/Application/Bootloader/RoutesBootloader.php` 中。您有许多关于如何配置路由的选项。将路由指向动作、控制器、控制器组，设置默认的模式参数、动词、中间件等。

删除方法 `defineRoutes`。我们将使用属性添加路由：

```diff app/scr/Application/Bootloader/RoutesBootloader.php
class RoutesBootloader extends BaseRoutesBootloader
{
    ...

-    protected function defineRoutes(RoutingConfigurator $routes): void
-    {
-        ...
-    }
 }
```

> **另请参阅**
> 在[HTTP — 路由](../http/routing.md)部分阅读更多关于路由的信息。

由于在此示例中我们不会使用 REST API，因此让我们删除 `api` 中间件组并添加 `Spiral\Filter\ValidationHandlerMiddleware` 来处理验证错误。在 `app/src/Application/Bootloader/RoutesBootloader.php` 文件中，更新 `middlewareGroups` 方法，如下所示：

```diff app/src/Application/Bootloader/RoutesBootloader.php
class RoutesBootloader extends BaseRoutesBootloader
{
    ...

    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                 ...
+                \Spiral\Filter\ValidationHandlerMiddleware::class,
+                \Spiral\Auth\Middleware\AuthMiddleware::class,
            ],
-           'api' => [
-                ...
-           ],
        ];
    }
 }
```

> **注意**
> 提醒一下，当对代码进行更改时，删除所有未使用的导入非常重要。这些可能会使代码混乱，并使其更难阅读和维护。

### 广播

为了在应用程序中启用广播功能，必须配置 `broadcasting.php` 配置文件。此配置将允许应用程序将事件传输到 Centrifugo 服务器。

```php app/config/broadcasting.php
return [
    'connections' => [
        'centrifugo' => [
            'driver' => 'centrifugo',
        ],
    ],
];
```

此外，有必要将 `BROADCAST_CONNECTION` 环境变量设置为新创建的连接：

```dotenv .env
# Broadcast
BROADCAST_CONNECTION=centrifugo
```

按照这些步骤将使应用程序能够将事件广播到 Centrifugo 服务器。

> **注意**
> `centrifugo` 驱动程序由 `spiral/roadrunner-bridge` 软件包提供。

### 数据库连接

为了使我们的应用程序正常运行，必须建立数据库连接。数据库配置文件可以在 `app/config/database.php` 文件中找到。对于此特定应用程序，我们将使用 PostgreSQL 数据库。

```diff app/config/database.php
use Cycle\Database\Config;

return [
    'logger' => [
        'default' => env('DB_LOGGER_DRIVER'),
        'drivers' => [
            // 'runtime' => 'stdout'
        ],
    ],

+    'default' => env('DB_CONNECTION', 'default'),
    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],
    'drivers' => [
-        'runtime' => new Config\SQLiteDriverConfig(
-            connection: new Config\SQLite\FileConnectionConfig(
-                database: directory('runtime') . '/db.sqlite'
-            ),
-            queryCache: true
-        ),
+        'runtime' => new Config\PostgresDriverConfig(
+            connection: new Config\Postgres\TcpConnectionConfig(
+                database: env('DB_DATABASE', 'homestead'),
+                user: env('DB_USERNAME', 'homestead'),
+                password: env('DB_PASSWORD', 'secret'),
+                port: (int) env('DB_PORT', 5432),
+            ),
+            schema: 'public',
+            queryCache: true,
+            options: [
+                'withDatetimeMicroseconds' => true,
+                'logQueryParameters' => env('DB_LOG_QUERY_PARAMETERS', false),
+            ],
+        ),
    ],
];
```

我们可以在 `.env` 文件中存储数据库名称、用户名、密码和端口，将以下行添加到其中：

```dotenv .env
DB_HOST=localhost
DB_NAME=homestead
DB_USER=homestead
DB_PASSWORD=secret
DB_PORT=5432
```

要验证数据库连接是否已成功建立，应执行以下命令：

```terminal
php app.php db:list
```

> **另请参阅**
> 有关数据库配置的更多信息，请参阅
> [数据库配置](../database/configuration.md)文档。

### 连接数据库 Seeder

我们将需要一些应用程序的示例数据。让我们安装[数据库 Seeder](../testing/database.md)。

```terminal
composer require spiral-packages/database-seeder --dev
```

要激活该软件包，必须将 `Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader` 启动器添加到
`LOAD` 部分：

```diff app/src/Application/Kernel.php
         PrototypeBootloader::class,
+        \Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader::class,
```

完成这些步骤后，将启用数据库 Seeder 软件包，并可用于为应用程序提供示例数据。

## 搭建数据库

该框架提供了使用迁移文件集合配置数据库模式的功能。要在您的应用程序中启动迁移配置过程，请执行以下命令：

```terminal
php app.php migrate:init
```

执行上一个命令后，您可以使用以下命令观察迁移表的结构：

```terminal
php app.php db:list
php app.php db:table migrations
```

初始化迁移配置后，您可以手动编写迁移文件或允许 Cycle ORM 为您生成它们。

### 定义 ORM 实体

让我们使用[脚手架](../basics/scaffolding.md)组件创建 `Thread`、`Message` 和 `User` 实体及其存储库：

```terminal
php app.php create:entity thread -f id:primary -f name:string -e
php app.php create:entity message -f id:primary -f message:string -e
php app.php create:entity user -f id:primary -f username:string -f password:string -e
```

> **注意**
> 成功执行上一个命令后，生成的类可能位于 `app/src/Database` 和 `app/src/Repository` 目录中。

### Thread 实体

创建 `Thread` 实体后，它看起来像这样：

```php app/src/Database/Thread.php
namespace App\Database;

use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: '\App\Repository\ThreadRepository')]
class Thread
{
    #[Column(type: 'primary')]
    public int $id;

    #[Column(type: 'string')]
    public string $name;
}
```

让我们让它井然有序：

```php app/src/Database/Thread.php
namespace App\Database;

use App\Repository\ThreadRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: ThreadRepository::class)]
class Thread implements \JsonSerializable
{
    #[Column(type: 'primary')]
    private int $id;

    public function __construct(
        #[Column(type: "string")]
        private string $name,
    ) {
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function jsonSerialize(): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
        ];
    }
}
```

可以类似地修改生成的 `Message` 和 `User` 实体及其存储库，以确保它们的属性和功能与应用程序要求保持一致。

### Message 实体

```php app/src/Database/Message.php
namespace App\Database;

use App\Repository\MessageRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;
use Cycle\Annotated\Annotation\Relation\BelongsTo;

#[Entity(repository: MessageRepository::class)]
class Message implements \JsonSerializable
{
    #[Column(type: 'primary')]
    private int $id;

    public function __construct(
        #[BelongsTo(target: Thread::class)]
        private Thread $thread,
        #[BelongsTo(target: User::class)]
        private User $user,
        #[Column(type: "text")]
        private string $text,
    ) {
    }

    public function jsonSerialize(): array
    {
        return [
            'id' => $this->id,
            'user' => $this->user,
            'text' => $this->text,
        ];
    }
}
```

#### Message 存储库

```php app/src/Repository/MessageRepository.php
namespace App\Repository;

use App\Database\Message;
use Cycle\ORM\Select\Repository;

final class MessageRepository extends Repository
{
    /**
     * @return Message[]
     */
    public function findAllByThread(int $threadId): array
    {
        return $this->findAll([
            'thread_id' => $threadId,
        ], [
            'id' => 'ASC',
        ]);
    }
}
```

### User 实体

```php app/src/Database/User.php
namespace App\Database;

use App\Repository\UserRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;
use Cycle\Annotated\Annotation\Table\Index;

#[Entity(repository: UserRepository::class)]
#[Index(columns: ['username'], unique: true)]
class User implements \JsonSerializable
{
    #[Column(type: 'primary')]
    private int $id;

    public function __construct(
        #[Column(type: "string")]
        private string $username,
        #[Column(type: "string")]
        private string $password,
    ) {
    }

    public function getId(): int
    {
        return $this->id;
    }

    public function getPassword(): string
    {
        return $this->password;
    }

    public function jsonSerialize(): array
    {
        return [
            'id' => $this->id,
            'username' => $this->username,
        ];
    }
}
```

#### User 存储库

```php app/src/Repository/UserRepository.php
namespace App\Repository;

use App\Database\User;
use Cycle\ORM\Select\Repository;

final class UserRepository extends Repository
{
    public function findByUsername(string $username): ?User
    {
        return $this->findOne(['username' => $username]);
    }
}
```

> **注意**
> 在[此处](../basics/orm.md)阅读更多关于 Cycle 的信息。

运行配置命令以收集所有可用的原型类：

```terminal
php app.php configure
```

### 生成迁移

要生成数据库模式，请使用以下命令：

```terminal
php app.php cycle:migrate -v
```

生成的迁移可以在 `app/migrations/` 目录中找到。您可以使用以下命令执行迁移：

```terminal
php app.php migrate -vv
```

您可以使用 `db:list` 命令来观察生成的表。

```terminal
php app.php db:list
```

## 工厂和 Seeder

为了生成测试数据，我们需要描述生成实体规则的工厂和填充数据库的 Seeder。为了保持与应用程序代码的分离，这些工厂和 Seeder 应存储在一个名为 `app/database` 的单独文件夹中。

让我们将一个单独的 `Database` 命名空间添加到 Composer 自动加载，您可以按如下方式更新 `composer.json` 文件：

```diff composer.json
--- a/composer.json
+++ b/composer.json
"autoload-dev": {
    "psr-4": {
        "Tests\\": "tests",
+       "Database\\": "app/database"
    },
    //
},
```

更新 `composer.json` 文件后，运行以下命令以更新自动加载器：

```terminal
composer dump-autoload
```

### UserFactory

下一步是创建 `UserFactory` 类，它将负责生成用户实体。为了实现这一点，扩展 `Spiral\DatabaseSeeder\Factory` 提供的 `AbstractFactory` 类，并实现所需的方法。

要创建它，请运行以下命令：

```teriminal
php app.php create:factory UserFactory
```

创建后，修改 `app/database/Factory/UserFactory.php` 文件的内容，使其看起来像这样：

```php app/database/Factory/UserFactory.php
namespace Database\Factory;

use App\Database\User;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

final class UserFactory extends AbstractFactory
{
    public function entity(): string
    {
        return User::class;
    }

    public function makeEntity(array $definition): User
    {
        return new User(
            username: $definition['username'],
            password: $definition['password'],
        );
    }

    public function definition(): array
    {
        return [
            'username' => $this->faker->userName(),
            'password' => \password_hash('secret', \PASSWORD_BCRYPT),
        ];
    }
}
```

### ThreadFactory

要生成线程的测试数据，请创建 `ThreadFactory` 类。

要创建它，请运行以下命令：

```teriminal
php app.php create:factory ThreadFactory
```

创建后，修改 `app/database/Factory/ThreadFactory.php` 文件的内容，使其看起来像这样：

```php app/database/Factory/ThreadFactory.php
namespace Database\Factory;

use App\Database\Thread;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

final class ThreadFactory extends AbstractFactory
{
    public function entity(): string
    {
        return Thread::class;
    }

    public function makeEntity(array $definition): Thread
    {
        return new Thread(
            name: $definition['name']
        );
    }

    public function definition(): array
    {
        return [
            'name' => $this->faker->sentence,
        ];
    }
}
```

### MessageFactory

要生成消息的测试数据，请创建 `MessageFactory` 类。

要创建它，请运行以下命令：

```teriminal
php app.php create:factory MessageFactory
```

创建后，修改 `app/database/Factory/MessageFactory.php` 文件的内容，使其看起来像这样：

```php app/database/Factory/MessageFactory.php
namespace Database\Factory;

use App\Database\Message;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

final class MessageFactory extends AbstractFactory
{
    public function makeEntity(array $definition): object
    {
        return new Message(
            $definition['thread'],
            $definition['user'],
            $definition['text'],
        );
    }

    public function entity(): string
    {
        return Message::class;
    }

    public function definition(): array
    {
        return [
            'thread' => ThreadFactory::new()->make(),
            'user' => UserFactory::new()->make(),
            'text' => $this->faker->paragraph,
        ];
    }
}
```

为了使用测试数据填充数据库，请创建使用先前创建的工厂的 Seeder。

### UserTableSeeder

要创建 `UserTableSeeder` 类，请运行以下命令：

```teriminal
php app.php create:seeder UserTableSeeder
```

创建 `UserTableSeeder` 类后，修改 `app/database/Seeder/UserTableSeeder.php` 文件的内容，使其看起来像这样：

```php app/database/Seeder/UserTableSeeder.php
namespace Database\Seeder;

use Database\Factory\UserFactory;
use Spiral\DatabaseSeeder\Seeder\AbstractSeeder;

final class UserTableSeeder extends AbstractSeeder
{
    public function run(): \Generator
    {
        yield UserFactory::new(['username' => 'john'])->makeOne();
        yield UserFactory::new(['username' => 'bill'])->makeOne();
    }
}
```

> **注意**
> 我们将只创建 2 个用户：`john` 和 `bill`，如果您需要更多用户，您可以以相同的方式创建。

### ThreadTableSeeder

要创建 `ThreadTableSeeder` 类，请运行以下命令：

```teriminal
php app.php create:seeder ThreadTableSeeder
```

创建 `ThreadTableSeeder` 类后，修改 `app/database/Seeder/ThreadTableSeeder.php` 文件的内容，使其看起来像这样：

```php app/database/Seeder/ThreadTableSeeder.php
namespace Database\Seeder;

use Database\Factory\ThreadFactory;
use Spiral\DatabaseSeeder\Seeder\AbstractSeeder;

final class ThreadTableSeeder extends AbstractSeeder
{
    public function run(): \Generator
    {
        yield ThreadFactory::new(['name' => 'First thread'])->makeOne();
    }
}
```

> **注意**
> 我们将只创建一个线程。对于我们的例子来说，这已经足够了。

现在让我们执行一个控制台命令，它将使用测试记录填充数据库：

```terminal
php app.php db:seed
```

## 控制器

### 登录控制器

首先，我们需要创建一个控制器，该控制器将在我们的聊天应用程序中对用户进行身份验证。

让我们使用脚手架创建它：

```terminal
php app.php create:controller login -a loginForm -a login -p
```

> **注意**
> 使用选项 `-a` 预先生成控制器操作，并使用选项 `-p` 预先加载原型扩展。

生成的代码：

```php app/src/Endpoint/Web/LoginController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

class LoginController
{
    use PrototypeTrait;

    #[Route(route: 'path', name: 'name')]
    public function loginForm(): ResponseInterface
    {
    }

    #[Route(route: 'path', name: 'name')]
    public function login(): ResponseInterface
    {
    }
}
```

#### 登录表单

为了呈现登录表单，我们将使用 `Stempler` 模板引擎。

这是登录表单的代码。

```php app/src/Endpoint/Web/LoginController.php
use Psr\Http\Message\ServerRequestInterface;

final class LoginController
{
    // ...

    #[Route('/login', methods: ['GET'])]
    public function loginForm(ServerRequestInterface $request): ResponseInterface|string
    {
        return $this->views->render('login', [
            'csrf' => $request->getAttribute('csrfToken'),
            'errors' => [],
        ]);
    }
}
```

我们使用 csrf 令牌来保护登录表单免受 CSRF 攻击。该令牌由 `Spiral\Csrf\Middleware\CsrfMiddleware` 中间件生成并存储在请求属性中。

让我们为登录表单创建一个视图模板 `app/views/login.dark.php`：

```html app/views/login.dark.php

<html>
<head>
    <title>Login</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body>
<div class="min-h-screen bg-gray-100 flex flex-col justify-center sm:py-12">
    <div class="p-10 xs:p-0 mx-auto md:w-full md:max-w-md">
        <h1 class="font-bold text-center text-2xl mb-5">Your Logo</h1>
        <form action="/login" method="POST" class="bg-white shadow w-full rounded-lg divide-y divide-gray-200">
            <input type="hidden" name="csrf-token" value="{{ $csrf }}"/>

            @foreach ($errors ?? [] as $error)
            <div class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded relative" role="alert">
                <strong class="font-bold">Error!</strong>
                <span class="block sm:inline">{{ $error }}</span>
            </div>
            @endforeach

            <div class="px-5 py-7">
                <label class="font-semibold text-sm text-gray-600 pb-1 block">Username</label>
                <input type="text" class="border rounded-lg px-3 py-2 mt-1 mb-5 text-sm w-full" name="username"/>

                <label class="font-semibold text-sm text-gray-600 pb-1 block">Password</label>
                <input type="password" name="password" class="border rounded-lg px-3 py-2 mt-1 mb-5 text-sm w-full"/>

                <button type="submit"
                        class="bg-blue-500 hover:bg-blue-600 text-white w-full py-2.5 rounded-lg text-sm text-center inline-block">
                    <span class="inline-block mr-2">Login</span>
                </button>
            </div>
        </form>
    </div>
</div>
</body>
</html>
```

#### 登录处理程序

现在让我们创建一个登录处理程序。它将对用户进行身份验证并将他们重定向到聊天页面。

我们将使用 Cycle ORM 存储身份验证令牌，因此我们需要将 `AUTH_TOKEN_STORAGE` 环境变量设置为 `cycle`：

```dotenv .env
AUTH_TOKEN_STORAGE=cycle
```

现在我们需要创建一个请求过滤器，它将验证登录表单数据。

```php app/src/Endpoint/Web/Filter/LoginRequest.php
namespace App\Entrypoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

final class LoginRequest extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $username;
    #[Post]
    public string $password;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => ['notEmpty', 'string'],
            'password' => ['notEmpty', 'string'],
        ]);
    }
}
```

以及 `InvalidCredentialsException` 异常，如果用户凭据无效，则将抛出该异常：

```php app/src/Application/Exception/InvalidCredentialsException.php
namespace App\Application\Exception;

final class InvalidCredentialsException extends \Exception
{

}
```

现在我们可以实现登录操作：

```php app/src/Endpoint/Web/LoginController.php
use App\Application\Exception\InvalidCredentialsException;
use App\Entrypoint\Web\Filter\LoginRequest;

final class LoginController
{
    // ...

    #[Route('/login', methods: ['POST'])]
    public function login(LoginRequest $filter): ResponseInterface
    {
        $user = $this->users->findByUsername($filter->username);

        if (!$user || !\password_verify($filter->password, $user->getPassword())) {
            throw new InvalidCredentialsException('Invalid username or password!');
        }

        $token = $this->authTokens->create($user->jsonSerialize());
        $this->auth->start($token);

        return $this->response->redirect('/');
    }
}
```

> **注意**
> 最好在一个特殊的服务中验证用户密码。但是对于我们的示例来说，这已经足够了。

用户通过身份验证后，我们使用包含用户数据的有效负载创建一个新的身份验证令牌。

```php
[
    'id' => ...,
    'username' => ...,
]
```

> **注意**
> 您可以在令牌有效负载中存储任何数据。基本上，令牌应该包含您需要识别用户的所有数据。在大多数情况下，`id` 或 `username` 就足够了。

#### 处理无效凭据异常

现在我们需要处理 `InvalidCredentialsException` 异常并在登录表单上显示错误消息。

在我们的应用程序中，我们将错误存储在会话中。

> **警告**
> 会话不是存储错误以便在请求之间共享的最佳位置。但是对于我们的示例来说，这已经足够了。

让我们创建一个新的服务，它将把错误存储在会话中：

```php app/src/Endpoint/Web/SessionErrors.php
namespace App\Entrypoint\Web;

use Spiral\Prototype\Annotation\Prototyped;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Session\SessionSectionInterface;

#[Prototyped(property: 'errors')]
final class SessionErrors
{
    use PrototypeTrait;

    public function clear(): void
    {
        $this->session()->clear();
    }

    /**
     * @return array<non-empty-string, non-empty-string[]>
     */
    public function getErrors(): array
    {
        $errors = $this->session()->getAll();

        // clear errors after reading
        $this->clear();

        return $errors;
    }

    /**
     * @param non-empty-string $key
     * @param non-empty-string $error
     */
    public function addError(string $key, string $error): void
    {
        $this->session()->set($key, $error);
    }

    private function session(): SessionSectionInterface
    {
        return $this->session->getSection('errors');
    }
}
```

并运行以下命令将服务注册为原型：

```bash
php app.php prototype:dump
```

好的，现在我们可以使用我们的服务作为原型，使用 `Spiral\Prototype\Traits\PrototypeTrait` 以及属性名称 `errors`。

让我们创建一个新的中间件来处理异常：

```php app/src/Endpoint/Web/Middleware/HandleInvalidCredentialsMiddleware.php
namespace App\Entrypoint\Web\Middleware;

use App\Application\Exception\InvalidCredentialsException;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

final class HandleInvalidCredentialsMiddleware implements MiddlewareInterface
{
    use PrototypeTrait;

    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        try {
            $response = $handler->handle($request);

            // Flush errors after successful request
            $this->errors->clear();

            return $response;
        } catch (InvalidCredentialsException $e) {
            // Add error to the session and redirect to the login form
            $this->errors->addError('username', $e->getMessage());

            return $this->response->redirect('/login');
        }
    }
}
```

并在 `RoutesBootloader` 中注册中间件：

```diff app/src/Application/Bootloader/RoutesBootloader.php
final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                // ...
+               \App\Entrypoint\Web\Middleware\HandleInvalidCredentialsMiddleware::class,
            ],
        ];
    }
}
```

为了在登录表单上显示错误，我们需要从 `SessionErrors` 服务中获取它们并传递给视图：

```diff app/src/Endpoint/Web/LoginController.php
final class LoginController
{
    // ...

    #[Route('/login', methods: ['GET'])]
    public function loginForm(ServerRequestInterface $request): ResponseInterface|string
    {
        return $this->views->render('login', [
            'csrf' => $request->getAttribute('csrfToken'),
+           'errors'