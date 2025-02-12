# Cookbook — 长时间启动

Spiral 包含了大量相互无缝协作的组件。 在本文中，我们将向您展示如何使用 REST API、ORM、迁移、请求验证、自定义注解（可选）和域拦截器创建一个演示博客应用程序。

> **注意**
> 组件和方法将仅在基础级别进行介绍。 阅读相应的章节以获取更多信息。 您可以在 [此处](https://github.com/spiral/demo) 找到演示仓库。

## 安装

要开始构建您的简单应用程序，您可以通过运行以下命令轻松安装包含大多数所需组件的默认 `spiral/app` 软件包：

```terminal
composer create-project spiral/app spiral-demo
```

在安装过程中，您将被提示使用 Spiral 安装程序选择各种选项，例如应用程序预设、是否使用 Cycle ORM、使用哪个集合、使用哪个验证器组件等等。 对于本教程，我们建议选择如上所示的选项：

```terminal
✔ 您想安装哪个应用程序预设？ > Web
✔ 创建一个默认的应用程序结构和演示数据？ > 否
✔ 您想使用 SAPI 吗？ > 否
✔ 您需要 Cycle ORM 吗？ > 是
✔ 您想使用哪个集合与 Cycle ORM 一起使用？ > Doctrine 集合
✔ 您想使用哪个验证器组件？ > Spiral Validator
✔ 您想使用 Queue 组件吗？ > 否
✔ 您想使用 Cache 组件吗？ > 否
✔ 您想使用 Mailer 组件吗？ > 否
✔ 您想使用 Storage 组件吗？ > 否
✔ 您想使用哪个模板引擎？ > Stempler
✔ 您需要 Data Grid 吗？ > 是
✔ 您想使用 Event Dispatcher 吗？ > 否
✔ 您需要定时任务调度器吗？ > 否
✔ 您需要 Translator 吗？ > 否
✔ 您需要 Temporal 吗？ > 否
✔ 您需要 RoadRunner Metrics 吗？ > 否
✔ 您需要 Sentry 吗？ > 否
```

## 配置

Spiral 应用程序使用位于 `app/config` 中的配置文件进行配置，您可以使用硬编码值进行配置，或者使用可用的函数 `env` 和 [`directory`](../start/structure.md) 获取值。

`spiral/app` 软件包使用 [DotEnv](../extension/dotenv.md) 扩展，该扩展将从 `.env` 文件加载 ENV 变量。

> **注意**
> 使用 `.rr.yaml` 文件调整应用程序服务器及其插件。

应用程序依赖项定义在 `composer.json` 中，并在 `app/src/Application/Kernel.php` 中作为 Bootloader 激活。 默认构建包含大量预先配置的组件。

### 开发者模式

为了简化应用程序的调整，请以开发者模式重启应用程序服务器。 在此模式下，服务器仅使用一个工作进程，并在每个请求后重新加载它。

```yaml .rr.yaml
http:
  ...
  pool:
    debug: true
```

您还可以通过 `rr` 应用程序的 `-c` 标志创建和使用备用配置文件。

```terminal
./rr serve -c .rr.dev.yaml
```

> **更多信息**
> 阅读有关 RoadRunner 官方文档中工作进程的更多信息 [documentation](https://roadrunner.dev/docs/app-server-cli/2.x/en)。

### 精简启动

在我们的演示应用程序中，我们不需要会话、cookie、CSRF、通过 RoutesBootloader 进行的路由配置。
删除这些组件及其 bootloader。

从 `app/src/Application/Kernel.php` 中删除以下 bootloader：

```php app/src/Application/Kernel.php
// from http
Spiral\Bootloader\Http\CookiesBootloader::class,
Spiral\Bootloader\Http\SessionBootloader::class,
Spiral\Bootloader\Http\CsrfBootloader::class,
Spiral\Bootloader\Http\PaginationBootloader::class,
```

> **注意**
> 在[此处](../framework/bootloaders.md) 阅读有关 Bootloader 的更多信息。

默认情况下，路由规则位于 `app/src/Application/Bootloader/RoutesBootloader.php` 中。 您可以通过多种方式配置路由。 将路由指向操作、控制器、控制器组，设置默认的模式参数、谓词、中间件等。

删除方法 `defineRoutes`。 我们将使用属性添加路由：

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

> **更多信息**
> 在 [HTTP — 路由](../http/routing.md) 章节中阅读有关路由的更多信息。

从 **web** 中间件组中删除 `CookiesMiddleware`、`SessionMiddleware`、`CsrfMiddleware`，对于本教程，我们只需要 `ValidationHandlerMiddleware`：

```diff app/scr/Application/Bootloader/RoutesBootloader.php
class RoutesBootloader extends BaseRoutesBootloader
{
    ...
    
    protected function middlewareGroups(): array
    {
        return [
           'web' => [
-                CookiesMiddleware::class,
-                SessionMiddleware::class,
-                CsrfMiddleware::class,
                 ValidationHandlerMiddleware::class
           ],
        ];
    }
 }
```

删除未使用的导入：

```diff app/scr/Application/Bootloader/RoutesBootloader.php
- use Spiral\Cookies\Middleware\CookiesMiddleware;
- use Spiral\Csrf\Middleware\CsrfMiddleware;
- use Spiral\Router\Loader\Configurator\RoutingConfigurator;
- use Spiral\Session\Middleware\SessionMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
     ...
}
```

> **注意**
> 请注意，由于我们删除了呈现 `app/views/home.dark.php` 所需的依赖项，因此应用程序目前无法工作。

### 数据库连接

我们的应用程序需要一个数据库才能运行。 默认情况下，数据库配置位于 `app/config/database.php` 文件中。 演示应用程序附带一个预先配置的内存 SQLite 数据库。 让我们将配置更改为将数据存储在 **runtime/db.sqlite** 文件中。

```php app/config/database.php
use Cycle\Database\Config;

return [
    'default'   => 'default',
    'databases' => [
        'default' => ['driver' => 'sqlite'],
    ],
    'drivers' => [
-        'sqlite' => new Config\SQLiteDriverConfig(
-            connection: new Config\SQLite\MemoryConnectionConfig(),
-            queryCache: true
-        ),
+        'sqlite' => new Config\SQLiteDriverConfig(
+            connection: new Config\SQLite\FileConnectionConfig(
+                database: directory('runtime') . '/db.sqlite'
+            ),
+            queryCache: true
+        ),
        // ...
    ],
];
```

要将默认数据库更改为 MySQL，请更改配置的 `drivers` 部分，我们可以将数据库名称、用户名、密码和端口存储在 `.env` 文件中，将以下行添加到其中：

```dotenv .env
DB_HOST=localhost
DB_NAME=name
DB_USER=username
DB_PASSWORD=password
DB_PORT=3306
```

> **注意**
> 更改这些值以匹配您的数据库参数。

我们可以使用 `env` 函数访问 ENV 变量：

```php app/config/database.php
use Cycle\Database\Config;

return [
    'default'   => 'default',
    'databases' => [
        'default' => ['driver' => 'mysql'],
    ],
    'drivers'   => [
        'mysql' => new Config\MySQLDriverConfig(
            connection: new Config\MySQL\TcpConnectionConfig(
                database: env('DB_NAME'),
                host: env('DB_HOST'),
                port: (int) env('DB_PORT', 3306),
                user: env('DB_USER'),
                password: env('DB_PASSWORD')
            ),
            queryCache: true
        ),
    ]
];
```

要检查数据库连接是否成功，请运行：

```terminal
cd spiral-demo
php app.php db:list
```

> **更多信息**
> 在 [此处](../basics/orm.md) 阅读有关数据库的更多信息。

### 连接数据库 Seeder

我们将需要一些示例数据用于应用程序。
让我们安装 [Database Seeder](https://github.com/spiral-packages/database-seeder)。

```terminal
composer require spiral-packages/database-seeder --dev
```

将 bootloader `Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader` 添加到 `defineBootloaders` 方法中以激活该软件包：

```php app/src/Application/Kernel.php
use Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    { 
        return [
            // ...
            DatabaseSeederBootloader::class,
            // ...
        ];
    }
}
```

### 创建路由

我们可以在控制器中使用此属性，如下所示：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Router\Annotation\Route;

final class HomeController
{
    #[Route(route: '/', name: 'index', methods: 'GET')]
    public function index(): string
    {
        return 'hello world';
    }
    
    #[Route(route: '/open/<id>', name: 'open', methods: 'GET')] 
    public function open(string $id)
    {
        dump($id);
    }
}
```

> **更多信息**
> 在[此处](../http/routing.md) 阅读有关该扩展的更多信息。

运行 CLI 命令以检查可用的路由列表：

```terminal
php app.php route:list
```

> **注意**
> 使用附加的路由参数来配置中间件、路由组等。

在以下示例中，为了简单起见，我们将坚持使用带有属性的路由。

要刷新路由缓存（当禁用 `DEBUG` 时）：

```terminal
php app.php cache:clean
```

一旦应用程序安装并配置完毕，您就可以通过运行以下命令立即启动服务器并打开您的应用程序：

```terminal
./rr serve
```

您刚刚启动了 [RoadRunner](../start/server.md) 服务器。 相同的服务器可以在生产中使用，使您的开发环境类似于最终设置。 开箱即用，该服务器包括使用 HTTP/2、[GRPC](../grpc/configuration.md)、[队列](../queue/configuration.md)、[WebSockets](../websockets/configuration.md) 等编写可移植应用程序的工具，并且不需要外部代理即可运行。

默认情况下，该应用程序在 `http://127.0.0.1:8080` 上可用。

### 域核心

域核心表示驱动应用程序功能的 fundamental 域逻辑和业务规则。 它构成了代码库的核心方面，并且通常包含软件中最重要的和最复杂的元素。

可以使用路由参数和 `Guard` 属性来修改应用程序的默认行为，以启用 Cycle 实体解析。

以下是一个示例：

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Cycle\Interceptor\CycleInterceptor;
use Spiral\DataGrid\Interceptor\GridInterceptor;
use Spiral\Domain\GuardInterceptor;

/**
 * @link https://spiral.dev/docs/http-interceptors
 */
final class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [CoreInterface::class => [self::class, 'domainCore']];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        GridInterceptor::class,
        GuardInterceptor::class,
    ];
}
```

此 bootloader 默认添加，我们不需要修改它。

> **注意**
> 在[此处](../http/interceptors.md) 阅读有关 Http 拦截器的更多信息。

## 支架数据库

该框架可以使用一组迁移文件配置数据库模式。 要在您的应用程序中配置迁移，请运行：

```terminal
php app.php migrate:init
```

您现在可以使用以下命令观察迁移表结构：

```terminal
php app.php db:list
php app.php db:table migrations
```

您可以手动编写迁移，也可以让 Cycle ORM 为您生成迁移。

> **注意**
> 在[此处](https://cycle-orm.dev/docs/database-migrations) 阅读有关迁移的更多信息。
> 使用 [Scaffolder](../basics/scaffolding.md) 组件手动创建迁移。

### 定义 ORM 实体

演示应用程序附带 [Cycle ORM](https://cycle-orm.dev)。 默认情况下，您可以使用属性来配置您的实体。

让我们使用 Scaffolder 扩展创建 `Post`、`User` 和 `Comment` 实体及其存储库：

```terminal
php app.php create:entity post -f id:primary -f title:string -f content:text -e
php app.php create:entity user -f id:primary -f name:string -e
php app.php create:entity comment -f id:primary -f message:string
```

> **注意**
> 观察在 `app/src/Database` 和 `app/src/Repository` 中生成的类。

Post:

```php app/src/Database/Post.php
namespace App\Domain\Blog\Entity;

use App\Repository\PostRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: PostRepository::class)]
class Post
{
    #[Column(type: 'primary')]
    public int $id;

    #[Column(type: 'string')]
    public string $title;

    #[Column(type: 'text')]
    public string $content;
}
```

我们可以将 `$title` 和 `$content` 属性的定义移动到 `__construct` 方法：

```php app/src/Database/Post.php
use App\Domain\Blog\Repository\PostRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: PostRepository::class)]
class Post
{
    #[Column(type: 'primary')]
    public int $id;

    public function __construct(
        #[Column(type: 'string')]
        public string $title,

        #[Column(type: 'text')]
        public string $content
    ) {
    }
}
```

User:

```php app/src/Database/User.php
use App\Domain\Blog\Repository\UserRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: UserRepository::class)]
class User
{
    #[Column(type: 'primary')]
    public int $id;

    #[Column(type: 'string')]
    public string $name;
}
```

我们可以将 `$name` 属性的定义移动到 `__construct` 方法：

```php app/src/Database/User.php
use App\Domain\Blog\Repository\UserRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: UserRepository::class)]
class User
{
    #[Column(type: 'primary')]
    public int $id;

    public function __construct(
        #[Column(type: 'string')]
        public string $name
    ) {
    }
}
```

Comment:

```php app/src/Database/Comment.php
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity]
class Comment
{
    #[Column(type: 'primary')]
    public int $id;

    #[Column(type: 'string')]
    public string $message;
}
```

我们可以将 `$message` 属性的定义移动到 `__construct` 方法：

```php app/src/Database/Comment.php
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity]
class Comment
{
    #[Column(type: 'primary')]
    public int $id;

    public function __construct(
        #[Column(type: 'string')]
        public string $message
    ) {
    }
}
```

让我们将 `Spiral\Prototype\Annotation\Prototyped` 属性添加到存储库，以便我们可以在开发期间将它们用作用户和帖子属性：

:::: tabs

::: tab PostRepository

```php app/src/Repository/PostRepository.php
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'posts')]
class PostRepository extends Repository
{
}
```

:::

::: tab UserRepository

```php app/src/Repository/UserRepository.php
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'users')]
class UserRepository extends Repository
{
}
```

:::

::::

您可以使用 [Scaffolder 配置](../basics/scaffolding.md)更改默认的目录映射、标头等。

> **注意**
> 在[此处](../basics/orm.md) 阅读有关 Cycle 的更多信息。

运行配置命令以收集所有可用的原型类：

```terminal
php app.php configure
```

### 生成迁移

要生成数据库模式，请运行：

```terminal
php app.php cycle:migrate -v
```

生成的迁移位于 `app/migrations/` 中。 使用以下命令执行它：

```terminal
php app.php migrate -vv
```

您现在可以使用 `db:list` 命令观察生成的表。

### 创建关系

使用 [属性](https://cycle-orm.dev/docs/annotated-relations)定义实体之间的关系。 配置 Post 和 Comment 的 BelongsTo 关系，并在 User 和 Post 之间添加 HasMany 关系。

Post:

```php app/src/Database/Post.php
use App\Repository\PostRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;
use Cycle\Annotated\Annotation\Relation;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;

#[Entity(repository: PostRepository::class)]
class Post
{
    #[Column(type: 'primary')]
    public int $id;

    /**
     * @var Collection|Comment[]
     * @psalm-var Collection<int, Comment>
     */
    #[Relation\HasMany(target: Comment::class)]
    public Collection $comments;

    public function __construct(
        #[Column(type: 'string')]
        public string $title,

        #[Column(type: 'text')]
        public string $content,

        #[Relation\BelongsTo(target: User::class, nullable: false)]
        public User $author
    ) {
        $this->comments = new ArrayCollection();
    }
}
```

Comment:

```php app/src/Database/Comment.php
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;
use Cycle\Annotated\Annotation\Relation;

#[Entity]
class Comment
{
    #[Column(type: 'primary')]
    public int $id;

    public function __construct(
        #[Column(type: 'string')]
        public string $message,

        #[Relation\BelongsTo(target: User::class, nullable: false)]
        public User $author,

        #[Relation\BelongsTo(target: Post::class, nullable: false)]
        public Post $post
    ) {
    }
}
```

再次生成并运行迁移：

```terminal
php app.php cycle:migrate -v
php app.php migrate -vv
```

> **注意**
> 您可以使用 `php app.php cycle:migrate -r` 命令在一个命令中生成并运行迁移。

您可以检查外键是否存在：

```terminal
php app.php db:table comments
```

> **注意**
> 更改任何实体时，不要忘记运行 `php app.php cycle:migrate`。

## 工厂和 Seeder

要生成测试数据，我们需要工厂，它们将描述生成实体的规则。
以及将填充数据库的 Seeder。

我们将它们与应用程序代码分开存储，在 `app/database` 文件夹中。
让我们向 Composer 自动加载添加一个单独的 `Database` 命名空间：

```diff composer.json
--- a/composer.json
+++ b/composer.json
"autoload": {
    "psr-4": {
        "App\\": "app/src",
+       "Database\\": "app/database"
    },
    //
},
```

```terminal
composer dump-autoload
```

### CommentFactory

让我们创建 `CommentFactory` 类，将其从 `Spiral\DatabaseSeeder\Factory\AbstractFactory` 扩展，并实现所需的方法：

```php app/database/Factory/CommentFactory.php
namespace Database\Factory;

use App\Database\Comment;
use App\Database\Post;
use App\Database\User;
use Faker\Generator;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

class CommentFactory extends AbstractFactory
{
    /**
     * 返回完全限定的数据库实体类名
     */
    public function entity(): string
    {
        return Comment::class;
    }

    /**
     * 返回一个实体
     */
    public function makeEntity(array $definition): Comment
    {
        return new Comment($definition['message'], $definition['author'], $definition['post']);
    }

    /**
     * 使用给定作者生成 Comment
     */
    public function withAuthor(User $author): self
    {
        return $this->state(fn(Generator $faker, array $definition) => [
            'author' => $author,
        ]);
    }

    /**
     * 使用给定帖子生成 Comment
     */
    public function withPost(Post $post): self
    {
        return $this->state(fn(Generator $faker, array $definition) => [
            'post' => $post,
        ]);
    }

    /**
     * 返回包含生成规则的数组
     */
    public function definition(): array
    {
        return [
            'message' => $this->faker->sentence(12),
            'author' => UserFactory::new()->makeOne(),
            'post' => PostFactory::new()->makeOne()
        ];
    }
}
```

### PostFactory

让我们创建 `PostFactory` 类：

```php app/database/Factory/PostFactory.php
namespace Database\Factory;

use App\Database\Post;
use App\Database\User;
use Faker\Generator;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

class PostFactory extends AbstractFactory
{
    /**
     * 返回完全限定的数据库实体类名
     */
    public function entity(): string
    {
        return Post::class;
    }

    /**
     * 返回一个实体
     */
    public function makeEntity(array $definition): Post
    {
        return new Post($definition['title'], $definition['content'], $definition['author']);
    }

    /**
     * 使用给定作者生成 Post
     */
    public function withAuthor(User $author): self
    {
        return $this->state(fn(Generator $faker, array $definition) => [
            'author' => $author,
        ]);
    }

    /**
     * 返回包含生成规则的数组
     */
    public function definition(): array
    {
        return [
            'title' => $this->faker->sentence(12),
            'content' => $this->faker->text(900),
            'author' => UserFactory::new()->makeOne()
        ];
    }
}
```

### UserFactory

让我们创建 `UserFactory` 类：

```php app/database/Factory/UserFactory.php
namespace Database\Factory;

use App\Database\User;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

class UserFactory extends AbstractFactory
{
    /**
     * 返回完全限定的数据库实体类名
     */
    public function entity(): string
    {
        return User::class;
    }

    /**
     * 返回一个实体
     */
    public function makeEntity(array $definition): User
    {
        return new User($definition['name']);
    }

    /**
     * 返回包含生成规则的数组
     */
    public function definition(): array
    {
        return [
            'name' => $this->faker->name(),
        ];
    }
}
```

### BlogSeeder

让我们创建 `BlogSeeder` 类：

```php app/database/Seeder/BlogSeeder.php
namespace Database\Seeder;

use Database\Factory\CommentFactory;
use Database\Factory\PostFactory;
use Database\Factory\UserFactory;
use Spiral\DatabaseSeeder\Attribute\Seeder;
use Spiral\DatabaseSeeder\Seeder\AbstractSeeder;

#[Seeder]
class BlogSeeder extends AbstractSeeder
{
    public function run(): \Generator
    {
        $users = UserFactory::new()->times(100)->make();
        yield from $users;

        $posts = [];
        for ($i = 0; $i < 1000; $i++) {
            $posts[] = PostFactory::new()
                ->withAuthor($users[array_rand($users)])
                ->makeOne();
        }
        yield from $posts;

        for ($i = 0; $i < 1000; $i++) {
            yield CommentFactory::new()
                ->withAuthor($users[array_rand($users)])
                ->withPost($posts[array_rand($posts)])
                ->makeOne();
        }
    }
}
```

现在，让我们执行一个控制台命令，该命令将使用测试记录填充数据库：

```terminal
php app.php db:seed
```

## 控制器

创建一组 REST 终端，以通过 API 检索帖子数据。 我们可以从一个简单的控制器 `App\Endpoint\Web\PostController` 开始。 使用支架创建它：

```terminal
php app.php create:controller post -a test -a get -p
```

> **注意**
> 使用选项 `-a` 预生成控制器操作，并使用选项 `-p` 预加载原型扩展。

生成的代码：

```php app/src/Endpoint/Web/PostController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

final class PostController
{
    use PrototypeTrait;

    #[Route(route: 'path', name: 'name')]
    public function test(): ResponseInterface
    {
    }

    #[Route(route: 'path', name: 'name')]
    public function get(): ResponseInterface
    {
    }
}
```

### Test 方法

您可以从您的控制器方法返回各种类型的数据。 这些是有效返回值：

- 字符串
- PSR-7 响应
- 数组（作为 JSON）
- JsonSerializable 对象

> **注意**
> 使用自定义 [domain core](../http/interceptors.md) 执行特定于域的响应转换。 您还可以使用 `$this->response` 帮助程序将数据写入 PSR-7 响应对象。

为了演示目的，返回 `array`，`status` 键将被视为响应状态。

```php
// ...
#[Route(route: '/api/test/<id>', name: "post.test", methods: 'GET')]
public function test(string $id): array
{
    return [
        'status' => 200,
        'data'   => [
            'id' => $id
        ]
    ];
}
```

打开 `http://localhost:8080/api/test/123` 以观察结果。

或者，使用 ResponseWrapper 帮助程序：

```php
use Psr\Http\Message\ResponseInterface;
use Spiral\Router\Annotation\Route;

// ...

#[Route(route: '/api/test/<id>', methods: 'GET')] 
public function test(string $id): ResponseInterface
{
    return $this->response->json(
        [
            'data' => [
                'id' => $id
            ]
        ],
        200
    );
}
```

> **注意**
> 我们不会继续使用 test 方法。

### 获取帖子

要获取帖子详细信息，请使用 `PostRepository`，在构造函数、`get` 方法中请求此类依赖关系，或使用原型快捷方式 `posts`。 您可以通过路由参数访问 `id`：

```php app/src/Endpoint/Web/PostController.php
use App\Database\Post;
use Spiral\Http\Exception\ClientException\NotFoundException;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

final class PostController
{
    use PrototypeTrait;

    #[Route(route: '/api/post/<id:\d+>', name: 'post.get', methods: 'GET')]
    public function get(string $id): array
    {
        /** @var Post $post */
        $post = $this->posts->findByPK($id);
        if ($post === null) {
            throw new NotFoundException('post not found');
        }

        return [
            'post' => [
                'id'      => $post->id,
                'author'  => [
                    'id'   => $post->author->id,
                    'name' => $post->author->name
                ],
                'title'   => $post->title,
                'content' => $post->content,
            ]
        ];
    }
}
```

您可以使用连接的 `CycleInterceptor`（确保已连接 `AppBootloader`）替换直接存储库访问并使用 `Post` 作为方法注入：

```php app/src/Endpoint/Web/PostController.php
use App\Database\Post;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

final class PostController
{
    use PrototypeTrait;

    #[Route(route: '/api/post/<post:\d+>', name: 'post.get', methods: 'GET')]  
    public function get(Post $post): array
    {
        return [
            'post' => [
                'id'      => $post->id,
                'author'  => [
                    'id'   => $post->author->id,
                    'name' => $post->author->name
                ],
                'title'   => $post->title,
                'content' => $post->content,
            ]
        ];
    }
}
```

> **注意**
> 考虑使用视图对象将响应数据映射为 `JsonSerializable` 形式。

### 帖子视图映射器

您可以使用任何现有的序列化解决方案（例如 `jms/serializer`）或编写自己的解决方案。 创建一个原型视图对象，以将帖子数据映射为包含评论的 JSON 格式：

```php app/src/Endpoint/Web/View/PostView.php
namespace App\Endpoint\Web\View;

use App\Database\Post;
use Psr\Http\Message\ResponseInterface;
use Spiral\Core\Attribute\Singleton;
use Spiral\Prototype\Annotation\Prototyped;
use Spiral\Prototype\Traits\PrototypeTrait;

#[Singleton]
#[Prototyped(property: 'postView')]
final class PostView
{
    use PrototypeTrait;

    public function map(Post $post): array
    {
        return [
            'post' => [
                'id'      => $post->id,
                'author'  => [
                    'id'   => $post->author->id,
                    'name' => $post->author->name
                ],
                'title'   => $post->title,
                'content' => $post->content,
            ]
        ];
    }

    public function json(Post $post): ResponseInterface
    {
        return $this->response->json($this->map($post), 200);
    }
}
```

> **注意**
> 运行 `php app.php configure` 以生成 IDE 突出显示并注册原型类。

按如下方式修改控制器：

```php app/src/Endpoint/Web/PostController.php
use App\Database\Post;
use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

final class PostController
{
    use PrototypeTrait;

    #[Route(route: '/api/post/<post:\d+>', name: 'post.get', methods: 'GET')]   
    public function get(Post $post): ResponseInterface
    {
        return $this->postView->json($post);
    }
}
```

> **注意**
> 您应该观察到行为没有变化。

### 获取多个帖子

使用直接存储库访问来加载多个帖子。 首先，让我们加载所有可用的帖子及其作者。

在 `PostRepository` 中创建 `findAllWithAuthors` 方法：

```php app/src/Repository/PostRepository.php
use Cycle\ORM\Select;
use Cycle\ORM\Select\Repository;
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'posts')]
class PostRepository extends Repository
{
    public function findAllWithAuthor(): Select
    {
        return $this->select()->load('author');
    }
}
```

在 `PostController` 中创建方法 `list`：

```php app/src/Endpoint/Web/PostController.php
#[Route(route: '/api/post', name: 'post.list', methods: 'GET')]
public function list(): array
{
    $posts = $this->posts->findAllWithAuthor();

    return [
        'posts' => array_map([$this->postView, 'map'], $posts->fetchAll())
    ];
}
```

> **注意**
> 您可以使用 `http://localhost:8080/api/post` 看到所有帖子的 JSON。

### 数据网格

上面提供的方法有其局限性，因为您必须手动分页、过滤和排序结果。
使用 [Data Grid 组件](../component/data-grid.md) 为您处理数据格式化。
安装应用程序时，我们选择安装 Data Grid 组件，因此该组件已在我们的应用程序中安装并配置。

要使用数据网格，我们首先必须指定我们的数据模式，创建 `App\View\PostGrid` 类：

```php app/src/Endpoint/Web/View/PostGrid.php
use App\Database\Post;
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\Equals;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\IntValue;
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'postGrid')]
final class PostGrid extends GridSchema
{
    public function __construct(
        private readonly PostView $view,
    ) {
        $this->addFilter('author', new Equals('author.id', new IntValue()));

        