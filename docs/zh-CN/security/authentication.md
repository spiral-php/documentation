# 安全 - 用户认证

该框架包含一组组件，用于通过来自不同来源的临时或永久令牌对用户进行授权，并安全地管理用户上下文。

> **注意**
> 该组件不强制任何特定的用户实体接口，也不将应用程序限制为仅 HTTP 范围（也可能进行 GRPC 认证）。

## 工作原理

![Auth](https://user-images.githubusercontent.com/773481/210746599-cb43c8ad-8021-4c9a-8a9a-45eea55b4c22.png)

认证扩展将为 `Spiral\Auth\AuthContextInterface` 创建一个 IoC 作用域，该作用域指向当前授权的参与者（用户、API 客户端）。参与者使用 `Spiral\Auth\TokenInterface` 从 `Spiral\Auth\ActorProviderInterface` 中获取。

令牌由 `Spiral\Auth\TokenStorageInterface` 管理，并且始终包含负载（例如 `["userID" => $id]`，LDAP 凭据等）。令牌负载用于通过 `Spiral\Auth\ActorProviderInterface` 查找当前应用程序用户。

令牌存储既可以将令牌存储在外部来源（如数据库、Redis 或文件）中，也可以动态解码。该框架开箱即用地包含多个令牌实现，以便更方便地使用。

> **注意**
> 您可以在一个应用程序中使用多个令牌和参与者提供程序。

## 安装和配置

要激活该组件，请添加引导程序 `Spiral\Bootloader\Auth\HttpAuthBootloader`：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Auth\HttpAuthBootloader::class,
        // ...
    ];
}
```

有关引导程序的更多信息，请阅读 [框架 — 引导程序](../framework/bootloaders.md) 部分。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Auth\HttpAuthBootloader::class,
    // ...
];
```

有关引导程序的更多信息，请阅读 [框架 — 引导程序](../framework/bootloaders.md) 部分。
:::

::::

## 认证令牌存储

`Spiral\Auth\TokenStorageInterface` 是 Spiral 框架中的一个接口，它定义了一组用于处理身份验证令牌的存储、检索和删除的标准化方法。 它充当实际存储机制的抽象层，该机制可以是 `session`、`cache`、`database` 等。

### 配置

您可以通过 `.env` 文件中的 `AUTH_TOKEN_STORAGE` 环境变量指定身份验证令牌的默认存储机制。

**例如：**

```.env
AUTH_TOKEN_STORAGE=session
```

或者，在 `app/config/auth.php` 配置文件中设置默认存储机制，如下所示：

```php app/config/auth.php
return [
    'defaultStorage' => env('AUTH_TOKEN_STORAGE', 'session'),
    // ... 其他存储选项
]
```

这允许您指定默认情况下应从容器中检索 `TokenStorageInterface` 的哪个实现。

### 访问默认令牌存储

要从应用程序的服务容器中检索默认令牌存储实例，请使用以下代码：

```php
$storage = $container->get(\Spiral\Auth\TokenStorageInterface::class); 
// 将返回默认的 Session 令牌存储
```

### 令牌存储提供程序

令牌存储提供程序是一项服务，它允许您通过其名称检索特定的令牌存储实例。 例如，要获取 JWT（JSON Web Token）存储实例，您可以执行以下操作：

```php
$container->get(\Spiral\Auth\TokenStorageProviderInterface::class)
    ->getStorage('jwt');
```

### 令牌存储作用域

`Spiral\Auth\TokenStorageScope` 旨在充当上下文感知服务，该服务提供了一种简单的方式来访问每个传入请求的正确令牌存储。 与跨多个请求存在的全局服务或控制器（单例）不同，它确保您正在使用与当前请求相关的正确令牌存储实例。

```php
use Spiral\Auth\TokenStorageScope;

final readonly class UserController {
    public function __construct(
        private \Spiral\Auth\TokenStorageScope $tokenStorage,
    ) {}
    
    public function currentUser() 
    {
        $this->tokenStorage->load('some-id');
      
        $this->tokenStorage->create(['id' => 'some-id']);
          
        $this->tokenStorage->delete($token);
    }
}
```

对于应用程序的每个传入请求，它将引导您找到为该特定请求指定的令牌存储的特定实例。 这确保您始终使用正确的、特定于请求的令牌存储。

### Session 令牌存储

要在 PHP session 中存储令牌，请为 `AUTH_TOKEN_STORAGE` 环境变量使用 `session` 值。

```dotenv .env
AUTH_TOKEN_STORAGE=session
```

### 数据库令牌存储

该框架可以通过 Cycle ORM 将令牌存储在数据库中。 如果您想使用这种类型的令牌，您需要安装 `spiral/cycle-bridge` 包。

```terminal
composer require spiral/cycle-bridge
```

> **更多信息**
> 阅读有关 [基础知识 — 数据库和 ORM](../basics/orm.md) 部分中 `spiral/cycle-bridge` 的安装和配置的更多信息。

然后，您需要激活 `Spiral\Cycle\Bootloader\AuthTokensBootloader`：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Auth\HttpAuthBootloader::class,
        \Spiral\Cycle\Bootloader\AuthTokensBootloader::class,
        // ...
    ];
}
```

有关引导程序的更多信息，请阅读 [框架 — 引导程序](../framework/bootloaders.md) 部分。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Auth\HttpAuthBootloader::class,
    \Spiral\Cycle\Bootloader\AuthTokensBootloader::class,
    // ...
];
```

有关引导程序的更多信息，请阅读 [框架 — 引导程序](../framework/bootloaders.md) 部分。
:::

::::

要将令牌存储在 Cycle ORM 中，请为 `AUTH_TOKEN_STORAGE` 环境变量使用 `cycle` 值。

```dotenv .env
AUTH_TOKEN_STORAGE=cycle
```

您必须生成并运行数据库迁移：

```terminal
php app.php migrate:init
php app.php cycle:migrate -v -r
```

或者运行 `cycle:sync` 以创建所需的表。

### 自定义令牌存储

请记住，如果提供的实现不能满足您的需求，您还可以创建自己的 `TokenStorageInterface` 自定义实现。 这允许您使用您选择的任何存储机制来存储应用程序中的身份验证令牌。

要在 Spiral 中创建自定义存储，您将需要创建一个实现 `Spiral\Auth\TokenStorageInterface` 接口的类。

这是一个自定义令牌存储实现的一个简单示例：

```php
use Firebase\JWT\JWT;
use Firebase\JWT\Key;
use Firebase\JWT\ExpiredException;
use Spiral\Auth\TokenInterface;
use Spiral\Auth\TokenStorageInterface;

final class JwtTokenStorage implements TokenStorageInterface
{
    /** @var callable */
    private $time;

    public function __construct(
        private readonly TokenEncoder $tokenEncoder,
        private readonly string $secret,
        private string $algorithm = 'HS256',
        private readonly string $expiresAt = '+30 days',
        callable $time = null
    ) {
        $this->tokenEncoder = $tokenEncoder;
        $this->expiresAt = $expiresAt;
        $this->time = $time ?? static function (string $offset): \DateTimeImmutable {
            return new \DateTimeImmutable($offset);
        };
    }

    public function load(string $id): ?TokenInterface
    {
        // ...
    }

    public function create(array $payload, \DateTimeInterface $expiresAt = null): TokenInterface
    {
        // ...
    }

    public function delete(TokenInterface $token): void
    {
        // ...
    }
}
```

> **注意**
> 自定义令牌存储的完整示例可以在 [此处](../cookbook/user-authentication.md) 找到。

Spiral 提供了几种注册令牌存储的方法。

:::: tabs

::: tab 引导程序
您将需要获取 `HttpAuthBootloader` 的一个实例并使用其 `addTokenStorage` 方法。
此方法接受两个参数：存储的名称和实现 `Spiral\Auth\TokenStorageInterface` 的类。

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Boot\Bootloader;
use Spiral\Bootloader\Auth\HttpAuthBootloader;

final class AppBootloader extends Bootloader
{
    public function init(HttpAuthBootloader $httpAuth, JwtTokenStorage $storage): void 
    {
        $httpAuth->addTokenStorage('jwt', $storage);
    }
}
```

:::

::: tab 配置
您也可以通过配置文件 `app/config/auth.php` 注册令牌存储。

```php app/config/auth.php
use Spiral\Core\Container\Autowire;

return [
    // ...
    'storages' => [
         'session' => \Spiral\Auth\Session\TokenStorage::class,
         'jwt' => new Autowire(\App\JwtTokenStorage::class, [
             'secret' => 'secret', 
             'algorithm' => 'HS256',
             'expiresAt' => '+30 days',
         ]),
         // ...
    ]
]
```

:::

::::

## 与 HTTP 层的用法

### 中间件

有三个中间件类可用于从请求中获取身份验证令牌，并根据令牌对用户进行身份验证。

- `Spiral\Auth\Middleware\AuthMiddleware` - 使用默认令牌存储和所有可用的传输方式（如 `cookies`、`headers`、`query parameters` 等）从请求中获取令牌。
- `Spiral\Auth\Middleware\AuthTransportMiddleware` - 使用默认令牌存储和特定的传输方式从请求中获取令牌。
- `Spiral\Auth\Middleware\AuthTransportWithStorageMiddleware` - 使用特定的令牌存储和特定的传输方式从请求中获取令牌。

[应用程序包](https://github.com/spiral/app) 提供了 `App\Application\Bootloader\RoutesBootloader`，您可以在其中轻松地定义中间件。

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Auth\Middleware\AuthMiddleware;
use Spiral\Auth\Middleware\AuthTransportMiddleware;
use Spiral\Auth\Middleware\AuthTransportWithStorageMiddleware;
use Spiral\Core\Container\Autowire;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...
    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                AuthMiddleware::class,
            ],
            'api' => [
                new Autowire(AuthTransportMiddleware::class, ['transportName' => 'header']),
            ],
            'api_jwt' => [
                new Autowire(AuthTransportWithStorageMiddleware::class, ['transportName' => 'header', 'storage' => 'jwt']),
            ],
        ];
    }
    // ...
}
```

> **更多信息**
> 阅读有关 [HTTP — 中间件](../http/middleware.md) 部分中中间件的更多信息。

### 令牌传输

在 Spiral 中，`Spiral\Auth\HttpTransportInterface` 用于使用 PSR-7 接口在 HTTP 请求和响应中读取和写入身份验证令牌。

此接口定义了三种方法：

- `fetchToken` 方法用于从传入请求中检索身份验证令牌。 这可能来自 cookie、标头或可能存储令牌的任何其他位置。

- `commitToken` 方法用于将身份验证令牌写入传出响应。 这可能涉及设置 cookie、添加标头或任何其他存储令牌的方法。

- `removeToken` 方法用于从传出响应中删除身份验证令牌。 这可能涉及取消设置 cookie 或删除标头。

有两个传输可用于 `HttpTransportInterface`：

- `cookie` 将身份验证令牌存储在 cookie 中。 当客户端向服务器发出请求时，cookie 将包含在请求中，并可用于识别客户端。

- `header` 默认情况下将身份验证令牌存储在 HTTP 标头 `X-Auth-Token` 中。

您可以使用 `AUTH_TOKEN_TRANSPORT` 环境变量为您的应用程序设置默认传输。

或者通过使用 `app/config/auth.php` 中的 `defaultTransport` 键定义：

```php app/config/auth.php
return [
    'defaultTransport' => env('AUTH_TOKEN_TRANSPORT', 'cookie'),
    // ... storages
]
```

Spiral 提供了几种注册令牌传输的方法。

:::: tabs

::: tab 引导程序
您将需要获取 `HttpAuthBootloader` 的一个实例并使用其 `addTransport` 方法。
此方法接受两个参数：传输的名称和实现 `Spiral\Auth\HttpTransportInterface` 的类。

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Boot\Bootloader;
use Spiral\Bootloader\Auth\HttpAuthBootloader;
use Spiral\Auth\Transport\CookieTransport;
use Spiral\Auth\Transport\HeaderTransport;

final class AppBootloader extends Bootloader
{
    public function boot(HttpAuthBootloader $httpAuth): void 
    {
        $httpAuth->addTransport(
          'cookie', 
          new CookieTransport(cookie: 'token', basePath: '/')
        );

        $httpAuth->addTransport(
          'header', 
          new HeaderTransport(header: 'X-Auth-Token')
        );
    }
}
```

:::

::: tab 配置
您也可以通过配置文件 `app/config/auth.php` 注册令牌传输。

```php app/config/auth.php
use Spiral\Auth\Transport\CookieTransport;
use Spiral\Auth\Transport\HeaderTransport;

return [
    // ...
    'transports' => [
         'header' => new HeaderTransport(header: 'X-Auth-Token'),
         'cookie' => new CookieTransport(cookie: 'token', basePath: '/')
         // ...
    ]
]
```

:::

::::

## 参与者提供程序和令牌负载

配置获取参与者/用户的方式的下一步是基于令牌负载，我们必须为这些目的实现并注册接口 `Spiral\Auth\ActorProviderInterface`。

```php
interface ActorProviderInterface
{
    public function getActor(TokenInterface $token): ?object;
}
```

对于这篇文章，我们将使用 Cycle Entity 和 Repository：

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

#[Cycle\Entity(repository: UserRepository::class)]
#[Index(columns: ['username'], unique: true)]
class User
{
    #[Cycle\Column(type: 'primary')]
    public int $id;

    #[Cycle\Column(type: "string")]
    public string $name;

    #[Cycle\Column(type: "string")]
    public string $username;

    #[Cycle\Column(type: "string")]
    public string $password;
}
```

我们可以在 UserRepository 中实现该接口：

```php
namespace App\Database\Repository;

use Cycle\ORM\Select\Repository;
use Spiral\Auth\ActorProviderInterface;
use Spiral\Auth\TokenInterface;

class UserRepository extends Repository implements ActorProviderInterface
{
    public function getActor(TokenInterface $token): ?object
    {
        if (!isset($token->getPayload()['userID'])) {
            return null;
        }

        return $this->findByPK($token->getPayload()['userID']);
    }
}
```

迁移完成后，我们可以创建我们的第一个用户：

```php
use Cycle\ORM\EntityManagerInterface;

public function index(EntityManagerInterface $entityManager)
{
    $user = new User();
    
    $user->name = 'Antony';
    $user->username = 'username';
    $user->password = \password_hash('password', PASSWORD_DEFAULT);

    $entityManager->persist($user)->run();
}
```

注册参与者提供程序以启用它，在您的应用程序中创建并激活 Bootloader：

```php app/src/Application/Bootloader/UserBootloader.php
namespace App\Application\Bootloader;

use App\Database\Repository\UserRepository;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Auth\AuthBootloader;

class UserBootloader extends Bootloader
{
    public function boot(AuthBootloader $auth): void
    {
        $auth->addActorProvider(UserRepository::class);
    }
}
```

### 注销

要注销用户，请调用身份验证上下文或 AuthScope 的 `close` 方法：

```php
public function logout(): void
{
    $this->auth->close();
}
```

## RBAC 安全

您可以使用经过身份验证的用户作为 RBAC 安全组件的参与者，请确保在您的 `App\Database\User` 中实现 `Spiral\Security\ActorInterface`：

```php
namespace App\Database;

use Spiral\Security\ActorInterface;
use Cycle\Annotated\Annotation as Cycle;

#[Cycle\Entity(repository: UserRepository::class)]
#[Index(columns: ['username'], unique: true)]
class User implements ActorInterface
{
    #[Cycle\Column(type: 'primary')]
    public int $id;

    #[Cycle\Column(type: "string")]
    public string $name;

    #[Cycle\Column(type: "string")]
    public string $username;

    #[Cycle\Column(type: "string")]
    public string $password;

    public function getRoles(): array
    {
        return ['user'];
    }
}
```

并激活引导程序 `Spiral\Bootloader\Auth\SecurityActorBootloader` 以将两个组件链接在一起：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Auth\SecurityActorBootloader::class,
        // ...
    ];
}
```

有关引导程序的更多信息，请阅读 [框架 — 引导程序](../framework/bootloaders.md) 部分。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Auth\SecurityActorBootloader::class,
    // ...
];
```

有关引导程序的更多信息，请阅读 [框架 — 引导程序](../framework/bootloaders.md) 部分。
:::

::::

## 防火墙中间件

您可以通过附加防火墙中间件来保护您的某些路由目标，以防止未经授权的访问。

Spiral 提供了以下防火墙中间件：

#### 将覆盖目标 URL 的防火墙

```php
use Spiral\Auth\Middleware\Firewall\OverwriteFirewall;

// ...

(new Route('/account/<controller>/<action>', $accountTarget))
        ->withMiddleware(new OverwriteFirewall(new Uri('/account/login')));
```

#### 将抛出异常的防火墙

```php
use Spiral\Auth\Middleware\Firewall\ExceptionFirewall;
use Spiral\Http\Exception\ClientException\ForbiddenException;

// ...

(new Route('/account/<controller>/<action>', $accountTarget))
        ->withMiddleware(new ExceptionFirewall(new ForbiddenException()));
```

```php
use Spiral\Http\Exception\ClientException\RedirectFirewall;
use Psr\Http\Message\ResponseFactoryInterface;

// ...

(new Route('/account/<controller>/<action>', $accountTarget))
        ->withMiddleware(new RedirectFirewall(
            uri: new Uri('/account/login'),
            status: 302,
            responseFactory: $container->get(ResponseFactoryInterface::class)
        ));
```

#### 将重定向到目标 URL 的防火墙

#### 自定义防火墙

要实现您的防火墙，请扩展 `Spiral\Auth\Middleware\Firewall\AbstractFirewall`：

```php
final class CustomFirewall extends AbstractFirewall
{
    public function __construct(
        // args...
    ) {
    }

    protected function denyAccess(Request $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // return response
    }
}
```

## 事件

| 事件                            | 描述                                                          |
|---------------------------------|---------------------------------------------------------------|
| Spiral\Auth\Event\Authenticated | 事件将在用户身份验证成功 `之后` 触发。                       |
| Spiral\Auth\Event\Logout        | 事件将在用户注销成功 `之后` 触发。                           |

> **注意**
> 要了解有关调度事件的更多信息，请参阅我们文档中的 [事件](../advanced/events.md) 部分。
