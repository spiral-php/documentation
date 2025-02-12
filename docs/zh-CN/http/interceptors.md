# HTTP — 拦截器

Spiral 提供了用于 HTTP 请求的拦截器，允许你在请求生命周期的各个阶段拦截和修改请求和响应。

`Spiral\Boot\CoreInterface` 接口通常在容器中绑定到 `Spiral\Core\Core` 类（默认情况）。 `Core` 类负责处理控制器，并且是应用程序的入口点。它负责解析控制器，处理请求并返回响应。 它还负责管理应用程序的生命周期，并跟踪当前的请求和响应。

> **了解更多**
> 可以在 [框架 — 拦截器](../framework/interceptors.md) 章节中阅读更多关于拦截器的信息。

## 域核心构建器

框架提供了一个方便的 Bootloader，名为 `Spiral\Bootloader\DomainBootloader`，允许开发者注册拦截器，并为应用程序添加通用功能，如日志记录、错误处理和安全措施，只需在一个地方进行配置，而不是将它们添加到每个控制器中。

该 Bootloader 还提供了配置拦截器执行顺序的能力，使开发人员可以控制应用程序的流程。

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Interceptor\CustomInterceptor;
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        HandleExceptionsInterceptor::class,
        JsonPayloadResponseInterceptor::class,
    ];
}
```

## 示例

### Cycle 实体解析

[Cycle Bridge](https://github.com/spiral/cycle-bridge/) 包提供了 `Spiral\Cycle\Interceptor\CycleInterceptor`。 使用 `CycleInterceptor` 根据参数值自动解析实体注入：

要激活拦截器：

```php app/src/Application/Bootloader/DomainBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Cycle\Interceptor\CycleInterceptor;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        // ...
        CycleInterceptor::class,
    ];
}
```

你可以在你的 `UserController` 方法中使用任何 cycle 实体注入，`<id>` 参数将用作主键。 如果找不到实体，将抛出 404 异常。

```php app/src/Endpoint/Web/UserController.php
namespace App\Endpoint\Web;

use App\Domain\Blog\Entity\User;
use Spiral\Router\Annotation\Route;

final class UserController
{
    #[Route(route: '/users/<id>')]
    public function show(User $user)
    {
        dump($user);
    }
}
```

> **了解更多**
> 在 [HTTP — 路由](../http/routing.md#routes-based-on-attributes) 章节中阅读更多关于基于注解的路由的信息。

如果你期望多个实体，你必须使用命名参数：

```php app/src/Endpoint/Web/BlogController.php
namespace App\Endpoint\Web;

use App\Domain\Blog\Entity\Blog;
use App\Domain\Blog\Entity\Author;
use Spiral\Router\Annotation\Route;

final class BlogController
{
    #[Route(route: '/blog/<author>/<post>')]
    public function show(Author $author, Blog $post)
    {
        dump($author, $blog);
    }
}
```

> **注意**
> 方法的参数必须命名为路由参数。

### Guard 拦截器

使用 `Spiral\Domain\GuardInterceptor` 实现 RBAC 预授权逻辑（确保安装并激活 `spiral/security`）。

```php app/src/Application/Bootloader/DomainBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Domain\GuardInterceptor;
use Spiral\Security\Actor\Guest;
use Spiral\Security\PermissionsInterface;
use Spiral\Security\Rule;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        // ...
        GuardInterceptor::class
    ];

    public function boot(PermissionsInterface $rbac): void
    {
        $rbac->addRole(Guest::ROLE);
        $rbac->associate(Guest::ROLE, 'home.*', Rule\AllowRule::class);
        $rbac->associate(Guest::ROLE, 'home.about', Rule\ForbidRule::class);
    }
}
```

你可以使用属性配置要应用于控制器操作的权限：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Domain\Annotation\Guarded;

class HomeController
{
    #[Guarded(permission: 'home.index')]
    public function index(): string
    {
        return 'OK';
    }

    #[Guarded(permission: 'home.about')]
    public function about(): string
    {
        return 'OK';
    }
}
```

要指定当未检查权限时使用的后备操作，请使用 `Guarded` 的 `else` 属性：

```php app/src/Endpoint/Web/HomeController.php
#[Guarded(permission: 'home.about', else: 'notFound')]
public function about(): string
{
    return 'OK';
}
```

> **注意**
> 允许的值：`notFound` (404), `forbidden` (401), `error` (500), `badAction` (400)。

使用属性 `Spiral\Domain\Annotation\GuardNamespace` 指定控制器 RBAC 命名空间并从每个操作中删除前缀。 你还可以在指定命名空间时跳过 `Guarded` 中的权限定义（安全组件将使用 `namespace.methodName` 作为权限名称）。

```php app/src/Endpoint/Web/HomeController.php
use Spiral\Domain\Annotation\Guarded;
use Spiral\Domain\Annotation\GuardNamespace;

#[GuardNamespace(namespace: 'home')]
class HomeController
{
    #[Guarded]
    public function index(): string
    {
        return 'OK';
    }

    #[Guarded(else: 'notFound')]
    public function about(): string
    {
        return 'OK';
    }
}
```

#### 规则上下文

你可以将所有方法参数用作规则上下文，例如，我们可以创建一个规则：

```php app/src/Application/Security/SampleRule.php
namespace App\Application\Security;

use Spiral\Security\ActorInterface;
use Spiral\Security\RuleInterface;

class SampleRule implements RuleInterface
{
    public function allows(ActorInterface $actor, string $permission, array $context): bool
    {
        return $context['user']->getID() !== 1;
    }
}
```

要激活该规则：

```php app/src/Application/Bootloader/DomainBootloader.php
namespace App\Application\Bootloader;

use App\Application\Security\SampleRule;
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Cycle\Interceptor\CycleInterceptor;
use Spiral\Domain\GuardInterceptor;
use Spiral\Security\Actor\Guest;
use Spiral\Security\PermissionsInterface;
use Spiral\Security\Rule;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        //...
        CycleInterceptor::class,
        GuardInterceptor::class
    ];

    public function boot(PermissionsInterface $rbac): void
    {
        $rbac->addRole(Guest::ROLE);
        $rbac->associate(Guest::ROLE, 'home.*', SampleRule::class);
        $rbac->associate(Guest::ROLE, 'home.about', Rule\ForbidRule::class);
    }
}

```

> **注意**
> 确保该路由包含 `<id>` 或 `<user>` 参数。

并修改该方法：

```php
#[Guarded] 
public function index(User $user): string
{
    return 'OK';
}
```

该方法将不允许使用用户 ID `1` 调用该方法。

> **注意**
> 确保在域核心中 `GuardInterceptor` 之前启用 `CycleInterceptor`。

### DataGrid 拦截器

你可以使用 `DataGrid` 属性和 `GridInterceptor` 自动将数据网格规范应用于可迭代输出。
此拦截器在端点调用后被调用，因为它使用输出。

```php app/src/Endpoint/Web/UsersController.php
use App\Domain\User\Repository\UserRepository;
use App\Intergarion\Keeper\View\UserGrid;
use Spiral\DataGrid\Annotation\DataGrid;
use Spiral\Router\Annotation\Route;

class UsersController
{
    #[Route(route: '/users', name: 'users')]
    #[DataGrid(grid: UserGrid::class)]
    public function list(UserRepository $userRepository): iterable
    {
        return $userRepository->select();
    }
}   
```

> **注意**
> `grid` 属性应该引用一个在构造函数中声明了规范的 `GridSchema` 类。

```php app/src/Intergarion/ViewKeeper/Keeper/UserGrid.php
namespace App\Intergarion\Keeper\View;

use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter;
use Spiral\DataGrid\Specification\Value;

class UserGrid extends GridSchema
{
    public function __construct()
    {
        $this->addSorter('email', new Sorter\Sorter('email'));
        $this->addSorter('name', new Sorter\Sorter('name'));
        $this->addFilter('status', new Filter\Equals('status', new Value\EnumValue(new Value\StringValue(), 'active', 'disabled')));
        $this->setPaginator(new PagePaginator(20, [10, 20, 50, 100]));
    }
}
```

（可选）你可以指定 `view` 属性，以指向每个记录的可调用 presenter。
如果没有指定，`GridInterceptor` 将在声明的网格中调用 `__invoke`。

```php app/src/Intergarion/ViewKeeper/Keeper/UserGrid.php
namespace App\Application\View;

use Spiral\DataGrid\GridSchema;
use App\Database\User;

class UserGrid extends GridSchema
{
    //...
    
    public function __invoke(User $user): array
    {
        return [
            'id'     => $user->id,
            'name'   => $user->name,
            'email'  => $user->email,
            'status' => $user->status
        ];
    }
}
```

你可以通过 `defaults` 属性或在你的网格中使用 `getDefaults()` 方法来指定网格默认值（例如默认排序、过滤、分页）：

```php app/src/Endpoint/Web/UsersController.php
#[DataGrid(
    grid: UserGrid::class,
    defaults: [
        'sort' => ['name' => 'desc'],
        'filter' => ['status' => 'active'],
        'paginate' => ['limit' => 50, 'page' => 10]
    ]
)]
```

默认情况下，网格输出将如下所示：

```json
{
  "status": 200,
  "data": [
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ]
}
```

你可以重命名 `data` 属性或在网格中传递确切的 `status` 代码 `options` 或 `getOptions()` 方法：

```php app/src/Endpoint/Web/UsersController.php
#[DataGrid(grid: UserGrid::class, options: ['status' => 201, 'property' => 'users'])]
```

```json
{
  "status": 201,
  "users": [
    ...
  ]
}
```

`GridInterceptor` 将创建一个 `GridFactoryInterface` 实例，以将给定的可迭代源包装在声明的网格模式中。 默认情况下使用 `GridFactory`，但是如果你需要更复杂的逻辑，例如使用自定义计数器或规范利用，你可以在注解中声明你自己的工厂：

```php app/src/Endpoint/Web/UsersController.php
#[DataGrid(grid: UserGrid::class, factory: InheritedFactory::class)]
```

### Pipeline 拦截器

此拦截器允许使用 `@Pipeline` 注解自定义端点拦截器。
当在域核心拦截器列表中声明时，此拦截器将指定注解的拦截器注入到声明 `PipelineInterceptor` 的位置。

```php app/src/Application/Bootloader/DomainBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\DataGrid\Interceptor\GridInterceptor;
use Spiral\Domain;
use Spiral\Cycle\Interceptor\CycleInterceptor;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        Domain\PipelineInterceptor::class, //all annotated interceptors go here
        Domain\GuardInterceptor::class,
        Domain\FilterInterceptor::class,
        GridInterceptor::class,
    ];
}
```

`Pipeline` 属性允许跳过后续拦截器：

```php
#[Pipeline(pipeline: [OtherInterceptor::class], skipNext: true)]
public function action(): string
{
    //
}
 ```

使用先前的 bootloader，我们将获得下一个拦截器列表：

- Spiral\Cycle\Interceptor\CycleInterceptor
- OtherInterceptor

> **注意**
> `PipelineInterceptor` 之后的所有拦截器都将被省略。

### 用例

例如，当端点不应应用任何拦截器或并非所有拦截器当前都需要时，它可能很有用：

```php
#[Route(route: '/show/<user:int>/email/<email:int>', name: 'emails')]
#[Pipeline(pipeline: [CycleInterceptor::class, GuardInterceptor::class], skipNext: true)]
public function email(User $user, Email $email, EmailFilter $filter): string
{
    $filter->setContext(compact('user', 'email'));
    if (!$filter->isValid()) {
        throw new ForbiddenException('Email doesn\'t belong to a user.');
    }
    //...
}
 ```

> **注意**
> 由于上下文复杂，因此不应在此处应用 `FilterInterceptor`，因此我们手动设置它并调用自定义的 `isValid()` 检查。 此外，`GridInterceptor` 在此处是多余的。

要完全控制拦截器列表，你需要将 `PipelineInterceptor` 指定为第一个。

### 整合

一起使用所有拦截器来实现丰富的域逻辑和安全的控制器操作：

```php app/src/Application/Bootloader/DomainBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\DataGrid\Interceptor\GridInterceptor;
use Spiral\Domain;
use Spiral\Cycle\Interceptor\CycleInterceptor;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        Domain\GuardInterceptor::class,
        Domain\FilterInterceptor::class,
        GridInterceptor::class,
    ];
}
```

## 特定于路由的核心

要为特定路由激活一个核心，你可以创建 `InterceptableCore` 类的新实例，并将原始的核心实例作为参数传递。 然后你可以使用 `addInterceptor(` 方法注册特定于路由的拦截器。

```php
$customCore = new InterceptableCore($core);
$customCore->addInterceptor(new CustomInterceptor());

$router->setRoute(
    'home',
    new Route(
        '/home/<action>',
        (new Controller(HomeController::class))->withCore($customCore)
    )
);
```

## 路由参数类型转换

### 整型值转换

如果你想在控制器中使用类型化的路由参数注入，例如 `function user(int $id)`，你需要自己进行值转换。 你可以为此使用域拦截器。

你可以在下面看到一个简单拦截器的示例：

```php
class StringToIntParametersInterceptor implements CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        foreach ($parameters as $key => $parameter) {
            if (ctype_digit($parameter)) {
                $parameters[$key] = (int)$parameter;
            }
        }

        return $core->callAction($controller, $action, $parameters);
    }
}
```

### 值对象转换

你可以使用相同的方法将值转换为值对象。

例如，如果控制器操作期望 `Ramsey\Uuid\Uuid` 对象

```php app/src/Endpoint/Web/UsersController.php
use Ramsey\Uuid\UuidInterface;

class UserController 
{
    public function user(UuidInterface $uuid): User
    {
        // ...
    }
}
```

你可以使用以下拦截器自动将字符串值转换为 `Ramsey\Uuid\Uuid` 对象：

```php
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Ramsey\Uuid\UuidInterface;
use Ramsey\Uuid\Uuid;

final class UuidParametersConverterInterceptor implements CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        $refMethod = new \ReflectionMethod($controller, $action);

        // Iterate all Controller action arguments
        foreach ($refMethod->getParameters() as $parameter) {
            // If an arguments has Ramsey\Uuid\UuidInterface type hint.
            if ($parameter->getType()->getName() === UuidInterface::class) {
                // Replace argument value with Uuid instance.
                $parameters[$parameter->getName()] = Uuid::fromString($parameters[$parameter->getName()]);
            }
        }

        return $core->callAction($controller, $action, $parameters);
    }
}
```
