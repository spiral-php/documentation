# 容器 - 概述

Spiral 框架包含一组工具，可以更轻松地管理依赖关系并在代码中创建对象。其中一个主要工具是容器，它可以帮助您处理类依赖关系并自动将它们“注入”到类中。这意味着，您可以不必手动创建对象和设置依赖关系，而是由容器来处理这些事情。

## PSR-11 容器

Spiral 容器也遵循一组 [PSR-11 容器](https://github.com/php-fig/container) 标准，从而确保与其他库的兼容性。

在您的代码中，您可以通过请求 `Psr\Container\ContainerInterface` 来访问容器。

```php app/src/Endpoint/Web/UserController.php
final class UserController
{
    public function __construct(
        private readonly \Psr\Container\ContainerInterface $container
    ) {}

    public function show(string $id): void
    {
       $repository = $this->container->get(UserRepository::class);
       // ...
    }
}
```

## 依赖注入

Spiral 容器允许您对类使用**构造函数**和**方法**注入。这意味着，您可以通过构造函数或特定方法将依赖关系自动**注入**到类中。

:::: tabs

::: tab 构造函数

例如，在 `UserController` 类中，`UserRepository` 依赖关系通过 `__construct()` 方法注入。这意味着，当调用该方法时，容器将自动创建并提供 UserRepository 对象。

```php app/src/Endpoint/Web/UserController.php
use Psr\Container\ContainerInterface;

final class UserController
{
    public function __construct(
        private readonly UserRepository $users
    ) {}

    public function show(string $id): void
    {
       $user = $this->users->findOrFail($id);
       // ...
    }
}
```

:::

::: tab 方法

例如，在 `UserController` 类中，`UserRepository` 依赖关系通过 `show()` 方法注入。这意味着，当调用该方法时，容器将自动创建并提供 UserRepository 对象。

```php app/src/Endpoint/Web/UserController.php
final class UserController
{
    public function show(UserRepository $users, string $id): void
    {
       $user = $users->findOrFail($id);
       // ...
    }
}
```

:::
::::

> **注意**
> 此功能适用于控制器、控制台命令和队列任务等类。
> 此外，容器还支持高级功能，例如联合类型、可变参数、引用参数以及自动装配的默认对象值。

### 自动依赖解析

框架容器可以通过提供具体类的实例来自动解析构造函数或方法的依赖关系。

```php
class MyController
{
    public function __construct(
        OtherClass $class, 
        SomeInterface $some
    ) {
    }
}
```

在提供的示例中，容器将尝试通过自动构造 `OtherClass` 的实例来提供它。但是，除非您在容器中进行适当的绑定，否则 `SomeInterface` 无法被解析。

```php
$container->bind(SomeInterface::class, SomeClass::class); 
```

请注意，容器将尝试解析*所有*构造函数依赖关系（除非您手动提供某些值）。这意味着，所有类的依赖关系都必须可用，或者参数必须声明为可选：

```php
// 如果未提供 `value` 依赖关系，则将失败
__construct(OtherClass $class, $value)

// 如果未提供其他值，则将使用 `null` 作为 `value`
__construct(OtherClass $class, $value = null) 

// 如果 SomeInterface 未指向具体实现，则将失败
__construct(OtherClass $class, SomeInterface $some) 

// 如果未提供具体实现，则将使用 null 作为 `some` 的值
__construct(OtherClass $class, SomeInterface $some = null) 
```

## 服务

### 绑定器

框架提供了一个 `Spiral\Core\BinderInterface`，您可以使用它将一个类绑定到接口或别名。
在 [容器 - 配置](../container/configuration.md) 部分阅读更多相关信息。

### 工厂

框架提供了一个 `Spiral\Core\FactoryInterface`，您可以使用它来构造一个类，而无需解析其所有构造函数依赖关系。当您只需要类的特定子集依赖关系，或者想要为某些依赖关系传递特定值时，这可能很有用。

```php
public function makeClass(FactoryInterface $factory): MyClass
{
    return $factory->make(UserService::class, [
        'table' => 'users'
    ]); 
}
```

在上面的示例中，`make()` 方法用于创建 `UserService` 的实例，并将值 `table` 作为参数构造函数依赖项传递。其他构造函数依赖关系将由容器自动解析。

这允许您更好地控制类的构造，并且在需要创建具有不同构造函数依赖关系的类的多个实例时尤其有用。

### 解析器

框架提供了 `Spiral\Core\ResolverInterface`，您可以使用它将方法参数解析为动态目标，例如控制器方法。当您想要调用一个方法并且需要在运行时解析其依赖关系时，这可能很有用。

```php
abstract class Handler
{
    public function __construct(
        protected readonly ResolverInterface $resolver
    ) {
    }

    public function run(array $params): bool
    {
        $method = new \ReflectionMethod($this, 'do');

        return $method->invokeArgs(
            $this, 
            $this->resolver->resolveArguments($method, $params) // resolve missing arguments
        );
    }
}
```

`run()` 方法使用 `ResolverInterface` 以方法注入的方式调用 `do` 方法。现在，`do` 方法可以请求方法注入：

```php
class UserStoreHandler extends Handler
{
    public function do(UserService $service): bool
    {
        // Store user
    }
}
```

这样，您可以轻松地在运行时解析方法的依赖关系，并使用所需的参数调用它，而不管依赖关系是作为参数传递还是需要由容器解析。

#### 支持的类型

##### 联合类型

`ResolverInterface` 的默认实现支持联合类型。将传递所需类型的一个可用依赖关系：

```php
use Doctrine\Common\Annotations\Reader;
use Spiral\Attributes\ReaderInterface;

final class Entities
{
    public function __construct(
        private Reader|ReaderInterface $reader
    ) {
    }
}
```

##### 可变参数

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int ...$bar) => $bar;

// array passed by parameter name
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => [1, 2]]
);

dump($args); // [1, 2]

// array passed by parameter name with named arguments inside
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => ['ab' => 1, 'bc' => 2]]
);

dump($args); // ['ab' => 1 'bc' => 2]

// value passed by parameter name
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => 1]
);

dump($args); // [1]

```

##### 引用参数

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int $bar) => $bar;

$bar = 1;

$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => &$bar]
);

$bar = 42;
dump($args); // [42]
```

##### 默认对象值

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(stdClass $std = new \stdClass()) => $std;

$args = $resolver->resolveArguments(new \ReflectionFunction($function));

dump($args); 

// array(1) {
//   [0] =>
//   class stdClass#369 (0) {
//   }
// }
```

#### 参数验证

在某些情况下，您可能需要验证一个函数或方法的参数。为此，您可以使用公共的 `validateArguments` 方法，在该方法中，您需要传递 `ReflectionMethod` 或 `ReflectionFunction` 以及`参数数组`。如果您使用 `resolveArguments` 方法接收参数并且未在 `$validate` 参数中传递 `false`，则不需要额外的验证。它们将自动被检查。如果参数无效，将抛出 `Spiral\Core\Exception\Resolver\InvalidArgumentException`。

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int $bar) => $bar;

$resolver->validateArguments(new \ReflectionFunction($function), [42]);
```

### 调用器

框架提供了 `Spiral\Core\InvokerInterface`，您可以使用它来调用所需的方法，并自动解析其依赖关系。当您想要调用对象上的方法并且需要在运行时解析其依赖关系时，这可能很有用。

#### 调用类方法

`InvokerInterface` 有一个 `invoke()` 方法，您可以使用它来调用对象上的方法，并为其依赖关系传递特定值。

这是一个如何使用它的例子：

```php
use Spiral\Core\InvokerInterface;

abstract class Handler
{
    public function __construct(
        protected readonly InvokerInterface $invoker
    ) {
    }

    public function run(array $params): bool
    {
        return $this->invoker->invoke([$this, 'do'], $params)
    }
}
```

如果作为第一个可调用数组值传递一个字符串 `['user-service', 'store']`，它（`user-service`）将从容器中请求。

```php
$container->bind('user-service', UserService::class);
// ...
$invoker->invoke(
    ['user-service', 'store'], 
    $params
);
```

容器将解析类 `user-service` 并在其上调用 `store` 方法。

这允许您轻松地调用由容器管理的类上的方法，而无需手动实例化它们。这在您希望将一个类用作服务并从代码库的多个部分调用其方法的情况下特别有用。

> **注意**
> 一个方法可以具有任何可见性（public、protected 或 private），并且仍然可以被调用。

#### 调用可调用对象

`InvokerInterface` 也可以用于调用闭包并自动解析它们的依赖关系。

```php
$invoker->invoke(
    function (MyClass $class, string $parameter) {
        return new MyClassService($class);
    },
    [
        'parameter' => 'value',
    ]
); 
```

这允许您轻松地调用具有所需依赖关系的闭包，而不管依赖关系是作为参数传递还是需要由容器解析。

### 作用域

开发长期运行的应用程序的一个重要方面是适当的上下文管理。在守护程序应用程序中，您不再允许将用户请求视为全局单例对象，并将对其实例的引用存储在您的服务中。

实际上，这意味着您必须在处理用户输入时显式请求上下文。Spiral 通过使用全局 IoC（控制反转）容器提供了一种简单的方法来管理这一点。这充当一个上下文载体，允许您请求特定的实例，这些实例被限定到特定的上下文，就好像它们是全局对象一样。

在 [容器 - IoC 作用域](../container/scopes.md) 部分阅读更多关于**作用域**的信息。

## 替换应用程序容器

在某些情况下，您可能希望替换应用程序中的容器实例。您可以在创建应用程序实例时执行此操作。

```php app.php
use Spiral\Core\Container;
use App\Application\Kernel;

$container = new Container();
$container->bind(...);

$app = Kernel::create(
    directories: ['root' => __DIR__],
    container: $container
)
```
