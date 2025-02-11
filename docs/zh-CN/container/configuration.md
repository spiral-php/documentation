# 容器 — 配置

Spiral 容器允许您通过创建接口或别名与特定实现的绑定来配置它。您可以使用 [引导程序](../framework/bootloaders.md) 来定义这些绑定。

## 概述

配置容器有两种方式，一种是使用 `Spiral\Core\BinderInterface`，另一种是使用 `Spiral\Core\Container`。

### 将接口绑定到实现

要将接口绑定到具体的实现，可以使用以下代码片段：

:::: tabs

::: tab BinderInterface

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bind(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

:::

::: tab Container

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;

public function boot(Container $container): void
{
    $container->bind(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

:::
::::

### 将接口绑定到单例

要绑定一个单例，您可以使用以下代码片段：

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

您还可以使用 `Autowire` 类来绑定特定的参数，如下所示：

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\Autowire;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        new Autowire(CycleUserRepository::class, ['table' => 'users'])
    );
}
```

最后，您可以使用闭包自动配置您的类，通过将一个闭包传递给 `bind` 或 `bindSingleton` 方法，如下所示：

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        static fn() => new CycleUserRepository(table: users)
    );
}
```

闭包也支持依赖注入：

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\Autowire;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        static fn(UserConfig $config) => new CycleUserRepository(table: $config->getTable())
    );
}
```

当执行此闭包时，容器将自动解析 `UserConfig` 的实例，并将其作为参数传递给闭包。这允许您轻松地使用依赖项配置您的类，而无需手动实例化和管理它们。

### 检查容器是否具有绑定

要检查容器是否具有绑定，请使用：

```php
use Spiral\Core\Container;

public function boot(Container $container): void
{
    $container->has(UserRepositoryInterface::class)
}
```

### 移除绑定

要移除容器绑定：

```php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->removeBinding(UserRepositoryInterface::class)
}
```

容器支持 `WeakReference` 绑定：

:::: tabs

::: tab 字符串别名

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;
use WeakReference;

public function boot(Container $container): void
{
    $object = new stdClass();
    $hash = \spl_object_hash($object);
    $reference = WeakReference::create($object);

    $container->bind('test-alias', $reference);
    
    dump($hash === \spl_object_hash($container->get('test-alias'))); // true
    
    unset($object);
    // 新对象无法创建，因为类名未被存储
    dump($container->get('test-alias')); // null
}
```

:::

::: tab 类名别名

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;
use WeakReference;

public function boot(Container $container): void
{
    $object = new stdClass();
    $hash = \spl_object_hash($object);
    $reference = WeakReference::create($object);

    $container->bind(stdClass::class, $reference);
    
    dump($hash === \spl_object_hash($container->get(stdClass::class))); // true
    
    unset($object);
    // 使用别名类创建新实例
    dump($hash === \spl_object_hash($container->get(stdClass::class))); // false
}
```

:::
::::

## 高级绑定

从 **Spiral Framework 的 3.8.0 版本**开始，我们已经用新的基于数据传输对象（DTO）的结构替换了以前基于数组的结构，用于在容器内存储有关绑定的信息。这种新方法提供了更结构化和有组织的方式来存储绑定信息。通过此更改，开发人员现在可以使用这些对象轻松配置容器绑定。

为了演示增强的容器功能，这里有一个示例，展示了新的绑定配置：

```php
use Spiral\Core\Config\Factory;

$container->bind(LoggerInterface::class, new Factory(
    callable: static function() {
        return new Logger(....);
    }, 
    singleton: true
))
```

以下是可用的绑定 DTO：

### Alias (别名)

这个简化的 DTO 允许您在容器内创建到另一个绑定的链接。

```php
use Spiral\Core\Config\Alias;

$container->bind(
    \DatetimeImmutable::class, 
    static fn() => new \DatetimeImmutable()
);

$container->bind(
    'now', 
    new Alias(alias: \DatetimeImmutable::class)
);
```

在此示例中，我们首先将 `\DatetimeImmutable` 类绑定到一个工厂函数，该函数在每次请求时创建一个新的 `\DatetimeImmutable` 实例。然后，我们创建一个名为 now 的 Alias 绑定，并将其与 `\DatetimeImmutable` 类绑定关联。

现在，您可以通过别名从容器中请求 `\DatetimeImmutable` 类 `$container->get('now')`

Alias 还提供了第二个参数 - `singleton`。通过将 singleton 设置为 `true`，别名类将成为一个单例。这意味着每当您从容器中请求 `$container->get('now')` 时，它将始终返回相同的实例。另一方面，调用 `$container->get(\DatetimeImmutable::class)` 将在每次请求时返回一个带有当前时间的新的 `\DatetimeImmutable` 实例。

### Autowire (自动装配)

`Spiral\Core\Config\Autowire` 绑定充当 `Spiral\Core\Container\Autowire` 类的包装器，提供了一种通过注入类的依赖项来自动解析和实例化类的方法。

```php
use Spiral\Core\Config\Autowire;
use Spiral\Core\Container\Autowire as AutowireAlias;

$container->bind(MyClass::class, new Autowire(
    autowire: new AutowireAlias(MyClass::class, ['foo' => 'bar']),
    singleton: true
));
```

在此示例中，`singleton` 参数设置为 true，表明容器应将 `MyClass` 视为单例。当您从容器中请求 MyClass 的实例 `$container->get(MyClass::class)` 时，它将始终返回相同的实例。

### Factory (工厂)

`Spiral\Core\Config\Factory` 绑定充当使用闭包创建混合类型的简单工厂。

```php
use Spiral\Core\Config\Factory;

$container->bind('time', new Factory(
    callable: static fn() => time(),
));
```

每次您从容器中请求当前时间 `$container->get('time')` 时，它将返回当前时间戳。

此外，`Factory` 绑定可以配置为单例：

```php
$container->bind('time', new Factory(
    callable: static fn() => time(),
    singleton: true,
));
```

通过将 `singleton` 参数设置为 `true`。这意味着，每当您从容器中请求时间值 `$container->get('time')` 时，它将在多次调用中返回相同的值。

### DeferredFactory (延迟工厂)

`Spiral\Core\Config\DeferredFactory` 绑定允许您使用特殊的数组可调用对象将延迟工厂绑定到容器。

```php
use Spiral\Core\Config\DeferredFactory;

$container->bind('some-binding', new DeferredFactory(
    factory: [MyClass::class, 'handle'],
));
```

在上面的示例中，我们将键 some-binding 绑定到 `DeferredFactory` 实例。factory 属性设置为 `[MyClass::class, 'handle']`。当从容器 `$container->get('some-binding')` 请求 some-binding 键时，`DeferredFactory` 将解析 `MyClass` 的一个实例，然后调用该实例的 handle 方法以产生所需的值。

此外，绑定可以配置为单例：

```php
$container->bind('some-binding', new DeferredFactory(
    factory: ...,
    singleton: true
));
```

### Scalar (标量)

Scalar 绑定是为容器引入的新功能。它提供了一种方便的方式在容器内存储和检索静态标量值。这对于配置路径、常量或应用程序需要的任何其他标量值很有用。

```php
use Spiral\Core\Config\Scalar;

$container->bind('app-path', new Scalar(value: '/var/www/my-app'));
```

### Shared (共享)

Shared 绑定允许您将持久对象绑定到容器中的一个键。一旦创建了该对象，每次请求该键时都将重用该对象，并且在后续请求期间无法提供自定义参数。

```php
use Spiral\Core\Config\Shared;

$container->bind(MyClass::class, new Shared(value: new MyClass(...)));
```

重要的是要注意，对于 `Shared` 绑定，在后续请求期间无法提供自定义参数。该对象将使用初始参数创建并按原样重用。

当您希望确保在整个应用程序中使用同一对象实例时，这特别有用。它提供了持久性，并防止创建具有不同参数的多个实例。

### Inflector (变形器)

变形器允许您在容器中创建对象后对其进行操作。这对于将常见修改或注入应用于特定类型的对象特别有用。

```php
use Spiral\Core\Config\Inflector;

$container->bind(LoggerAwareInterface::class, new Inflector(
    inflector: static function (LoggerAwareInterface $obj, LoggerInterface $logger): LoggerAwareInterface {
        $obj->setLogger($logger);

        return $obj;
    }
));
```

在这种情况下，任何实现 `LoggerAwareInterface` 的对象都将根据指定的配置设置其记录器。

Inflector 绑定是一个强大的工具，用于在应用程序中的对象之间一致地应用常见修改或注入。它简化了配置和自定义从容器检索的对象的流程。

### WeakReference (弱引用)

WeakReference 功能允许您在容器中使用弱引用对象。弱引用是指向对象的引用，当没有对该对象的强引用时，它们不会阻止该对象被垃圾回收。

```php
se Spiral\Core\Config\WeakReference;

$obj = new MyClass();

$container->bind(MyClass::class, new WeakReference(
    reference: new \WeakReference($obj)
));

$obj === $container->get(MyClass::class); // true

unset($obj);

$obj1 = $container->get(MyClass::class); // 将创建一个新对象
$obj1 === $container->get(MyClass::class); // true
```

当您从容器中检索 MyClass `$container->get(MyClass::class)` 时，容器会返回原始对象，因为它仍然存在。但是，当您取消设置 `$obj` 变量时，删除对该对象的强引用，它就有资格进行垃圾回收。对 `$container->get(MyClass::class)` 的后续调用将创建一个新的 `MyClass` 实例，因为原始对象已被垃圾回收。

在某些情况下，使用弱引用可能是有益的，在这种情况下，您希望控制对象的生命周期，并允许在没有更多对它的强引用时对其进行垃圾回收。

## 惰性单例

该框架还允许您使用“惰性单例”，这些类会自动被容器视为单例，而无需显式地将它们绑定为单例。

:::: tabs

::: tab Attribute (特性)
`Spiral\Core\Attribute\Singleton` 允许您将一个类标记为单例。通过将此特性应用于一个类，您指示只应创建该类的一个实例并在整个应用程序中共享。此特性可以用作指定单例行为的接口的替代方法。

```php
use Spiral\Core\Attribute\Singleton;

#[Singleton]
final class UserService
{
    public function store(User $user): void
    {
        //...
    }
}
```

> **注意**
> 在 [容器 - 特性](./attributes.md) 部分阅读有关容器特性的更多信息。

:::

::: tab SingletonInterface (单例接口)

要使用此功能，您可以简单地在您的类中实现 `Spiral\Core\Container\SingletonInterface` 接口，如下所示：

```php
use Spiral\Core\Container\SingletonInterface;

final class UserService implements SingletonInterface
{
    public function store(User $user): void
    {
        //...
    }
}
```

:::
::::

现在，容器将自动将此类视为单例，并且只为整个应用程序创建一个实例，无论请求多少次。

```php
protected function index(UserService $service): void
{
    dump($this->container->get(UserService::class) === $service);
}
```
