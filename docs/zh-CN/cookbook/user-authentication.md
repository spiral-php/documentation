# Cookbook — 基于 JWT 令牌存储的用户身份验证

Spiral 被设计成灵活的，并允许开发者使用多种身份验证存储机制。除了传统的基于会话的身份验证和基于数据库的身份验证，该框架还允许使用类似 [JSON Web Tokens (JWT)](https://jwt.io/) 的身份验证方法。

使用 JWT 进行身份验证允许开发者将用户身份验证信息存储在一个紧凑的、自包含的令牌中，该令牌可以轻松地在客户端和服务器之间传输。这在客户端和服务器分布在不同环境或网络中的情况下特别有用，因为它允许安全高效的通信，而无需额外的基础设施或依赖项。

Spiral 提供了许多接口和类，用于在应用程序中实现用户身份验证。例如，`Spiral\Auth\TokenStorageInterface` 可用于为已验证用户生成和存储令牌，而 `Spiral\Auth\HttpTransportInterface` 可用于向客户端发送或接收令牌。

> **了解更多**
> 在 [Security — 用户身份验证](../security/authentication.md) 部分阅读有关接口的更多信息。

在应用程序中实现用户身份验证之前，你需要将 `Spiral\Auth\Middleware\AuthMiddleware` 添加到中间件列表中。此中间件负责检查传入的请求是否包含有效的身份验证令牌，如果找到令牌，则从令牌存储中加载用户的身份验证信息。

> **了解更多**
> 在 [HTTP — 中间件](../http/middleware.md) 部分阅读有关中间件注册的更多信息。

当 `AuthMiddleware` 从传入的请求中加载身份验证令牌时，它会创建一个新的作用域，并将 `Spiral\Auth\AuthContext` 的实例绑定到作用域容器中的 `Spiral\Auth\AuthContextInterface`。这允许你通过将 `AuthContextInterface` 注入到你的控制器或其他组件中，在当前请求期间访问 `AuthContext` 及其关联的身份验证信息（例如用户的 ID）。

**以下是身份验证过程的示意图：**

![Auth](https://user-images.githubusercontent.com/773481/211204078-a19fa13e-e326-448a-be58-84869e344bea.jpg)

## 实现 JWT 令牌存储

在使用 JWT 进行身份验证的情况下，你需要实现 `JWTTokenStorage` 并将其定义为默认的令牌存储。

以下是如何实现 `JWTTokenStorage` 类的示例：

```php
namespace App\Auth\Storage;

use Firebase\JWT\JWT;
use Firebase\JWT\Key;
use Firebase\JWT\ExpiredException;
use Spiral\Auth\TokenInterface;
use Spiral\Auth\TokenStorageInterface;

final class JWTTokenStorage implements TokenStorageInterface
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
        try {
            $token = (array) JWT::decode($id, new Key($this->secret, $this->algorithm));
        } catch (ExpiredException $exception) {
            throw $exception;
        } catch (\Throwable $exception) {
            return null;
        }

        if (
            false === isset($token['data'])
            || false === isset($token['iat'])
            || false === isset($token['exp'])
        ) {
            return null;
        }

        return new Token(
            $id,
            $token,
            (array) $token['data'],
            (new \DateTimeImmutable())->setTimestamp($token['iat']),
            (new \DateTimeImmutable())->setTimestamp($token['exp'])
        );
    }

    public function create(array $payload, \DateTimeInterface $expiresAt = null): TokenInterface
    {
        $issuedAt = ($this->time)('now');
        $expiresAt = $expiresAt ?? ($this->time)($this->expiresAt);
        $token = [
            'iat' => $issuedAt->getTimestamp(),
            'exp' => $expiresAt->getTimestamp(),
            'data' => $payload,
        ];

        return new Token(
            JWT::encode($token,$this->secret,$this->algorithm),
            $token,
            $payload,
            $issuedAt,
            $expiresAt
        );
    }

    public function delete(TokenInterface $token): void 
    {
        // We don't need to do anything here since JWT tokens are self-contained.
    }
}
```

以下是如何实现 `Token` 类的示例：

```php
namespace App\Auth\Storage;

use DateTimeInterface;
use Spiral\Auth\TokenInterface;

final class Token implements TokenInterface
{
    public function __construct(
        private readonly string $id,
        private readonly array $token,
        private readonly array $payload,
        private readonly \DateTimeImmutable $issuedAt,
        private readonly \DateTimeImmutable $expiresAt
    ) {
    }

    public function getID(): string
    {
        return $this->id;
    }

    public function getToken(): array
    {
        return $this->token;
    }

    public function getPayload(): array
    {
        return $this->payload;
    }

    public function getIssuedAt(): \DateTimeImmutable
    {
        return $this->issuedAt;
    }

    public function getExpiresAt(): DateTimeInterface
    {
        return $this->expiresAt;
    }
}
```

通过配置文件 `app/config/auth.php` 注册它。

```php app/config/auth.php
use Spiral\Core\Container\Autowire;

return [
    // ...
    'storages' => [
         'jwt' => new Autowire(\App\Auth\Storage\JWTTokenStorage::class, [
             'secret' => 'secret', 
             'algorithm' => 'HS256',
             'expiresAt' => '+30 days',
         ]),
         // ...
    ]
]
```

之后，你可以将 `JWTTokenStorage` 设置为默认的令牌存储：

```dotenv .env
AUTH_TOKEN_STORAGE=jwt
```

## 实现用户身份验证的步骤

### 1. 创建登录表单

实现用户身份验证的第一步是创建一个登录表单，允许用户输入他们的凭据（例如用户名和密码）。

### 2. 实现一个请求过滤器

接下来，你需要实现一个请求过滤器，它可以处理来自登录表单的登录请求。请求过滤器应验证用户的凭据，并在凭据有效时对用户进行身份验证。

> **了解更多**
> 在 [Filters — 安装和配置](../filters/configuration.md) 部分阅读有关请求过滤器的更多信息。

以下是一个可用于验证用户的请求过滤器的示例：

```php
namespace App\Filters;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Filter;
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
        return new FilterDefinition(validationRules: [
            'username' => ['required', 'string', ['string::longer', 3], ['string::shorter', 32]],
            'password' => ['required'],
        ]);
    }
}
```

### 3. 在控制器中创建一个登录方法

为了对后续请求进行用户身份验证，你需要创建一个控制器中的登录方法，专门用于身份验证。一旦登录方法被实现，你就可以通过调用它并传入用户的凭据来对用户进行身份验证。

以下是一个可用于验证用户的登录方法的示例：

```php app/src/Endpoint/Web/LoginController.php
namespace App\Endpoint\Web;

use App\Filters\LoginRequest;
use App\Repository\UserRepository;
use Spiral\Auth\AuthScope;
use Spiral\Auth\TokenStorageInterface;
use Spiral\Router\Annotation\Route;

class LoginController
{
    public function __construct(
        private readonly UserRepository $users,
        private readonly AuthScope $auth,
        private readonly TokenStorageInterface $tokens
    ) {
    }

    #[Route(route: '/login', name: 'login.post', methods: ['POST'])]
    public function login(LoginRequest $request)
    {
        // application specific login logic
        $user = $this->users->findByUsername($request->username);

        if (
            $user === null
            ||
            !$user->verifyPassword($request->password)
        ) {
            // Invalid credentials ...
        }
        
        // If credentials are valid, we can log the user in
    }
}
```

### 4. 验证用户

如果凭据有效，该方法应创建一个令牌，该令牌可用于对后续请求进行用户身份验证。此方法应使用 `AuthContextInterface` 生成带有负载（例如 `userID => id`）的令牌。

```php app/src/Endpoint/Web/LoginController.php
public function login(LoginRequest $request)
{
    // ... see above
    
    // If credentials are valid, we can log the user in
    
    $this->auth->start(
        $this->authTokens->create(['userID' => $user->id])
    );
    
    // Send success response ...
}
```

### 5. 将令牌发送给客户端

生成身份验证令牌并将其存储在令牌存储中后，最后一步是将令牌发送给客户端。`AuthMiddleware` 中间件将使用 `AuthContext` 中定义的传输或默认传输（如果未定义）自动将令牌发送给客户端。

## 使用已验证用户

`AuthMiddleware` 负责检查传入的请求是否包含身份验证令牌，如果找到令牌，则从令牌存储中加载用户的身份验证信息。

这意味着，如果已验证用户向你的 Spiral 应用程序发出请求，则 `AuthMiddleware` 将自动加载用户的令牌，并使用它来验证用户。然后，用户身份验证信息将通过 `AuthContextInterface` 提供给你的控制器和其他组件。

### 访问已验证用户

当你使用 `getActor()` 方法从 `AuthContextInterface` 检索用户对象时，将使用默认的 `Spiral\Auth\ActorProviderInterface` 从适当的数据源加载该对象。

`ActorProviderInterface` 是一个方便的接口，它提供了一种从各种数据源（例如数据库、微服务或其他外部系统）加载用户对象的标准方法。通过实现此接口，你可以创建一个自定义提供程序，该提供程序可以从你所需的数据源中检索用户对象。

要查看用户是否已通过身份验证，只需检查身份验证上下文是否具有非空的 actor：

```php
use Spiral\Http\Exception\ClientException\ForbiddenException;

public function index()
{
    if ($this->auth->getActor() === null) {
        throw new ForbiddenException();
    }
    
    dump($this->auth->getActor());
}
```

### 退出用户

要注销用户，请调用身份验证上下文或 `AuthContextInterface` 的 close 方法：

```php app/src/Endpoint/Web/LoginController.php
public function logout(): void
{
    $this->auth->close();
}
```
