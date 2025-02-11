# 验证 — Symfony 验证器

Spiral 提供了一个验证组件，允许你使用 [Symfony Validation bridge](https://github.com/spiral-packages/symfony-validator) 包来验证数据。 这个验证桥接器提供了与 [Symfony Validator](https://github.com/symfony/validator) 组件的集成，后者是一个功能更强大、特性更丰富的验证库。

> **另请参阅**
> 可以在 [验证](factory.md) 部分阅读更多关于验证的内容。

## 安装

要安装该组件，运行以下命令：

```terminal
composer require spiral-packages/symfony-validator
```

要启用该组件，你只需要将 `Spiral\Validation\Symfony\Bootloader\ValidatorBootloader` 添加到启动加载器列表中，该列表位于你的应用程序的类中。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Validation\Symfony\Bootloader\ValidatorBootloader::class,
        // ...
    ];
}
```

可以在 [框架 — 启动加载器](../framework/bootloaders.md) 部分阅读更多关于启动加载器的内容。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Validation\Symfony\Bootloader\ValidatorBootloader::class,
    // ...
];
```

可以在 [框架 — 启动加载器](../framework/bootloaders.md) 部分阅读更多关于启动加载器的内容。
:::

::::

## 使用

当验证组件在你的应用程序中启用后，它将以 `\Spiral\Validation\Symfony\FilterDefinition` 类作为验证名称注册自己，并可用于 Spiral 框架验证组件。

你可以使用 `Spiral\Validator\ValidatorInterface` 接口来访问验证器并执行验证任务。 或者，你可以使用 `Spiral\Validation\ValidationProviderInterface` 接口通过其类名访问验证器。

```php
use Spiral\Http\Request\InputManager;
use Spiral\Validation\ValidationProviderInterface;

class UserController
{
    public function create(InputManager $input, ValidationProviderInterface $provider)
    {
        $validator = $provider->getValidation(\Spiral\Validation\Symfony\FilterDefinition::class)
            ->validate(...);
    }
}
```

## 过滤器

`spiral/filters` 组件是一个用于验证 Spiral 中 HTTP 请求数据的工具。它允许你创建一个 "过滤器" 对象，该对象定义了应该从请求对象中提取并映射到过滤器对象属性的所需数据。

> **另请参阅**
> 可以在 [过滤器 — 过滤器对象](../filters/filter.md) 部分阅读更多关于过滤器的内容。

### 使用属性的过滤器

实现请求字段映射的一种方法是使用 PHP 属性。这允许你指定哪个请求字段应该映射到每个过滤器属性。

这是一个带有属性的过滤器对象的示例：

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Validation\Symfony\Attribute\Input\File;
use Spiral\Validation\Symfony\AttributesFilter;
use Symfony\Component\HttpFoundation\File\UploadedFile;
use Symfony\Component\Validator\Constraints;

final class CreatePostFilter extends AttributesFilter
{
    #[Post]
    #[Constraints\NotBlank]
    #[Constraints\Length(min: 5)]
    public string $title;

    #[Post]
    #[Constraints\NotBlank]
    #[Constraints\Length(min: 5)]
    public string $slug;

    #[Post]
    #[Constraints\NotBlank]
    #[Constraints\Positive]
    public int $sort;
    
    #[File]
    #[Constraints\Image]
    public UploadedFile $image;
}
```

> **注意**
> 所有可用的验证规则都在官方的 [Symfony 文档](https://symfony.com/doc/6.0/validation.html#constraints) 中进行了描述。

### 使用 FilterDefinition 的过滤器

如果你更喜欢使用数组配置验证规则，你可以在 `filterDefinition` 方法中定义字段映射。

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validation\Symfony\FilterDefinition;
use Symfony\Component\Validator\Constraints;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Validation\Symfony\Attribute\Input\File;

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
            'title' => [new Constraints\NotBlank(), new Constraints\Length(min: 5)],
            'slug' => [new Constraints\NotBlank(), new Constraints\Length(min: 5)],
            'sort' => [new Constraints\NotBlank(), new Constraints\Positive()],
            'image' => [new Constraints\Image()],
        ]);
    }
}
```

### 使用数组映射的过滤器

如果你更喜欢使用数组配置字段映射，你可以在 `filterDefinition` 方法中定义字段映射。

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validation\Symfony\FilterDefinition;
use Symfony\Component\Validator\Constraints;

final class CreatePostFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'title' => [new Constraints\NotBlank(), new Constraints\Length(min: 5)],
            'slug' => [new Constraints\NotBlank(), new Constraints\Length(min: 5)],
            'sort' => [new Constraints\NotBlank(), new Constraints\Positive()],
            'image' => [new Constraints\Image()]
        ],
        [
            'title' => 'title',
            'slug' => 'slug',
            'sort' => 'sort',
            'image' => 'symfony-file:image'
        ]);
    }
}
```
