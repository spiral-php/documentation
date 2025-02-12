# 验证 — Spiral 验证器

Spiral 提供了一个验证组件，允许您使用 [spiral/validator](https://github.com/spiral/validator) 包来验证数据。这是一个简单、轻量级的验证器，它提供了一个基于数组的领域特定语言 (DSL) 来构建复杂的验证链。

> **另请参阅**
> 在 [验证](factory.md) 部分阅读更多关于验证的内容。

## 安装和配置

要安装该组件，请运行以下命令：

```terminal
composer require spiral/validator
```

要启用该组件，您只需将 `Spiral\Validator\Bootloader\ValidatorBootloader` 添加到引导程序列表中，该列表位于您的应用程序的类中。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Validator\Bootloader\ValidatorBootloader::class,
        // ...
    ];
}
```

在 [框架 — 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的内容。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Validator\Bootloader\ValidatorBootloader::class,
    // ...
];
```

在 [框架 — 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的内容。
:::

::::

该组件的配置文件应位于 `app/config/validator.php`。您可以在此处注册应用程序所需的所有验证检查器、条件和别名。

```php
use Spiral\Validator;

return [
    // 检查器使用容器解析，并提供在通用名称和类下隔离一些验证规则的能力。您可以随时注册新的检查器，而不会出现任何性能问题。
    'checkers'   => [
        'type'     => Validator\Checker\TypeChecker::class,
        'number'   => Validator\Checker\NumberChecker::class,
        'mixed'    => Validator\Checker\MixedChecker::class,
        'address'  => Validator\Checker\AddressChecker::class,
        'string'   => Validator\Checker\StringChecker::class,
        'file'     => Validator\Checker\FileChecker::class,
        'image'    => Validator\Checker\ImageChecker::class,
        'datetime' => Validator\Checker\DatetimeChecker::class,
        'entity'   => Validator\Checker\EntityChecker::class,
        'array'    => Validator\Checker\ArrayChecker::class,
    ],

    // 启用/禁用验证条件
    'conditions' => [
        'absent'     => Validator\Condition\AbsentCondition::class,
        'present'    => Validator\Condition\PresentCondition::class,
        'anyOf'      => Validator\Condition\AnyOfCondition::class,
        'noneOf'     => Validator\Condition\NoneOfCondition::class,
        'withAny'    => Validator\Condition\WithAnyCondition::class,
        'withoutAny' => Validator\Condition\WithoutAnyCondition::class,
        'withAll'    => Validator\Condition\WithAllCondition::class,
        'withoutAll' => Validator\Condition\WithoutAllCondition::class,
    ],

    // 别名仅用于简化开发人员的工作。
    'aliases'    => [
        'notEmpty'   => 'type::notEmpty',
        'notNull'    => 'type::notNull',
        'required'   => 'type::notEmpty',
        'datetime'   => 'type::datetime',
        'timezone'   => 'type::timezone',
        'bool'       => 'type::boolean',
        'boolean'    => 'type::boolean',
        'arrayOf'    => 'array::of',
        'cardNumber' => 'mixed::cardNumber',
        'regexp'     => 'string::regexp',
        'email'      => 'address::email',
        'url'        => 'address::url',
        'file'       => 'file::exists',
        'uploaded'   => 'file::uploaded',
        'filesize'   => 'file::size',
        'image'      => 'image::valid',
        'array'      => 'is_array',
        'callable'   => 'is_callable',
        'double'     => 'is_double',
        'float'      => 'is_float',
        'int'        => 'is_int',
        'integer'    => 'is_integer',
        'numeric'    => 'is_numeric',
        'long'       => 'is_long',
        'null'       => 'is_null',
        'object'     => 'is_object',
        'real'       => 'is_real',
        'resource'   => 'is_resource',
        'scalar'     => 'is_scalar',
        'string'     => 'is_string',
        'match'      => 'mixed::match',
    ]
];
```

## 用法

当在您的应用程序中启用验证组件时，它将把自己注册为 `\Spiral\Validator\FilterDefinition` 类中的验证名称，并可用于 Spiral 验证组件。

您可以使用 `Spiral\Validator\ValidatorInterface` 接口来访问验证器并执行验证任务。或者，您可以使用 `Spiral\Validation\ValidationProviderInterface` 接口通过其类名访问验证器。

```php
use Spiral\Http\Request\InputManager;
use Spiral\Validation\ValidationProviderInterface;

class UserController
{
    public function create(InputManager $input, ValidationProviderInterface $provider)
    {
        $validator = $provider->getValidation(\Spiral\Validator\FilterDefinition::class)
            ->validate(...);
    }
}
```

## 过滤器

`spiral/filters` 组件是用于在 Spiral 中验证 HTTP 请求数据的工具。它允许您创建“过滤器”对象，该对象定义应从请求对象中提取并映射到过滤器对象属性的所需数据。

> **另请参阅**
> 在 [过滤器 — 过滤器对象](../filters/filter.md) 部分阅读更多关于过滤器的内容。

### 使用属性的过滤器

实现请求字段映射的一种方法是使用 PHP 属性。这允许您指定应将哪个请求字段映射到每个过滤器属性。

这是一个带有属性的过滤器对象的示例：

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;
use Spiral\Validator\Attribute\Input\File;

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
            'title' => ['required', ['string::length', 5]]
            'slug' =>  ['required', ['string::length', 5]]
            'sort' =>  ['required', 'integer']
            'image' => ['required', 'image']
        ]);
    }
}
```

通过实现 `Spiral\Filters\Model\HasFilterDefinition` 接口，您可以指定应应用于过滤器对象中包含的数据的验证规则。然后，当使用过滤器对象时，验证组件将使用这些规则来验证数据。

### 使用数组映射的过滤器

如果您更喜欢使用数组配置字段映射，则可以在 `filterDefinition` 方法中定义字段映射。

```php
namespace App\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

final class CreatePostFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'title' => ['required', ['string::length', 5]]
            'slug' =>  ['required', ['string::length', 5]]
            'sort' =>  ['required', 'integer']
            'image' => ['required', 'image']
        ], [
            'title' => 'title',
            'slug' => 'slug',
            'sort' => 'sort',
            'image' => 'symfony-file:image',
        ]);
    }
}
```

## 验证 DSL

默认的 Spiral 验证器接受嵌套数组形式的验证规则。键是要验证的 `property` 的 `name`，而值是 `规则数组`，这些规则将按顺序应用于该值：

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            'notEmpty', // key must not be empty
            'string'    // must be string
        ]
    ]
);

if (!$validator->isValid()) {
    dump($validator->getErrors());
}
```

在这种情况下，规则是检查器方法的名称或任何可用的 PHP 函数，它可以接受 `value` 作为第一个参数。

例如，我们可以在您的规则内直接使用 `is_numeric`：

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            'notEmpty',  // key must not be empty
            'is_numeric' // must be numeric
        ]
    ]
);
```

### 扩展声明

在许多情况下，您需要声明附加的规则参数、条件或自定义错误消息。为此，请将规则声明包装到数组 (`[]`) 中。

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            ['notEmpty'],  // key must not be empty
            ['is_numeric'] // must be numeric
        ]
    ]
);
```

> **注意**
> 如果规则不需要任何参数，您可以省略 `[]`。

### 检查器规则

您可以使用 `::` 前缀拆分您的规则名称，其中第一部分是检查器名称，第二部分是方法名称：

例如，我们获取 `Spiral\Validator\Checker\FileChecker` 检查器：

```php
final class FileChecker extends AbstractChecker
{
    // ...
    public function exists(mixed $file): bool // -> file::exists rule
    {
        return // check if the given file exists;
    }
    
    public function uploaded(mixed $file): bool // -> file::uploaded rule
    {
        return // check if the given file uploaded;
    }
    
    public function size(mixed $file, int $size): bool // -> file::size rule
    {
        return // check the given file size;
    }
}
```

在 `app/config/validator.php` 配置文件中注册它：

```php app/config/validator.php
use Spiral\Validator;

return [
    'checkers' => [
        'file' => Validator\Checker\FileChecker::class,
    ],

    // 如果您需要简化开发人员的工作，请注册别名。
    'aliases' => [
        'file' => 'file::exists',
        'uploaded' => 'file::uploaded',
        'filesize' => 'file::size',
    ]
];
```

并使用验证规则来验证文件：

```php
$validator = $validation->validate(
    ['file' => null],
    [
        'file' => [
            'file::uploaded', // you can use alias 'uploaded'
            ['file::size', 1024] // FileChecker::size($file, 1024)
        ]
    ]
);
```

### 参数

规则数组中列出的所有值都将作为规则参数传递。例如，要使用 `in_array` 检查值：

```php
$validator = $validation->validate(
    ['name' => 'f'],
    [
        'name' => [
            'notEmpty',
            ['in_array', ['a', 'b', 'c'], true] // in_array($value, ['a', 'b', 'c'], true)
        ]
    ]
);
```

指定正则表达式模式：

```php
$validator = $validation->validate(
    ['name' => 'b'],
    [
        'name' => [
            'notEmpty',
            ['regexp', '/^a+$/'] // aaa...
        ]
    ]
);
```

### 错误消息

验证器将为任何自定义规则呈现默认错误消息，要设置自定义错误消息，请设置规则属性：

```php
$validator = $validation->validate(
    ['file' => 'b'],
    [
        'file' => [
            'notEmpty',
            ['regexp', '/^a+$/', 'error' => 'Invalid pattern, "a+" wanted.'] // aaa...
        ]
    ]
);
```

> **注意**
> 您可以将自定义错误消息分配给任何规则。

#### 错误消息本地化

自定义错误消息将自动翻译。

```php
// app/locale/ru/messages.php
return [
    'This value is required.' => 'Значение не должно быть пустым.',
];
```

```php
$translator->setLocale('ru');

$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            ['notEmpty', 'error' => 'This value is required.'] // Will return ['key' => 'Значение не должно быть пустым.']
        ]
    ]
);
```

### 条件

在某些情况下，规则必须仅基于某些外部条件激活，为此使用规则属性 `if`：

```php
$validator = $validation->validate(
    [
        'password' => '',
        'confirmPassword' => ''
    ],
    [
        'password' => [
            ['notEmpty']
        ],
        'confirmPassword' => [
            ['notEmpty', 'if' => ['withAll' => ['password']]]
        ]
    ]
);
```

> **注意**
> 在示例中，如果 `password` 不为空，将显示 `confirmPassword` 上的 required 错误。

您可以使用多个条件或将它们与复杂规则结合起来：

```php
 $validator = $validation->validate(
    [
        'password'        => 'abc',
        'confirmPassword' => 'cde'
    ],
    [
        'password'        => [
            ['notEmpty']
        ],
        'confirmPassword' => [
            ['notEmpty', 'if' => ['withAll' => ['password']]],
            ['match', 'password', 'error' => 'Passwords do not match.']
        ]
    ]
);
```

有两个组合条件：`anyOf` 和 `noneOf`，它们包含嵌套条件：

```php
 $validator = $validation->validate(
    [
        'password'        => 'abc',
        'confirmPassword' => 'cde'
    ],
    [
        'password'        => [
            ['notEmpty']
        ],
        'confirmPassword' => [
            ['notEmpty', 'if' => ['anyOf' => ['withAll' => ['password'], 'withoutAll' => ['otherField']]]],
            [
                'match', 
                'password',
                'error' => 'Passwords do not match.',
                'if' => ['noneOf' => ['some condition', 'another condition']]
            ]
        ]
    ]
);
```

### 可用条件

以下条件可供使用：

| 名称       | 选项   | 描述                                        |
|------------|--------|---------------------------------------------|
| withAny    | *array* | 当至少一个字段不为空时。                    |
| withoutAny | *array* | 当至少一个字段为空时。                    |
| withAll    | *array* | 当所有字段都不为空时。                    |
| withoutAll | *array* | 当所有字段都为空时。                      |
| present    | *array* | 当所有字段都显示在请求中时。                |
| absent     | *array* | 当所有字段都未显示在请求中时。              |
| noneOf     | *array* | 当未满足任何嵌套条件时。                     |
| anyOf      | *array* | 当满足任何嵌套条件时。                      |

> **注意**
> 您可以使用 `Spiral\Validator\ConditionInterface` 创建您的条件。

## 验证规则

可以使用以下验证规则。

> **注意**
> 您可以使用 `Spiral\Validator\AbstractChecker` 或 `Spiral\Validator\CheckerInterface` 创建自己的验证规则。

### 规则别名

最常用的规则集通过一组快捷方式提供：

| 别名       | 规则                 |
|------------|----------------------|
| notEmpty   | type::notEmpty       |
| required   | type::notEmpty       |
| datetime   | datetime::valid      |
| timezone   | datetime::timezone   |
| bool       | type::boolean        |
| boolean    | type::boolean        |
| arrayOf    | array::of           |
| cardNumber | mixed::cardNumber    |
| regexp     | string::regexp       |
| email      | address::email       |
| url        | address::url         |
| file       | file::exists         |
| uploaded   | file::uploaded       |
| filesize   | file::size           |
| image      | image::valid         |
| array      | is_array             |
| callable   | is_callable          |
| double     | is_double          |
| float      | is_float          |
| int        | is_int               |
| integer    | is_integer           |
| numeric    | is_numeric           |
| long       | is_long              |
| null       | is_null              |
| object     | is_object            |
| real       | is_real            |
| resource   | is_resource          |
| scalar     | is_scalar            |
| string     | is_string            |
| match      | mixed::match         |

### 类型

> **注意**
> 前缀 `type::`

| 规则       | 参数                 | 描述                                                   |
|------------|----------------------|--------------------------------------------------------|
| notEmpty   | asString:*bool* - true  | 该值不应为空（与 `!empty` 相同）。                      |
| notNull    | ---                    | 该值不应为 null。                                      |
| boolean    | strict:*bool* - false  | 该值必须是布尔值或整数 [0,1]。                          |

> **注意**
> 此检查器的所有规则都可用，无需前缀。

### Required（必填）

| 规则       | 参数                 | 描述                               |
|------------|----------------------|------------------------------------|
| notEmpty   | asString:*bool* - true  | 该值不应为空。                     |

示例：

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'name' => ['required', 'my::abc']
        ]);
    }
}
```

### Mixed（混合）

> **注意**
> 前缀 `mixed::`

| 规则       | 参数                            | 描述                                                                                                |
|------------|---------------------------------------|-----------------------------------------------------------------------------------------------------|
| cardNumber | ---                                   | 检查通过 Luhn 算法传递的信用卡。                                                                   |
| match      | field:*string*, strict:*bool* - false | 检查该值是否与来自另一个字段的值匹配。                                                                |
| accepted   | ---                                   | 检查该值是否被接受（包含以下值之一 `'yes', true, 1, '1','on'`）                                   |
| declined   | ---                                   | 检查该值是否未被接受（包含以下值之一 `'no', false, 0, '0','off'`）                                 |

> **注意**
> 此检查器的 `cardNumber` 和 `match` 规则无需前缀即可使用。

### Address（地址）

> **注意**
> 前缀 `address::`

| 规则  | 参数                                              | 描述                                 |
|-------|---------------------------------------------------------|--------------------------------------|
| email | ---                                                     | 检查电子邮件是否有效。                     |
| url   | schemas:*?array* - null, defaultSchema:*?string* - null | 检查 URL 是否有效。                       |
| uri   | ---                                                     | 检查 URI 是否有效。                       |

> **注意**
> `email` 和 `url` 规则可通过别名无需 `address` 前缀，对于 `uri` 使用 `address::uri`。

### Number（数字）

> **注意**
> 前缀 `number::`

| 规则   | 参数                 | 描述                                                   |
|--------|----------------------------|--------------------------------------------------------|
| range  | begin:*float*, end:*float* | 检查该数字是否在指定的范围内。                       |
| higher | limit:*float*              | 检查该值是否大于或等于指定值。                       |
| lower  | limit:*float*              | 检查该值是否小于或等于指定值。                       |

### String（字符串）

> **注意**
> 前缀 `string::`

| 规则     | 参数              | 描述                                                             |
|----------|-------------------------|-------------------------------------------------------------------------|
| regexp   | expression:*string*     | 使用正则表达式检查字符串。                                                  |
| shorter  | length:*int*            | 检查字符串长度是否短于或等于指定值。                                  |
| longer   | length:*int*            | 检查字符串长度是否长于或等于指定值。                                  |
| length   | length:*int*            | 检查字符串长度是否等于指定值。                                        |
| range    | left:*int*, right:*int* | 检查字符串长度是否在指定范围内。                                  |
| empty    | *string*                | 检查字符串是否为空。                                                    |
| notEmpty | *string*                | 检查字符串是否不为空。                                                  |

**示例：**

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'name' => ['required', ['string::length', 5]]
        ]);
    }
}
```

### Array（数组）

> 前缀 `array::`

| 规则           | 参数           | 描述                                                             |
|----------------|----------------------|-------------------------------------------------------------------------|
| count          | length:*int*         | 检查数组的大小是否等于给定的值。                                      |
| shorter        | length:*int*         | 检查数组的大小是否小于或等于给定的值。                                |
| longer         | length:*int*         | 检查数组的大小是否大于或等于给定的值。                                |
| range          | min:*int*, max:*int* | 检查数组的大小是否在给定的最小值和最大值之间。                          |
| expectedValues | *array*              | 检查数组值是否包含在给定的值列表中。                                  |
| isList         | -                    | 检查数组是否为列表。                                                    |
| isAssoc        | -                    | 检查数组是否为关联数组。                                                  |

**示例：**

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyRequest extends Filter implements HasFilterDefinition
{
    #[Post]
    public array $tags;
    
    #[Post]
    public array $person = [];
    
    #[Post]
    public array $settings = [];
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'tags' => [
                ['notEmpty'],
                ['array::range', 1, 10]
            ],
            'person' => [  // <==== Request: ['ugly', 'old', 'long_hair']
                ['array::isList'],
                ['array::shorter', 3],
                ['array::expectedValues', ['good', 'bad', 'ugly', 'long_hair', 'young', 'old', 'strong']]
            ],
            'settings' => [  // <====== Request ['setting1' => 'value', 'setting2' => 'value']
                ['array::isAssoc'],
            ]
        ]);
    }
}
```

### File Checker（文件检查器）

> **注意**
> 前缀 `file::`

文件检查器完全支持以字符串形式或使用 `UploadedFileInterface` (PSR-7) 提供的文件名。

| 规则      | 参数         | 描述                                                                                   |
|-----------|--------------------|-----------------------------------------------------------------------------------------------|
| exists    | ---                | 检查文件是否存在。                                                                     |
| uploaded  | ---                | 检查文件是否已上传。                                                                   |
| size      | size:*int*         | 检查文件大小是否小于指定的 KB 值。                                                         |
| extension | extensions:*array* | 检查文件扩展名是否在白名单中。将使用上传文件的客户端名称！                                               |

### Image Checker（图像检查器）

> **注意**
> 前缀 `image::`

图像检查器扩展了文件检查器并完全支持其功能。

| 规则    | 参数                | 描述                                                                            |
|---------|---------------------------|----------------------------------------------------------------------------------------|
| type    | types:*array*             | 检查图像是否在允许的图像类型列表中。                                                      |
| valid   | ---                       | 用于检查图像是否具有允许的类型（允许 JPEG、PNG 和 GIF）的快捷方式。                                |
| smaller | width:*int*, height:*int* | 检查图像是否小于指定的形状（高度检查可选）。                                           |
| bigger  | width:*int*, height:*int* | 检查图像是否大于指定的形状（高度检查可选）。                                            |

### Datetime（日期时间）

> **注意**
> 前缀 `datetime::`

此检查器可以在构造函数中应用 `now` 值

| 规则     | 参数                                                                      | 描述                                                                  |
|----------|---------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| future   | orNow:*bool* - false,<br/>useMicroSeconds:*bool* - false                        | 该值必须是未来的日期。                                                        |
| past     | orNow:*bool* - false,<br/>useMicroSeconds:*bool* - false                        | 该值必须是过去的日期。                                                        |
| format   | format:*string*                                                                 | 该值应与指定的日期格式匹配。                                                    |
| before   | field:*string*,<br/>orEquals:*bool* - false,<br/>useMicroSeconds:*bool* - false | 该值应在给定的阈值之前。                                                      |
| after    | field:*string*,<br/>orEquals:*bool* - false,<br/>useMicroSeconds:*bool* - false | 该值应在给定的阈值之后。                                                      |
| valid    | ---                                                                             | 该值必须是有效的日期时间定义，包括数字时间戳。                                  |
| timezone | ---                                                                             | 该值必须是有效的时区。                                                        |

> **注意**
> 将 `useMicroSeconds` 设置为 true 允许检查带有微秒的日期时间。<br/>
> 小心，两个 `new \DateTime('now')` 对象将 99% 具有不同的微秒值，因此它们永远不会相等。

## 自定义验证规则

可以通过自定义检查器实现创建特定于应用程序的验证规则。

```php
namespace App\Security;

use Cycle\Database\Database;
use Spiral\Validator\AbstractChecker;

class DBChecker extends AbstractChecker
{
    public const MESSAGES = [
        // Method => Error message
        'user' => 'No such user.'
    ];

    public function user(int $id): bool
    {
        return $this->db->table('users')->select()->where('id', $id)->count() === 1;
    }
}
```

> **注意**
> 使用预构建的常量 `MESSAGES` 来定义自定义错误模板。

要激活检查器，请在 `ValidationBootloader` 中注册它：

```php
namespace App\Bootloader;

use App\Security\DBChecker;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Validator\Bootloader\ValidatorBootloader;

class CheckerBootloader extends Bootloader
{
    public function boot(ValidationBootloader $validation): void
    {
        // 注册自定义检查器
        $validation->addChecker('db', DBChecker::class);
        
        // 注册检查器的别名
        $validation->addAlias('db_user', 'db::user');
    }
}
```

您现在可以通过 `db::user`（或别名 `db_user`）规则使用验证。
