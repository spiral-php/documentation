# 容器 — 特性 (Attributes)

Spiral 提供了一组特性 (attributes)，可用于更好地控制依赖注入 (dependency resolution)：

## 单例 (Singleton)

使用 `#[Singleton]` 特性 (attribute) 在需要确保一个类只创建一个实例并在整个应用程序中使用时非常方便，例如配置管理器、数据库连接或日志记录器。

#### 示例

假设您的应用程序中有一个配置管理器，它从配置文件中读取并提供对各种配置设置的访问。每次需要访问设置时都重新加载和解析配置文件将是一种浪费。使用 #[Singleton] 特性 (attribute) 可确保一旦加载并解析了配置文件，它就会在应用程序的整个生命周期中保留在内存中。

```php
use Spiral\Core\Attribute\Singleton;

#[Singleton]
final class ConfigurationManager
{
    private readonly array $config;

    public function __construct()
    {
        $this->config = parse_ini_file('config.ini');
    }

    public function get(string $key)
    {
        return $this->config[$key] ?? null;
    }
}
```

## 作用域 (Scope)

允许您设置一个特定的作用域 (scope)，在该作用域中可以解析依赖项。如果尝试在不同的作用域中解析依赖项，将会抛出异常，指示作用域不匹配。此特性 (attribute) 有助于执行严格的作用域规则，并防止依赖项在意外的作用域中被错误地解析。

#### 示例

考虑一个应用程序，其中某些资源或操作仅限于经过身份验证的用户。对于此类资源，您需要：

1.  确保用户已通过身份验证。
2.  在当前请求期间，使经过身份验证的用户的数据在整个应用程序中可用。

这个类保存了关于经过身份验证的用户的信息。此数据仅用于在用户经过身份验证时（即，在 auth 作用域内）可用和解析。

```php
use Spiral\Core\Attribute\Scope;

#[Scope('auth')]
final readonly class AuthenticatedUser
{
    public function __construct(
        private int $id,
        private string $name, 
        private string $email,
    ) {
    }
}
```

中间件 (middleware) 检查用户是否已通过身份验证。如果已通过身份验证，它会在 IoC 容器中设置一个 auth 作用域并绑定经过身份验证的用户的数据到 AuthenticatedUser 类。

```php
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\RequestHandlerInterface as Handler;

final class AuthMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ContainerInterface $container
    ) {
    }

    public function process(Request $request, Handler $handler): Response
    {
        // This is a simplified authentication check.
        // In a real application, this might involve checking session data, JWT tokens, etc.
        if ($request->hasHeader('Authorization')) {
            // Fetch user data based on the authorization header. 
            // For simplicity, we're hardcoding user data here.
            $user = new AuthenticatedUser(1, 'John Doe', 'john.doe@example.com');

            // Set up the auth scope
            return $this->container->runScoped(
                closure: function (ContainerInterface $container) use ($next, $request) {
                    // Now, within this scope, you can get the authenticated user instance.
                    $authenticatedUser = $container->get(AuthenticatedUser::class);
                    
                    // Continue processing the request.
                    return $handler($request);
                },
                bindings: [AuthenticatedUser::class => $user],
                name: 'auth'
            );

        } else {
            // No authentication header found.
            // Return an unauthorized response or simply continue processing.
            return $handler($request);
        }
    }
}
```

假设我们有一个需要经过身份验证的用户的控制器或服务。通过尝试从容器中解析 `AuthenticatedUser` 类，我们可以确保我们正在获取经过身份验证的用户，或者我们在 auth 作用域内（感谢 #[Scope('auth')] 特性 (attribute)）。

```php
final class UserProfileController
{
    public function getProfile(AuthenticatedUser $user)
    {
        // Use the $user data to fetch and return the profile.
    }
}
```

> **注意**
> 在 [框架 — IoC 作用域 (Scopes)](../framework/scopes.md) 部分阅读有关容器作用域的更多信息。

## 终结 (Finalize)

允许您为类定义一个终结方法。当在作用域内解析依赖项，并且该作用域被销毁时，将调用此特性 (attribute) 指定的终结方法。终结方法的目的是在销毁作用域之前执行任何必要的清理或终结操作。此特性 (attribute) 提供了一种方便的方式来处理资源清理并确保在不再需要作用域时正确销毁对象。

#### 示例

考虑您正在使用数据库连接。一旦您完成了它，特别是在特定的作用域内，您可能想要关闭连接或释放其他资源。

```php
use Spiral\Core\Attribute\Finalize;

#[Finalize(method: 'closeConnection')]
final class DatabaseConnection
{
    private $connection;

    public function __construct()
    {
        // Initialize the database connection
    }

    public function query($sql)
    {
        // Execute the query on the database
    }

    public function closeConnection(): void
    {
        // Close the connection
    }
}
```

当在作用域内使用此类时，一旦作用域结束，将调用 closeConnection 方法，确保释放资源：

```php
$container->runScoped(
    closure: function (DatabaseConnection $db) {
        // Execute some database operations
        $users = $db->query('SELECT * FROM users');
        // ...  
    },
    bindings: [DatabaseConnection::class => new DatabaseConnection()],
);

// Once the scope is destroyed, the connection is automatically closed.
```

> **警告**
> 对象可能被泄露但已终结。您应该避免这种情况。

```php
$root = new Container();
$obj = $root->get(Foo::class);
unset($root); // The Foo finalizer will be called

// Here we have a leaked finalized object. It is `$obj`.
```

## 组合特性 (Attributes)

所有特性 (attributes) — `#[Finalize]`、`#[Singleton]` 和 `#[Scope]` — 彼此完全兼容。这意味着您可以在同一类上同时使用这些特性 (attributes)，从而可以对依赖项的行为和生命周期进行精细控制。

#### 示例

想象一下，您有一个缓存服务，其中：

1.  在整个应用程序的生命周期中，应该只创建一个缓存服务的实例（单例）。
2.  该服务应该仅在应用程序的某些部分（例如，在 HTTP 请求处理中）可用（作用域）。
3.  当应用程序关闭或作用域结束时，您希望确保所有待处理的缓存操作（如写回）都被终结，并且任何资源（如打开的文件句柄或网络连接）都被关闭。

```php
namespace App\Services;

use Psr\Log\LoggerInterface;
use Spiral\Core\Attribute\Finalize;
use Spiral\Core\Attribute\Scope;
use Spiral\Core\Attribute\Singleton;

#[Singleton]
#[Scope('http')]
#[Finalize(method: 'shutdown')]
final class CacheService
{
    private array $cache = [];
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {
    }

    public function get(string $key): ?string
    {
        return $this->cache[$key] ?? null;
    }

    public function set(string $key, string $value): void
    {
        $this->cache[$key] = $value;
    }

    public function shutdown(): void
    {
        $this->logger->info("CacheService is finalizing.");
        
        // Flush the cache to a persistent storage, close any resources, etc.
        $this->cache = [];
    }
}
```
