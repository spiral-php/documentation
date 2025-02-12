# Filters — 过滤器对象

Filter 类代表一组可以被过滤和验证的请求输入字段。它提供了一组创建过滤器的基本特性，例如将请求输入数据绑定到过滤器属性以及访问过滤后的数据的能力。

## 验证器

有三种验证器桥可用于 Spiral 过滤器。你可以根据你的需求和偏好，在你的应用程序中使用这些验证器桥中的任何一个。

### [Spiral 验证器](../validation/spiral.md)

这是默认的验证器桥。它是一个简单、轻量级的验证器，可以处理基本的验证任务。

```php app/src/Endpoint/Web/Filter/CreatePostFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Model\FilterInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\File;

class CreatePostFilter implements FilterInterface, HasFilterDefinition
{
    #[Post(key: 'title')]
    public string $title;
    
    #[Post(key: 'text')]
    public string $text;
    
    #[File]
    public UploadedFile $image;
    
    // ...

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(
            validationRules: [
                'title' => [
                    ['notEmpty'],
                    ['string::length', 50]
                ],
                'text' => [['notEmpty']],
                'image' => [['image::valid'], ['file::size', 1024]]
                
                // ...
            ]
        );
    }
}
```

### [Symfony 验证器](../validation/symfony.md)

此验证器桥提供与 Symfony 验证器组件的集成，这是一个功能更强大、功能更丰富的验证库。

```php app/src/Endpoint/Web/Filter/CreatePostFilter.php
namespace App\Endpoint\Web\Filter;

use Psr\Http\Message\UploadedFileInterface;
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

### [Laravel 验证器](../validation/laravel.md)

此验证器桥提供与 Laravel 验证器的集成，Laravel 验证器是 Laravel 框架中使用的验证组件。

```php app/src/Endpoint/Web/Filter/CreatePostFilter.php
namespace App\Endpoint\Web\Filter;

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

## 用法

所有过滤器对象都应实现 `Spiral\Filters\Model\FilterInterface`。该接口是可注入的，这意味着当你从容器中请求一个过滤器类时，它将被自动创建，并且请求数据将被映射到每个过滤器属性。

> **另请参阅**
> 在 [高级 — 容器注入器](../container/injectors.md) 部分阅读有关注入器的更多信息。

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Model\FilterInterface;
use Spiral\Filters\Attribute\Input\Post;

class MyFilter implements FilterInterface
{
    #[Post(key: 'text')]
    public string $text;
}
```

现在我们可以在我们的应用程序中使用过滤器了。有三种方法可以做到：

:::: tabs

::: tab DI
简单地将过滤器作为依赖项请求，例如，在某个控制器方法中：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use App\Endpoint\Web\Filter\MyFilter;

class HomeController
{
    public function index(MyFilter $filter): void
    {
        dump($filter);
    }
}
```

:::

::: tab 过滤器提供程序
你也可以使用 `Spiral\Filters\Model\FilterProviderInterface->createFilter` 创建一个过滤器实例：

```php
$provider = $container->get(\Spiral\Filters\Model\FilterProviderInterface::class);

$provider->createFilter(
    MyFilter::class, 
    $container->get(\Spiral\Filters\InputInterface::class)
);
```

:::

::: tab 容器
当你从容器中请求一个过滤器时，它将被自动创建，并且请求数据将被映射。

```php
dump($container->get(MyFilter::class)); 
```

:::

::::

## 过滤器模式

有两种方法可以定义过滤器模式：

:::: tabs

::: tab 属性

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Post;

class MyFilter implements FilterInterface
{
    #[Post(key: 'text')]
    public string $text;
}
```

请求数据将被映射到过滤器属性，并且你可以直接访问它们。

```php
public function index(MyFilter $filter): void
{
    dump($filter->text);
}
```

:::

::: tab 数组模式

在这种情况下，过滤器应实现 `Spiral\Filters\Model\HasFilterDefinition` 并扩展 `Spiral\Filters\Model\Filter` 抽象类，以便能够访问来自请求的映射数据。

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(
            mappingSchema: ['text' => 'data:text']
        );
    }
}
```

请求数据将根据 `mappingSchema` 进行映射，并且你可以使用 `getData` 方法访问它们。

```php
public function index(MyFilter $filter): void
{
    dump($filter->getData());
}
```

\
:::

::::

#### 来源

根据设计，你可以使用 [InputManager](../http/request-response.md) 的任何方法作为传递了源参数的来源。以下来源可用：

| 来源           | 描述                                                                    |
|----------------|-------------------------------------------------------------------------|
| uri            | 当前页面 Uri，以 `Psr\Http\Message\UriInterface` 的形式                 |
| path           | 当前页面路径                                                              |
| method         | HTTP 方法（GET，POST，...）                                              |
| isSecure       | 如果使用 https                                                           |
| isAjax         | 如果 `X-Requested-With` 设置为 `xmlhttprequest`                          |
| isJsonExpected | 当客户端期望 `application/json` 时                                       |
| remoteAddress  | 用户 IP 地址                                                            |

> **注意**
> 你可以在你的过滤器对象中使用这两种方法。在这两种情况下，过滤器提供程序都将为具有属性的属性构建一个映射模式，然后将该模式与过滤器定义中的模式合并。

## 属性

要使用请求过滤器属性，你可以使用可用的属性之一来指定属性的数据应来自何处。例如，你可以使用 `Spiral\Filters\Attribute\Input\Query` 属性将查询字符串参数映射到类属性，如下所示：

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
    #[Query(key: 'username')]
    public string $login;
}
```

在此示例中，`Query` 属性会将查询字符串参数的值（键为 `username`）映射到 `login` 属性。

当使用属性时，你可以指定一个 `key` 参数或省略它。如果省略 key 参数，则属性将使用类属性的名称作为 key。

例如：

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
    #[Query]
    public string $login;
}
```

在这种情况下，`Query` 属性会将查询字符串参数的值（键为 `login`）映射到 `login` 属性。

你可以在单个过滤器类中使用多个属性，将请求数据的不同部分映射到不同的类属性。

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Cookie;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
    #[Query(key: 'redirectURL')]
    public string $redirectTo;

    #[Cookie]
    public string $memberCookie;

    #[Post(key: 'user')]
    public string $username;

    #[Post]
    public string $password;

    #[Post(key: 'remember')]
    public string $rememberMe;
}
```

通过以这种方式使用多个属性，你可以轻松地提取并将请求数据的不同部分映射到适当的类属性。

### 可用属性

以下是可用的请求过滤器属性的列表：

| 属性                                              | 描述                                                                                                                                                                     |
|---------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Spiral\Filters\Attribute\Input\Post              | **POST** 数据中的值。                                                                                                                                                       |
| Spiral\Filters\Attribute\Input\Query             | 查询字符串中的值                                                                                                                                                            |
| Spiral\Filters\Attribute\Input\Input             | POST 数据或查询字符串中的值。                                                                                                                                                |
| Spiral\Filters\Attribute\Input\Data              | 任何请求数据（POST、查询字符串等）中的值。                                                                                                                                    |
| Spiral\Filters\Attribute\Input\File              | 上传的文件。它将返回 `Psr\Http\Message\UploadedFileInterface` 对象或 `null`。                                                                                               |
| Spiral\Filters\Attribute\Input\Cookie            | cookie 中的值。                                                                                                                                                             |
| Spiral\Filters\Attribute\Input\Header            | 来自请求头的值。                                                                                                                                                          |
| Spiral\Filters\Attribute\Input\IsAjax            | 一个布尔值，指示是否使用将 `X-Requested-With` 标头设置为 `xmlhttprequest` 的方式发出了请求。                                                                             |
| Spiral\Filters\Attribute\Input\IsJsonExpected    | 一个布尔值，指示客户端是否期望 `application/json` 响应。                                                                                                                     |
| Spiral\Filters\Attribute\Input\IsSecure          | 一个布尔值，指示是否通过 HTTPS 发出了请求。                                                                                                                                  |
| Spiral\Filters\Attribute\Input\Method            | 请求的 HTTP 方法（例如 GET、POST 等）。                                                                                                                                   |
| Spiral\Filters\Attribute\Input\Path              | 当前请求路径。                                                                                                                                                           |
| Spiral\Filters\Attribute\Input\RemoteAddress     | 客户端的 IP 地址。                                                                                                                                                          |
| Spiral\Filters\Attribute\Input\Route             | 当前路由属性中的值。                                                                                                                                                         |
| Spiral\Filters\Attribute\Input\Server            | 来自请求服务器数据的值。                                                                                                                                                       |
| Spiral\Filters\Attribute\Input\Uri               | 当前页面 URI，以 `Psr\Http\Message\UriInterface` 对象的形式。                                                                                                           |
| Spiral\Filters\Attribute\Input\BearerToken       | `Authorization` 标头的值到类属性。                                                                                                                                            |

#### 路由参数

每个路由都会将匹配的参数写入 `ServerRequestInterface` 属性 `matches` 中，可以在过滤器中访问路由值。

```php
$router->setRoute(
    'sample',
    new Route('/action/<id>.html', new Controller(HomeController::class))
);
```

:::: tabs

::: tab 属性

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Route;
use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
    #[Route(key: 'id')]
    public string $routeId;
}
```

:::

::: tab 数组模式

通过使用 `attribute:matches.{name}` 符号的数组映射：

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(mappingSchema: 
            [
                'routeId' => 'attribute:matches.id'
            ]
        );
    }
}
```

:::

::::

### 创建自定义属性

你可以通过扩展 `Spiral\Filters\Attribute\Input\AbstractInput` 类并实现 `getValue` 和 `getSchema` 方法来创建你自己的自定义请求过滤器属性。这允许你定义自己的数据检索逻辑并指定属性应返回的值的类型。

```php
namespace App\Validation\Attribute;

use Spiral\Attributes\NamedArgumentConstructor;
use Spiral\Filters\Attribute\Input\AbstractInput;
use Spiral\Filters\InputInterface;
use Symfony\Bridge\PsrHttpMessage\Factory\UploadedFile;

#[\Attribute(\Attribute::TARGET_PROPERTY), NamedArgumentConstructor]
final class Base64DecodedQuery extends AbstractInput
{
    /**
     * @param non-empty-string|null $key
     */
    public function __construct(
        public readonly ?string $key = null,
    ) {
    }

    public function getValue(InputInterface $input, \ReflectionProperty $property): ?string
    {
        $value = $input->getValue('query', $this->getKey($property));
        if ($value === null) {
            return null;
        }
        
        return \base64_decode($value);
    }

    public function getSchema(\ReflectionProperty $property): string
    {
        return 'query:' . $this->getKey($property);
    }
}
```

之后，我们可以在过滤器中使用我们的属性：

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

use App\Validation\Attribute\Base64DecodedQuery;
use Spiral\Filters\Model\Filter;

class MyFilter extends Filter
{
    #[Base64DecodedQuery]
    public ?string $hash = null;
}
```

### 点符号

数据**来源**可以使用点符号来指定，指向某些嵌套结构。

通过属性：

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Post(key: 'names.first')]
    public string $firstName;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'firstName' => ['string', 'required']
        ]);
    }
}
```

我们可以接受并验证以下数据结构：

```json
{
  "names": {
    "first": "Antony"
  }
}
```

> **注意**
> 错误消息将被正确地挂载到原始位置。你也可以将复合过滤器用于更复杂的使用案例。

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use App\Endpoint\Web\Filter\MyFilter;

class HomeController
{
    public function index(MyFilter $filter): void
    {
        dump($filter->number); // always int
    }
}
```

## 数据净化

`Spiral\Filters\Attribute\Setter` 属性允许你在将传入值设置为类属性之前，将一个过滤器函数应用于该值。如果你想在将该值存储在类中之前对该值执行某种转换或操作，这可能很有用。

要使用此属性，只需在类定义中与其他过滤器属性一起指定它。

例如：

```php app/src/Endpoint/Web/Filter/UserProfileFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Setter;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;

class UserProfileFilter extends Filter
{
    #[Post]
    #[Setter(filter: 'trim')]
    public string $login;

    #[Post]
    #[Setter(filter: 'intval')]
    public int $age;

    #[Post]
    #[Setter(filter: [self::class, 'sanitizeContent'])]
    public string $bio = '';
    
    protected static function sanitizeContent(string $content): string
    {
        return \strip_tags($content);
    }
}
```

> **注意**
> 默认的 PHP 函数（如 `intval` 和 `strval`）可以与此属性一起使用。这使得将常见的数据操作函数应用于传入值变得很容易。

你可以为单个类属性指定多个 `Setter` 属性，允许你将一系列过滤器函数应用于传入值，然后再设置它。

以下是有关如何使用多个 setter 属性的示例：

```php
#[Post]
#[Setter(filter: 'strval')]
#[Setter('ltrim', '-')]
#[Setter('rtrim', ' ')]
#[Setter('htmlspecialchars')]
public string $name;
```

> **注意**
> 应用过滤器函数的顺序很重要。在上面的示例中，首先应用 `strval`，然后应用 `ltrim`、`rtrim`，最后应用 `htmlspecialchars`。

## 验证

验证规则可以使用与 [验证器](../validation/spiral.md) 组件中相同的方法定义。

> **注意**
> 如果应验证过滤器对象，则 FilterDefinition 类应实现 `Spiral\Filters\Model\ShouldBeValidated`。

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

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
            'name' => ['string', 'required']
        ]);
    }
}
```

### 处理验证错误

当某个过滤器规则出现错误时，将抛出 `Spiral\Filters\Exception\ValidationException` 异常。

Spiral 将通过 `Spiral\Filter\ValidationHandlerMiddleware` 中间件自动捕获此异常，并通过 `Spiral\Filters\ErrorsRendererInterface` 返回带有错误消息的响应。

你只需要注册中间件：

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Filter\ValidationHandlerMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            // ...
            ValidationHandlerMiddleware::class,
        ];
    }
}
```

默认情况下，中间件使用 `Spiral\Filter\JsonErrorsRenderer` 呈现过滤器错误。 你可以通过将你自己的 `Spiral\Filters\ErrorsRendererInterface` 实现绑定到容器来更改呈现器。

```php app/src/Filter/CustomJsonErrorsRenderer.php
namespace App\Filter;

use Psr\Http\Message\ResponseInterface;
use Spiral\Filters\ErrorsRendererInterface;
use Spiral\Http\ResponseWrapper;

final class CustomJsonErrorsRenderer implements ErrorsRendererInterface
{
    public function __construct(
        private readonly ResponseWrapper $wrapper
    ) {
    }

    public function render(array $errors, mixed $context = null): ResponseInterface
    {
        return $this->wrapper->json([
            'errors' => $errors,
            'context' => (string) $context
        ])
            ->withStatus(422, 'The given data was invalid.');
    }
}
```

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Filters\ErrorsRendererInterface;
use App\Filter\CustomJsonErrorsRenderer;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // Custom renderer to the container binding
    protected const SINGLETONS = [
        ErrorsRendererInterface::class => CustomJsonErrorsRenderer::class,
    ];

    // ...
}
```

### 自定义错误

你可以像在验证器组件中一样，为任何规则指定自定义错误消息。

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

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
            'name' => [
                ['required', 'error' => 'Name must not be empty']
            ]
        ]);
    }
}
```

如果你计划稍后本地化错误消息，请将文本包装在 `[[]]` 中，以自动索引和替换翻译：

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

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
            'name' => [
                ['required', 'error' => '[[Name must not be empty]]']
            ]
        ]);
    }
}
```

## 用法

配置 Filter 后，你可以访问其字段（已过滤的数据）。

### 获取字段

要获取过滤后的数据，请使用过滤器属性或方法 `getData`（如果它扩展了 `Spiral\Filters\Model\Filter`）：

```php app/src/Endpoint/Web/Filter/MyFilter.php
namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;
    
    #[Post]
    public string $email;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'name' => ['required']
        ]);
    }
}
```

以下字段可用：

```php
public function index(MyFilter $filter): void
{
    dump($filter->getData()); // {name: ..., email: ...}

    // or
    dump($filter->email);
    dump($filter->name);
}
```
