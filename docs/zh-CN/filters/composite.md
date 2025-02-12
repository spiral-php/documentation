# 过滤器 — 复合过滤器

Spiral 允许创建复合过滤器，这些过滤器由其他过滤器组成。

让我们设想一下，我们有一个 `AddressFilter`，它代表单个地址，可以用作配置文件的组成部分，并有自己的验证规则：

> **注意**
> 在我们的示例中，我们将使用 [Spiral 验证器](../validation/spiral.md) 进行验证，但您可以使用任何其他验证库。

```php app/src/Endpoint/Web/Filter/AddressFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class AddressFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $city;

    #[Post]
    public string $address;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(validationRules: [
            'city' => ['required', 'string'],
            'address' => ['required', 'string'],
        ]);
    }
}
```

Spiral 提供了两种类型的嵌套过滤器：

## 嵌套过滤器

Spiral 允许您通过在父过滤器类中嵌套其他过滤器来创建复合过滤器。这是通过在父过滤器类中声明一个属性，并使用 `Spiral\Filters\attribute\NestedFilter` 属性来装饰它来完成的。该属性接受一个类参数，该参数设置为子过滤器类。这允许父过滤器以嵌套格式接受输入数据，其中使用此属性装饰的属性包含子过滤器的数据。这使得在多个级别验证和过滤数据以及在不同的父过滤器中重用子过滤器变得容易。

```php app/src/Endpoint/Web/Filter/ProfileFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\NestedFilter;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class ProfileFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;

    #[NestedFilter(class: AddressFilter::class)]
    public AddressFilter $address;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(validationRules: [
            'name' => ['required', 'string'],
        ]);
    }
}
```

此过滤器将接受以下格式的数据：

```json Post data
{
  "name": "Antony",
  "address": {
    "city": "San Francisco",
    "address": "Address"
  }
}
```

将请求输入数据传递给过滤器后，您可以使用过滤器类的属性访问已过滤的数据。

```php
public function index(ProfileFilter $profile): void
{
    dump($profile->address->city); // San Francisco
}
```

当使用嵌套过滤器时，父过滤器和子过滤器将一起进行验证。如果子过滤器中存在任何验证错误，它们将被挂载到父过滤器错误的子数组中。这允许您轻松识别哪些错误属于哪个过滤器，并且更易于向用户显示错误。

```json Validation errors
{
  "name": "This field is required.",
  "address": {
    "city": "This field is required."
  }
}
```

### 自定义前缀

`NestedFilter` 属性允许您为传递给子过滤器的数据指定自定义前缀。默认情况下，前缀与分配给嵌套过滤器属性的键相同，但在某些情况下，您可能需要使用不同的前缀。

```php app/src/Endpoint/Web/Filter/ProfileFilter.php
class ProfileFilter extends Filter implements HasFilterDefinition
{
    #[NestedFilter(class: AddressFilter::class, prefix: 'addr')]
    public AddressFilter $address;
    
    // ...
}
```

与此过滤器一起使用的 json 数据格式为：

```json Post data
{
  "name": "Antony",
  "addr": {
    "city": "San Francisco",
    "address": "Address"
  }
}
```

> **注意**
> 您可以在内部跳过使用 `address` 键，错误将相应地挂载。

### 复合过滤器

您可以使用嵌套子过滤器作为更大的复合过滤器的一部分。 使用前缀 `.` (root) 来实现：

```php app/src/Endpoint/Web/Filter/MultipleAddressesFilter.php
class MultipleAddressesFilter extends Filter implements HasFilterDefinition
{
    #[NestedFilter(class: AddressFilter::class, prefix: '.')]
    public AddressFilter $address;
    
    // ...
}
```

`AddressFilter` 将从顶层接收数据，这意味着您可以发送如下请求：

```json Post data
{
  "name": "Antony",
  "city": "San Francisco",
  "address": "Address"
}
```

## 过滤器数组

您可以使用 `Spiral\Filters\attribute\NestedArray` 属性同时填充过滤器数组。为了使用此属性，您需要在过滤器类中声明一个数组属性，并使用 `NestedArray` 属性装饰它，将数组中每个元素的过滤器类指定为类参数。

该属性中的 `input` 参数用于指定包含过滤器数组的输入源。

> **注意**
> 可用的输入源列表可以在
> [过滤器 — 过滤器对象](../filters/filter.md#available-attributes) 部分中找到。

```php app/src/Endpoint/Web/Filter/MultipleAddressesFilter.php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\NestedArray;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

final class MultipleAddressesFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;

    #[NestedArray(class: AddressFilter::class, input: new Post]
    public array $addresses;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(validationRules: [
            'name' => ['required', 'string'],
        ]);
    }
}
```

此过滤器将接受以下格式的数据：

```json Post data
{
  "key": "value",
  "addresses": [
    {
      "city": "San Francisco",
      "address": "Address"
    },
    {
      "city": "Minsk",
      "address": "Address #2"
    }
  ]
}
```

应用过滤器后，您可以使用数组符号访问数组中的各个过滤器。

```php
public function index(MultipleAddressesFilter $filter)
{
    dump($filter->addresses[0]->city); // San Francisco
    dump($filter->addresses[1]->city); // Minsk
}
```

> **注意**
> 如果嵌套过滤器中存在任何验证错误，它们将根据嵌套过滤器的结构进行挂载。

### 自定义前缀

是的，您可以在定义 `NestedArray` 属性时，将自定义前缀作为参数传递给输入源类的构造函数。通过这种方式，过滤器将使用自定义前缀而不是默认的键名来查找输入数据。

```php app/src/Endpoint/Web/Filter/MultipleAddressesFilter.php
class MultipleAddressesFilter extends Filter
{
    #[NestedArray(class: AddressFilter::class, input: new Post('addr'))]
    public array $addresses;
    
    // ...
}
```

此过滤器支持以下数据格式：

```json
{
  "key": "value",
  "addr": [
    {
      "city": "San Francisco",
      "address": "Address"
    },
    {
      "city": "Minsk",
      "address": "Address #2"
    }
  ]
}
```
