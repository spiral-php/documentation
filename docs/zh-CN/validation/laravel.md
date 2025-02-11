# 验证 (Validation) — Laravel 验证器 (Validator)

Spiral 提供了验证组件，允许你使用 [Laravel 验证桥 (Validation bridge)](https://github.com/spiral-packages/laravel-validator) 包来验证数据。这个验证桥提供了与 Laravel 验证器 (Validator) 的集成，Laravel 验证器是 Laravel 框架中使用的验证组件。

> **查看更多 (See more)**
> 在 [验证 (Validation)](factory.md) 章节中阅读更多关于验证的内容。

## 安装 (Installation)

要安装该组件，请运行以下命令：

```terminal
composer require spiral-packages/laravel-validator
```

要启用该组件，你只需要将 `Spiral\Validation\Laravel\Bootloader\ValidatorBootloader` 添加到启动器 (bootloaders) 列表中，该列表位于你的应用程序的类中。

:::: tabs

::: tab 使用方法 (Using method)

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Validation\Laravel\Bootloader\ValidatorBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动器 (Framework — Bootloaders)](../framework/bootloaders.md) 章节中阅读更多关于启动器的内容。
:::

::: tab 使用常量 (Using constant)

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Validation\Laravel\Bootloader\ValidatorBootloader::class,
    // ...
];
```

在 [框架 — 启动器 (Framework — Bootloaders)](../framework/bootloaders.md) 章节中阅读更多关于启动器的内容。
:::

::::

## 使用 (Usage)

当验证组件在你的应用程序中启用时，它会将其自身注册为 `\Spiral\Validation\Laravel\FilterDefinition` 类中的验证名称，并可用于 Spiral 框架的验证组件。

你可以使用 `Spiral\Validation\ValidatorInterface` 接口来访问验证器并执行验证任务。或者，你可以使用 `Spiral\Validation\ValidationProviderInterface` 接口通过其类名来访问验证器。

```php
use Spiral\Http\Request\InputManager;
use Spiral\Validation\ValidationProviderInterface;

class UserController
{
    public function create(InputManager $input, ValidationProviderInterface $provider)
    {
        $validator = $provider->getValidation(\Spiral\Validation\Laravel\FilterDefinition::class)
            ->validate(...);
    }
}
```

## 过滤器 (Filters)

`spiral/filters` 组件是用于验证 Spiral 中 HTTP 请求数据的工具。它允许你创建一个 "过滤器 (Filter)" 对象，该对象定义了应该从请求对象中提取并映射到过滤器对象属性的所需数据。

> **查看更多 (See more)**
> 在 [过滤器 — 过滤器对象 (Filters — Filter object)](../filters/filter.md) 章节中阅读更多关于过滤器的内容。

### 使用属性的过滤器 (Filter with attributes)

实现请求字段映射的一种方法是使用 PHP 属性。这允许你指定应该将哪个请求字段映射到每个过滤器属性。

这是一个带有属性的过滤器对象的示例：

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validation\Laravel\FilterDefinition;
use Spiral\Validation\Laravel\Attribute\Input\File;
use Symfony\Component\HttpFoundation\File\UploadedFile;

final class CreatePostFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $title;

    #[Post]
    public string $slug;

    #[Post]
    public int $sort;

    #[File]
    public UploadedFile $image;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'title' => 'string|required|min:5',
            'slug' => 'string|required|min:5',
            'sort' => 'integer|required',
            'image' => 'required|image'
        ]);
    }
}
```

通过实现 `Spiral\Filters\Model\HasFilterDefinition` 接口，你可以指定应该应用于过滤器对象中包含的数据的验证规则。然后，当使用过滤器对象时，验证组件将使用这些规则来验证数据。

> **注意 (Note)**
> 验证规则在官方的 [Laravel 文档](https://laravel.com/docs/9.x/validation#available-validation-rules) 中进行了描述。

### 使用数组映射的过滤器 (Filter with array mapping)

如果你更喜欢使用数组来配置字段映射，你可以在 `filterDefinition` 方法中定义字段映射。

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validation\Laravel\FilterDefinition;

final class CreatePostFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'title' => 'string|required|min:5',
            'slug' => 'string|required|min:5',
            'sort' => 'integer|required',
            'image' => 'required|image'
        ], [
            'title' => 'title',
            'slug' => 'slug',
            'sort' => 'sort',
            'image' => 'symfony-file:image'
        ]);
    }
}
```
