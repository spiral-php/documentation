# 过滤器 — 入门指南

`spiral/filters` 是一个强大的组件，用于过滤和可选地验证输入数据。它允许你为每个输入字段定义一组规则，然后使用这些规则来确保输入数据具有正确的格式，并满足你设置的任何其他要求。你可以使用过滤器来验证来自各种来源的数据，例如 HTTP 请求、gRPC 请求、控制台命令等。

**下面是一个简单的过滤器示例：**

```php app/src/Endpoint/Web/Filter/UserFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

final class UserFilter extends Filter
{
    #[Query(key: 'username')]
    public string $username;
}
```

此示例展示了一个简单的过滤器，可以在控制器动作中被请求，它将自动将来自 HTTP 请求的数据映射到过滤器的属性。你可以选择为每个属性定义一组验证规则，以确保输入数据具有正确的格式。

使用过滤器的其中一个好处是，它有助于将你的输入验证逻辑集中在一个地方。这可以使你的代码更容易维护，因为你不需要在应用程序的多个地方重复验证逻辑。

此外，过滤器可以在应用程序的不同部分重复使用，这有助于减少代码重复，并使管理你的验证逻辑更加容易。

![Filters](https://user-images.githubusercontent.com/773481/211005150-ba8803ed-42c1-40eb-9cf0-7e45e15b0b73.png)
*在 HTTP 层中过滤和验证输入数据的过程的图示*

> **了解更多**
> 在 [Cookbook — Console command input validation](../cookbook/console-validation.md) 部分阅读更多关于如何使用过滤器进行控制台命令验证的内容。

<hr>

## 安装

> **注意**
> 该组件依赖于 [Validation](../validation/factory.md) 组件，如果你想使用验证功能，请务必先阅读它。

该组件不需要任何配置，可以使用启动加载器 `Spiral\Bootloader\Security\FiltersBootloader` 激活：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Security\FiltersBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于启动加载器的内容。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Security\FiltersBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于启动加载器的内容。
:::

::::

## 输入源

过滤器组件使用 `Spiral\Filter\InputInterface` 作为主要数据源：

```php
interface InputInterface
{
    public function withPrefix(string $prefix, bool $add = true): InputInterface;

    public function getValue(string $source, string $name = null);
}
```

默认情况下，此接口绑定到 [InputManager](../http/request-response.md)，并允许使用 **源** 和 **原始** 对（支持点表示法）访问任何请求的属性。

例如：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Filters\InputInterface;

class HomeController
{
    public function index(InputInterface $input): void
    {
        dump($input->getValue('query', 'abc')); // ?abc=1

        // 点表示法
        dump($input->getValue('query', 'a.b.c')); // ?a[b][c]=2

        // 与上面相同
        dump($input->withPrefix('a')->getValue('query', 'b.c')); // ?a[b][c]=2
    }
}
```

输入绑定是将数据从请求传递到过滤器对象的主要方式。

## 创建过滤器

过滤器对象的实现可能因包而异。默认实现通过抽象类 `Spiral\Filters\Model\Filter` 提供。

要创建一个自定义过滤器来验证具有键 `username` 的简单查询值，请使用脚手架命令：

```terminal
php app.php create:filter UserFilter -p username:query
```

> **注意**
> 在 [Basics — Scaffolding](../basics/scaffolding.md#request-filter) 部分阅读更多关于脚手架的内容。

执行此命令后，以下输出将确认创建成功：

```output
Declaration of ' [32mUserFilter [39m' has been successfully written into ' [33mapp/src/Endpoint/Web/Filter/UserFilter.php [39m'.
```

```php app/src/Endpoint/Web/Filter/UserFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

final class UserFilter extends Filter
{
    #[Query(key: 'username')]
    public string $username;
}
```

> **警告**
> 在你的过滤器中使用类型化属性时要小心。过滤器不会执行任何类型转换，如果输入数据与属性类型不匹配，将抛出异常。

你可以将过滤器作为方法注入来请求（它将自动绑定到当前的 HTTP 请求输入）：

```php app/src/Endpoint/Web/UserController.php
namespace App\Endpoint\Web;

class UserController
{
    public function show(Filter\UserFilter $filter): void
    {     
        dump($filter->username);
    }
}
```

## 输入类型转换

对于更高级的场景，需要将数据转换为自定义类型时，过滤器输入转换器就会发挥作用。

### 了解转换器

在深入探讨转换器的应用之前，有必要理解 `Spiral\Filters\Model\Mapper\CasterInterface`。

```php Spiral\Filters\Model\Mapper\CasterInterface
use Spiral\Filters\Model\FilterInterface;

interface CasterInterface
{

    public function supports(\ReflectionNamedType $type): bool;

    public function setValue(FilterInterface $filter, \ReflectionProperty $property, mixed $value): void;
}
```

此接口有两个关键方法：

- **supports:** 接收一个 `$type`，并让你决定是否可以使用此转换器转换该值。
- **setValue:** 将传入的值转换为你想要的类型。

### 可用的转换器

该组件提供了一组开箱即用的转换器：

#### UuidCaster

将字符串转换为 `Ramsey\Uuid\Uuid` 对象。

```php
namespace App\Endpoint\Web\Filter;

use Ramsey\Uuid\UuidInterface;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

final class UserFilter extends Filter
{
    #[Query]
    public UuidInterface $uuid;
}
```

#### EnumCaster

启用将字符串转换为枚举，从而提高类型安全性。

```php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

final class UserFilter extends Filter
{
    #[Query]
    public RoleEnum $role;
}
```

### 自定义转换器

**下面是一个简单的转换器示例：**

考虑一个场景，你正在处理一个 UUID 字符串，并打算将其转换为 UUID 对象。

```php app/src/Filter/Caster/UuidCaster.php
use Spiral\Filters\Model\FilterInterface;
use Spiral\Filters\Model\Mapper\CasterInterface;
use Ramsey\Uuid\UuidInterface;
use Ramsey\Uuid\Uuid;

final class UuidCaster implements CasterInterface
{
    public function supports(\ReflectionNamedType $type): bool
    {
        return $type->getName() === UuidInterface::class;
    }

    public function setValue(FilterInterface $filter, \ReflectionProperty $property, mixed $value): void
    {
        $property->setValue($filter, Uuid::fromString($value));
    }
}
```

> **注意**
> Uuid 转换器已由该组件提供。

要使你的自定义转换器可操作，你需要在应用程序的启动加载器中注册它。

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Filters\Model\Mapper\CasterRegistryInterface;

class AppBootloader extends Bootloader
{
    public function boot(CasterRegistryInterface $casterRegistry)
    {
        $casterRegistry->register(new UuidCaster());
    }
}
```

注册转换器后，每当你定义与转换器支持的类型匹配的属性的过滤器时，Spiral 将自动使用注册的转换器来转换数据。

### 处理类型转换错误

在处理请求数据过滤器时，确保这些参数与预期的数据类型匹配至关重要。例如，如果某个参数应该为 `string`，那么当它以不同的格式（如 `array` 或 `integer`）传入时，就不应处理。 Spiral 框架提供了一个有效的解决方案来优雅地处理这种类型不匹配的情况。

你可以对任何需要类型验证的属性使用 `Spiral\Filters\Attribute\CastingErrorMessage` 属性：

```php app/src/Endpoint/Web/Filter/UserFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Attribute\CastingErrorMessage;

final class UserFilter extends Filter
{
    #[Query(key: 'username')]
    #[CastingErrorMessage('Invalid type')]
    public string $username;
}
```

在这种情况下，`username` 应该为字符串。但是，可能会出现输入数据类型错误的情况，例如 `array` 或 `integer`。在这种情况下，过滤器将捕获异常并返回验证错误消息。

## 验证

默认情况下，过滤器不执行验证。但是，如果你想验证一个过滤器，你可以实现 `HasFilterDefinition` 接口，并使用 `FilterDefinition` 类和 `Spiral\Filters\Model\ShouldBeValidated` 接口实现为过滤器属性定义一组验证规则：

```php app/src/Filter/MyFilterDefinition.php
namespace App\Filter;

use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\ShouldBeValidated;

final class MyFilterDefinition implements FilterDefinitionInterface, ShouldBeValidated
{
    public function __construct(
        private readonly array $validationRules = [],
        private readonly array $mappingSchema = []
    ) {
    }

    public function validationRules(): array
    {
        return $this->validationRules;
    }

    public function mappingSchema(): array
    {
        return $this->mappingSchema;
    }
}
```

这是一个注册过滤器定义并与验证器绑定的示例，该验证器将用于使用 `MyFilterDefinition` 定义验证过滤器：

```php app/src/Application/Bootloader/ValidatorBootloader.php
namespace App\Application\Bootloader;

use App\Validation;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Validation\Bootloader\ValidationBootloader;
use Spiral\Validation\ValidationInterface;
use Spiral\Validation\ValidationProvider;

final class ValidatorBootloader extends Bootloader
{
    public function boot(ValidationProvider $provider): void
    {
        $provider->register(
            \App\Filter\MyFilterDefinition::class,
            static fn(Validation $validation): ValidationInterface => new MyValidation()
        );
    }
}
```

> **注意**
> 在 [here](../validation/factory.md) 阅读更多关于 Validation 组件的内容。

现在你可以在你的过滤器中使用过滤器定义：

```php app/src/Endpoint/Web/Filter/UserFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use App\Filter\MyFilterDefinition;

final class UserFilter extends Filter implements HasFilterDefinition
{
    #[Query]
    public string $username;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new MyFilterDefinition([
            'username' => ['string', 'required']
        ]);
    }
}
```

> **注意**
> 尝试使用 `?username=john` 的 URL。 `UserFilter` 将在将你的请求传递到控制器之前自动预先验证你的请求。
