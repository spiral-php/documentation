# 安全 — 基于角色的访问控制

该框架包含了组件 `spiral/security`，它提供了基于关联权限列表来授权用户/参与者访问特定资源或动作的能力。该组件实现了 [NIST RBAC 研究](https://csrc.nist.gov/projects/role-based-access-control) 中描述的“扁平 RBAC”模式。

该实现包含了多个补充功能，例如：

- 用于控制权限/许可上下文的附加 *规则* 层
- 使用通配符模式将角色分配给多个权限的能力
- 使用更高优先级规则覆盖角色-权限分配的能力

> **注意**
> 这种补充功能使得该组件可以作为 ACL、DAC 和 ABAC 安全模型的框架使用。

请确保启用 `Spiral\Bootloader\Security\GuardBootloader` 以激活该组件，无需配置。

> **注意**
> 该组件默认在 Web 和 GRPC 捆绑包中启用。

## 参与者 (Actor)

应用程序内的所有权限都将基于与当前 `Spiral\Security\ActorInterface` 关联的角色授予：

```php
interface ActorInterface
{
    public function getRoles(): array;
}
```

> **注意**
> 在用户请求期间使用 IoC 作用域来设置参与者。

阅读如何将已认证用户用作参与者 [此处](../security/authentication.md)。

默认情况下，应用程序使用 `Spiral\Security\Actor\Guest` 作为默认参与者。 您可以使用容器绑定全局地或在 IoC 作用域内设置参与者。

```php
namespace App\Controller;

use Spiral\Core\ScopeInterface;
use Spiral\Security\Actor\Actor;
use Spiral\Security\ActorInterface;

class HomeController
{
    public function index(ScopeInterface $scope): string
    {
        return $scope->runScope([ActorInterface::class => new Actor(['admin'])], function (): string {

            // the actor has role `admin` in this scope
            return 'ok';
        });
    }
}
```

> **注意**
> 您可以使用域核心拦截器、GRPC 拦截器、HTTP 中间件、自定义 IoC 作用域等设置活动的参与者。

为了简化本指南，我们将通过自定义 Bootloader 全局查看默认参与者：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container;
use Spiral\Security\Actor\Actor;
use Spiral\Security\ActorInterface;

class SecurityBootloader extends Bootloader
{
    public function boot(Container $container): void
    {
        $container->bindSingleton(ActorInterface::class, new Actor(['user']));
    }
}
```

## GuardInterface

要使用 RBAC 组件，我们必须注册可用角色并创建角色和权限之间的关联。为此目的，请使用相同的 Bootloader 和 `Spiral\Security\PermissionsInterface`。

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container;
use Spiral\Security\Actor\Actor;
use Spiral\Security\ActorInterface;
use Spiral\Security\PermissionsInterface;

class ActorBootloader extends Bootloader
{
    public function boot(Container $container, PermissionsInterface $rbac): void
    {
        $container->bindSingleton(ActorInterface::class, new Actor(['user']));

        $rbac->addRole('user');
        $rbac->associate('user', 'home.read');
    }
}
```

> **注意**
> 角色-规则-权限关联将在下面详细解释。

一旦 Bootloader 被激活，您就可以使用 `Spiral\Core\GuardInterface` 来检查对特定权限的访问，guard 将使用 `Spiral\Security\GuardScope` 通过动态作用域自动解析活动参与者 (Actor)。该接口提供了 `allows` 方法，我们可以使用它来检查参与者 (Actor) 是否有权访问特定的权限：

```php
namespace App\Controller;

use Spiral\Security\GuardInterface;

class HomeController
{
    public function index(GuardInterface $guard): string
    {
        if (!$guard->allows('home.read')) {
            return 'can not read';
        }

        return 'can read';
    }
}
```

> **注意**
> 更改默认参与者 (Actor) 角色以查看它如何影响结果。

使用 `guard` 原型属性来更快地开发。

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): string
    {
        if (!$this->guard->allows('home.read')) {
            return 'can not read';
        }

        return 'can read';
    }
}
```

> **注意**
> 您可以在控制器、服务和视图中使用 `GuardInterface`。

### 权限上下文

guard 对象的 `allows` 方法支持第二个参数，该参数定义了权限上下文，通常它必须包含当前参与者 (Actor) 尝试访问或编辑的目标实体的实例。

```php
namespace App\Controller;

use Spiral\Security\GuardInterface;

class HomeController
{
    public function index(GuardInterface $guard): string
    {
        if (!$guard->allows('home.read', ['key' => 'value'])) {
            return 'can not read';
        }

        return 'can read';
    }
}
```

在下面，我们将解释如何使用上下文来创建更复杂的角色-权限关联。

## 权限管理

RBAC 组件的核心部分是 `Spiral\Security\PermissionInterface`。虽然您可以将您的实现与动态的角色和权限配置一起使用，但默认情况下，它旨在在 Bootloader 中配置映射。

### 创建角色

每个应用程序都必须在 RBAC 组件中注册可用的用户角色。使用 `addRole` 方法：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Security\PermissionsInterface;

class SecurityBootloader extends Bootloader
{
    public function boot(PermissionsInterface $rbac): void
    {
        $rbac->addRole('guest');
    }
}
```

### 权限

创建角色后，您可以使用 `associate` 方法将它们与权限关联起来：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Security\PermissionsInterface;

class SecurityBootloader extends Bootloader
{
    public function boot(PermissionsInterface $rbac): void
    {
        $rbac->addRole('guest');
        
        $rbac->associate('guest', 'home.read');
    }
}
```

### 通配符关联

您可以一次将角色关联到多个权限：

```php
$rbac->associate('guest', 'home.(read|write)');
```

您还可以使用 `*` 模式创建与整个权限命名空间的关联：

```php
$rbac->associate('guest', 'home.*');
```

请注意，一次只能关联命名空间级别，如果您使用带有 `.` 分隔符的深层命名空间：

```php
$rbac->associate('guest', 'home.*');
$rbac->associate('guest', 'home.*.*');
// ...
```

### 排除权限

在某些情况下，您需要授予对所有命名空间权限的访问权限，但排除特定的权限。 使用第三个参数覆盖权限，该参数可以指定规则 (Rule)。要禁止角色访问，请使用 `Spiral\Security\Rule\ForbidRule`：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Security\PermissionsInterface;
use Spiral\Security\Rule\ForbidRule;

class SecurityBootloader extends Bootloader
{
    public function boot(PermissionsInterface $rbac): void
    {
        $rbac->addRole('guest');

        $rbac->associate('guest', 'home.*');
        $rbac->associate('guest', 'home.read', ForbidRule::class);
    }
}
```

用于创建关联的默认规则是 `AllowRule`。 上一个例子可以使用以下方式更好地解释：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Security\PermissionsInterface;
use Spiral\Security\Rule;

class SecurityBootloader extends Bootloader
{
    public function boot(PermissionsInterface $rbac): void
    {
        $rbac->addRole('guest');

        $rbac->associate('guest', 'home.*', Rule\AllowRule::class);
        $rbac->associate('guest', 'home.read', Rule\ForbidRule::class);
    }
}
```

> **注意**
> Guard 将检查所有参与者 (Actor) 角色，其中至少一个必须授予该权限。

## 规则

如上所述，所有角色-权限关联都使用一组规则进行控制。每个 Rule 都必须实现 `Spiral\Security\RuleInterface`。

```php
interface RuleInterface
{
    public function allows(ActorInterface $actor, string $permission, array $context): bool;
}
```

在检查角色对权限的访问权限时，该规则将接收由 `GuardInterface` -> `allows` 方法提供的当前上下文：

```php
if (!$guard->allows('home.read', ['key' => 'value'])) {
    return 'can not read';
}
```

这种方法允许您在角色和一组权限之间创建复杂的规则关联。

> **注意**
> 默认规则 `AllowRule` 和 `ForbidRule` 总是分别返回 `true` 和 `fast`。

### 自定义规则

要创建自定义规则，只需实现 `Spiral\Security\RuleInterface` 接口，例如，我们可以创建仅在上下文具有 `key` 等于 `value` 时才允许访问的规则：

```php
namespace App\Security;

use Spiral\Security\ActorInterface;
use Spiral\Security\RuleInterface;

class SampleRule implements RuleInterface
{
    public function allows(ActorInterface $actor, string $permission, array $context): bool
    {
        return $context['key'] === 'value';
    }
}
```

您可以使用类名将此规则分配给角色-权限关联：

```php
namespace App\Bootloader;

use App\Security\SampleRule;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Security\PermissionsInterface;

class SecurityBootloader extends Bootloader
{
    public function boot(PermissionsInterface $rbac): void
    {
        $rbac->addRole('guest');

        $rbac->associate('guest', 'home.*', SampleRule::class);
    }
}
```

现在，更改 `allows` 方法的上下文将导致批准或拒绝：

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): string
    {
        if ($this->guard->allows('home.read', ['key' => 'value'])) {
            echo 'yay';
        }

        if ($this->guard->allows('home.read', ['key' => 'else'])) {
            echo 'nope';
        }
    }
}
```

> **注意**
> 使用参与者 (Actor) 接口来创建更复杂的规则。

将域实体作为上下文传递以检查授权：

```php
if ($this->guard->allows('post.edit', ['post' => $post])) {
    echo 'yay';
}
```

以及用于检查用户是否是作者的规则：

```php
class SampleRule implements RuleInterface
{
    public function allows(ActorInterface $actor, string $permission, array $context): bool
    {
        return $context['post']->user === $actor;
    }
}
```

### 抽象规则

为了简化规则的创建，请使用 `Spiral\Security\Rule`，它使用 `check` 方法启用了方法注入。

```php
namespace App\Security;

use Spiral\Security\Rule;

class SampleRule extends Rule
{
    public function check(string $key): bool
    {
        return $key === 'value';
    }
}
```

> **注意**
> 您可以通过方法注入访问任何应用程序服务。

我们建议将规则声明为单例，以提高应用程序的性能。

```php
namespace App\Security;

use Spiral\Core\Container\SingletonInterface;
use Spiral\Security\Rule;

class SampleRule extends Rule implements SingletonInterface
{
    public function check(string $key): bool
    {
        return $key === 'value';
    }
}
```

## Guarded 属性

您可以使用 `Guarded` 属性来使用 [拦截器](../framework/interceptors.md) 自动检查对控制器方法的访问。

```php
namespace App\Controller;

use Spiral\Domain\Annotation\Guarded;

class HomeController
{
    #[Guarded(permission: 'home.index', else: 'notFound')]
    public function index(): string
    {
        return 'OK';
    }
}
```
