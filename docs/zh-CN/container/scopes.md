# 容器 — IoC 作用域

开发长期运行的应用程序的一个重要方面是适当的上下文管理。在后台应用程序中，您不再允许将用户请求视为一个全局单例对象，并在您的服务中存储对其实例的引用。

实际上，这意味着您必须在处理用户输入时显式地请求上下文。Spiral 提供了一种简单的方式来管理这一点，即使用全局 IoC（控制反转）容器。这充当上下文载体，允许您请求特定实例，这些实例被限定在一个特定的上下文中，就像它们是全局对象一样。

## 工作原理

当处理用户请求时，该请求的相关数据将被放置在 IoC 容器的一个作用域区域中。此数据仅在处理该请求的有限时间内可用。

您可以使用 `Spiral\Core\ContainerScopeInterface->runScoped` 方法来实现此目的。默认的 Spiral 容器 `Spiral\Core\Container` 实现了这个接口。

**这是一个例子**

```php
use Psr\Container\ContainerInterface;

$container->runScoped(
    closure: function (ContainerInterface $container) {
        dump($container->get(UserContext::class);
    },
    bindings: [UserContext::class => $user,],
);
```

在这个例子中，`runScoped` 方法正在创建一个临时的作用域，在该作用域内，`UserContext` 类被绑定到 `$user` 变量。回调函数随后可以使用容器来检索 `UserContext` 对象实例，该实例在回调的作用域内可用。

> **注意**
> 即使抛出异常，框架也会确保在执行后清理作用域。

## 使用容器作用域

从 **Spiral Framework 的 3.8.0 版本**开始，引入了一个名为 `Spiral\Core\ContainerScopeInterface` 的新接口。该接口取代了之前的 `Spiral\Core\ScopeInterface`，并提供了一种更方便、更高级的方式来处理控制反转 (IoC) 容器中的作用域。

Spiral Framework 的 Container 类现在实现了这个接口。这意味着当您在 Spiral 中使用 Container 实例时，可以直接通过容器实例使用这些新功能。

### 方法签名

这个接口允许您在一个独特定义且隔离的 IoC 容器作用域内运行特定的代码块（一个可调用对象）。其工作方式如下：

```php
public function runScoped(callable $closure, array $bindings = [], ?string $name = null, bool $autowire = true): mixed
```

**参数**

- `$closure`: 一个可调用对象，表示您想要在隔离作用域内运行的代码块。
- `$bindings`: 一个可选的数组，包含键值对，用于定义此作用域的特定绑定。
- `$name`: 一个可选的字符串，允许您为此作用域提供一个唯一的名称。
- `$autowire`: 一个布尔值标志（默认为 `true`），指定是否应该自动解析（自动装配）依赖项。

该方法返回传递的可调用对象的结果。

### 基本用法

当您调用 `runScoped` 方法时，它会在一个单独的、隔离的作用域中执行提供的 `$closure`（一个可调用对象，例如匿名函数）。在 `$bindings` 数组中指定的任何依赖项或绑定将仅在此特定作用域内可用。

当 `$autowire` 设置为 `true` 时，IoC 容器将根据类型提示自动解析并注入给定 `$closure` 所需的所有依赖项。

**这是一个例子**

```php
$result = $container->runScoped(closure: function (SomeInterface $instance) {
    // Your code here
}, bindings: [SomeInterface::class => SomeImplementation::class]);
```

当 `$autowire` 设置为 `false` 时，容器将不会自动解析 `$closure` 的依赖项。相反，整个容器实例本身将作为第一个参数传递给 `$closure`。这样，您可以在闭包内手动从容器中获取或操作必要的依赖项。

```php
$container->runScoped(closure: function (Contaner $container) {
    $instance = $container->get(SomeInterface::class);
    // Your code here
}, bindings: [SomeInterface::class => SomeImplementation::class], autowire: false);
```

### 配置命名作用域的绑定

您可以选择预先配置特定于命名作用域的绑定。容器中有一个 `getBinder` 方法，它允许您获取与特定命名作用域关联的 `BinderInterface` 实例。使用这个绑定器，您可以为该作用域设置默认绑定。

```php
// Configure `root` scope bindings (the current instance)
$container->bindSingleton(Interface::class, Implementation::class);

// Configure `request` scope default bindings
// Prefer way to make many bindings
$binder = $container->getBinder('request');
$binder->bindSingleton(Interface::class, Implementation::class);
$binder->bind(Interface::class, factory(...));
```

#### 作用域销毁

一旦一个作用域完成（销毁），在该作用域内初始化的所有依赖项（包括单例）也会被销毁。这确保了干净且有效的资源管理。下次创建具有相同名称的作用域时，这些绑定将重新解析，遵循您使用 `BinderInterface` 设置的配置。

#### 覆盖默认绑定

当您使用 `Container::runScoped()` 方法创建新作用域时，您可以选择传递 `$bindings` 参数来覆盖该特定作用域的默认绑定。

- 这些自定义绑定将在新创建的作用域中优先。
- 这不会影响您之前为该命名作用域定义的默认绑定。

**示例：**

```php
$container->bindSingleton(SomeInterface::class, SomeImplementation::class);

$container->runScoped(closure: function ($container) {
    $instance = $container->get(SomeInterface::class);
    // Your code here
}, bindings: [SomeInterface::class => AnotherImplementation::class], name: 'request');
```

在此示例中，即使 `request` 作用域可能具有 `SomeInterface::class` 的默认绑定，但该特定作用域运行将使用 `AnotherImplementation::class`。

### 作用域销毁

当一个依赖项在一个作用域内被解析，并且该作用域被销毁时，将调用此属性指定的 finalize 方法。finalize 方法的目的是在销毁作用域之前执行任何必要的清理或终结操作。在这种情况下，您可以使用 `Finalize` 属性来控制终结过程。该属性提供了一种方便的方法来处理资源清理，并确保在不再需要作用域时正确销毁对象。

```php
#[Finalize(method: 'finalize')]
class Foo
{
    public bool $finalized = false;

    public function finalize(Logger $logger): void
    {
        $this->finalized = true;
        $logger->log();
    }
}
```

### 命名作用域

您可以选择使用 `$name` 参数为作用域提供一个名称。这对于跟踪或管理特定作用域很有用。

**限制**

这些限制确保了正确的使用并防止在作用域层次结构中发生冲突。以下是关键限制：

1. **并行命名作用域：** 允许并行命名作用域。

2. **命名作用域的默认绑定：** 命名作用域可以具有与之关联的默认绑定。需要注意的是，更改默认绑定不会影响已经创建的具有相同作用域名称的容器实例，除了根容器。默认绑定提供了一种方便的方式来为特定的命名作用域设置通用依赖项。

3. **根作用域名称：** 层次结构中最父级的作用域默认被命名为根作用域。根作用域充当顶级作用域，并为所有其他作用域提供基础。它是依赖项解析的起点，并且可以具有其自己的默认绑定集。

### 解析规则

当一个嵌套作用域没有自己的绑定时，它将从父作用域解析依赖项。

### 访问作用域值

您可以直接从容器中访问在作用域中设置的值，或者通过依赖注入在您的服务或控制器中访问。但是，这应该仅在 IoC 作用域内完成。

**这是一个例子**

```php
public function doSomething(UserContext $user): void
{
    \dump($user);
}
```

简而言之，您可以在 IoC 作用域内从容器或注入中接收活动上下文，就像您通常为任何普通依赖项所做的那样，但您**绝不能**在作用域之间存储它。

## 上下文管理器

如上所述，您不允许存储对作用域实例的任何引用，以下代码无效，并将导致控制器锁定在第一个作用域值上：

```php Wrong approach
class HomeController
{
    public function __construct(
        private readonly UserContext $userContext // <====== !!!not allowed!!!
    ) {
    }
    
    public function index(): void
    {
        dump($this->userContext);
    }
}
```

相反，建议使用专门用于从单例中提供对 IoC 作用域访问的对象 - 上下文管理器。一个简单的上下文管理器可以这样编写：

```php
final class UserScope
{
    public function __construct(
        private readonly ContainerInterface $container
    ) {
    }

    public function getUserContext(): ?UserContext
    {
        // error checking is omitted
        return $this->container->get(UserContext::class);
    }

    public function getName(): string
    {
        return $this->getUserContext()->getName();
    }
}
```

您可以在任何服务中使用此管理器，包括单例。

```php app/src/Endpoint/Web/HomeController.php
class HomeController implements SingletonInterface
{
    public function __construct(
        private readonly UserScope $userManager
    ) {
    }
}
```

> **注意**
> 一个很好的例子是 `Spiral\Http\Request\InputManager`。该管理器充当仅在 http 分发器作用域内可用的
> `Psr\Http\Message\ServerRequestInterface` 的访问器。

## Fibers 支持

容器支持创建并行的独立作用域，这些作用域可以安全地与 fibers 一起使用。
