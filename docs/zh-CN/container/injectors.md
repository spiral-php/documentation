# 容器 — 注入器

Spiral 框架提供了一种使用注入接口来控制任何接口或抽象类子类的创建过程的方法。

**使用它有几个好处：**

- **灵活性：** 注入接口允许根据特定上下文创建特定类，从而在类的实例化方面提供高度的灵活性。
- **解耦：** 注入接口允许类与其依赖项解耦，使代码更模块化，更易于维护。
- **控制实例化：** 注入接口提供了一种控制类实例化的方法，从而可以轻松管理对象的生命周期并确保它们被正确初始化。

本指南演示了如何创建一个类实例并为其分配一个唯一的值，而无论哪个子类实现了它。

## 类注入器

让我们假设我们有一个接口 `Psr\SimpleCache\CacheInterface`，它提供一个简单的缓存接口。

### 创建一个注入器

注入器类应该实现 `Spiral\Core\Container\InjectorInterface`，它提供一个名为 `createInjection` 的方法。 每次从容器请求特定类时都会使用此方法。

在我们的示例中，我们可以将一个引导程序和一个注入器组合成一个实例：

```php app/src/Application/Bootloader/CacheBootloader.php
namespace App\Application\Bootloader;

use Psr\Container\ContainerInterface;
use Psr\SimpleCache\CacheInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\InjectorInterface;

class CacheBootloader extends Bootloader implements InjectorInterface
{
    public function __construct(
        private readonly ContainerInterface $container,
    ) {}

    public function boot(BinderInterface $binder): void
    {
        // 注册可注入的类
        $binder->bindInjector(CacheInterface::class, self::class);
    }

    public function createInjection(\ReflectionClass $class, string $context = null): CacheInterface
    {
        return match ($context) {
            'redis' => new RedisCache(...),
            'memcached' => new MemcachedCache(...),
            default => new ArrayCache(...),
        };
    }
}
```

> **注意**
> 不要忘记激活引导程序。

### 使用注入器

现在，我们可以使用注入器根据上下文获取特定的缓存实现。

当容器解析 `CacheInterface` 时，它将使用 `createInjection` 方法从注入器请求它。该方法接受两个参数 `$class` 和 `$context`。`$class` 参数返回所请求类的 `ReflectionClass` 对象，而 `$context` 参数返回一个参数或别名名称（例如，请求可注入类的的方法或函数的参数名称）。

以下是如何使用注入器的示例：

```php app/src/Endpoint/Web/BlogController.php
namespace App\Endpoint\Web;

use Psr\SimpleCache\CacheInterface;

class BlogController
{
    public function __construct(
        private readonly CacheInterface $redis,
        private readonly CacheInterface $memcached,
        private readonly CacheInterface $cache,
    ) {
        
        \assert($redis instanceof RedisCache);

        \assert($memcached instanceof MemcachedCache);
        
        \assert($cache instanceof ArrayCache);
    }
}
```

在此示例中，`BlogController` 类接受三个属性 `$redis`，`$memcached` 和 `$cache`。 这些属性都属于 `CacheInterface` 类型，但它们对应于接口的不同实现，具体取决于传递给注入器的 `createInjection` 方法的上下文。

### 类继承

使用注入器可以进行类继承。

> **注意**
> 目前，注入器仅支持扩展基类的类（而不是接口），但未来的 Spiral 版本也将支持接口继承。

```php
abstract class RedisCacheInterface implements CacheInterface
{

}
```

在此示例中，`RedisCacheInterface` 是一个实现 `CacheInterface` 的抽象类。

例如，在 `createInjection` 方法中，我们可以检查所请求的类是否是 `RedisCacheInterface` 的子类，并返回一个 `RedisCache` 实例，或者检查所请求的类是否是 `MemcachedCacheInterface` 的子类，并返回一个 `MemcachedCache` 实例。

```php app/src/Application/Bootloader/CacheBootloader.php
public function createInjection(\ReflectionClass $class, string $context = null): CacheInterface
{
    if ($class->isSubclassOf(RedisCacheInterface::class)) {
        return new RedisCache(...);
    }
    
    return match ($context) {
        'redis' => new RedisCache(...),
        'memcached' => new MemcachedCache(...),
        default => new ArrayCache(...),
    };
}
```

## 枚举注入器

`spiral/boot` 组件提供了一种使用 `Spiral\Boot\Injector\InjectableEnumInterface` 接口解析枚举类的便捷方法。当容器请求一个枚举时，它将调用一个指定的方法来确定该枚举的当前值并注入任何必需的依赖项。

**使用枚举注入有几个好处：**

- **类型安全：** 注入允许类型安全，确保将正确类型的变量传递给方法或类。
- **动态解析：** 注入允许根据应用程序的当前状态动态解析枚举值，这使得更改枚举的值变得容易，而无需手动更新应用程序中的多个地方。
- **可重用性：** 注入可以在应用程序的不同部分重复使用，从而更有效地管理枚举实例的创建。
- **提高可读性：** 注入通过提供代码中使用的枚举实例的清晰含义来使代码更具可读性和自解释性。
- **提高可维护性：** 注入使代码更易于维护，因为变量集中在一个地方，并且可以轻松管理。

让我们创建一个 `Enum`，我们可以轻松地确定应用程序的环境。

```php
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\Injector\ProvideFrom;
use Spiral\Boot\Injector\InjectableEnumInterface;

#[ProvideFrom(method: 'detect')]
enum AppEnvironment: string implements InjectableEnumInterface
{
    case Production = 'prod';
    case Stage = 'stage';
    case Testing = 'testing';
    case Local = 'local';

    public function isProduction(): bool
    {
        return $this === self::Production;
    }

    public function isTesting(): bool
    {
        return $this === self::Testing;
    }

    public function isLocal(): bool
    {
        return $this === self::Local;
    }

    public function isStage(): bool
    {
        return $this === self::Stage;
    }

    public static function detect(EnvironmentInterface $environment): self
    {
        $value = $environment->get('APP_ENV');

        return \is_string($value)
            ? (self::tryFrom($value) ?? self::Local)
            : self::Local;
    }
}
```

`ProvideFrom` 属性用于指定一个 `detect` 方法，该方法用于确定枚举的当前值。当容器请求枚举时，将调用该方法，并且来自容器的任何必需依赖项都将被注入到该方法中。

### 用法

容器将自动注入具有正确值的枚举。

```php app/src/Endpoint/Console/MigrateCommand.php
final class MigrateCommand extends Command 
{
     const NAME = '...';

     public function perform(AppEnvironment $appEnv): int
     {
           if ($appEnv->isProduction()) {
                 // 拒绝
           }
           
           // 执行迁移
     }
}
```
