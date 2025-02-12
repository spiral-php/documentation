# HTTP — 请求与响应

您的控制器或端点需要一种访问活动的 PSR-7 请求和生成响应的方法。在本节中，我们将介绍在 MVC 设置中使用请求/响应的方法。

> **注意**
> 中间件和原生 PSR-15 处理器可以直接接收 PSR-7 对象。

## 请求范围

获取用户请求的最快方法是进行 `Psr\Http\Message\ServerRequestInterface` 方法注入。

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ServerRequestInterface;
use Spiral\Core\Container\SingletonInterface;

class HomeController implements SingletonInterface
{
    public function index(ServerRequestInterface $request): void
    {
        dump($request->getHeaders());
    }
}
```

> **警告**
> **不允许**在单例中使用 `Psr\Http\Message\ServerRequestInterface` 作为构造函数注入。

一旦获得请求，您就可以使用它来调用 [PSR-7 标准](https://www.php-fig.org/psr/psr-7/) 中可用的所有读取方法。

## InputManager

或者，您可以使用上下文管理器 `Spiral\Http\Request\InputManager`，该管理器可以存储在单例服务/控制器中，并且始终指向当前的用户请求。此对象提供了几个用户友好的方法来读取传入数据。

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Core\Container\SingletonInterface;
use Spiral\Http\Request\InputManager;

class HomeController implements SingletonInterface
{
    private InputManager $input;

    public function __construct(InputManager $input)
    {
        $this->input = $input;
    }

    public function index(): void
    {
        dump($this->input->query->all());
    }
}
```

您还可以通过 `Spiral\Prototype\Traits\PrototypeTrait` 访问 `Spiral\Http\Request\InputManager`。

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): void
    {
        // $this->request is alias to $this->input
        dump($this->request->data->all());
    }
}
```

> **警告**
> 建议避免直接访问 `Psr\Http\Message\ServerRequestInterface` 和 `Spiral\Http\Request\InputManager`，除非必要，请改用**请求过滤器**。

您可以使用 `Spiral\Http\Request\InputManager` 通过其名称（嵌套结构允许使用点符号）访问完整的输入数据数组或任何特定字段。每个输入结构都使用 `Spiral\Http\Request\InputBag` 类表示，该类具有一组常用方法。让我们以访问查询为例：

```php
/** @var \Spiral\Http\Request\InputManager $input */

// Get instance of QueryBag associated with query data
dump($input->query);

// Get all query params as array
dump($input->query->all());

// Count of query params
dump($input->query->count());

// Check if parameter "name" is presented in query
dump($input->query->has('name'));

// Get value for parameter "name"
dump($input->query->get('name'));

// Both get and has methods support dot notation for nested structures
dump($input->query->has('name.subName'));
dump($input->query->get('name.subName'));

// Fetch only given query params (no dot notation allowed), only existed records will be returned
dump($input->query->fetch(["name", "nameB"]);

// Fetch only given query params (no dot notation allowed), non existed records will be filled with `null`
dump($input->query->fetch(['name', 'nameB'], true, null);

// In addition query get method has short alias in input manager
dump($input->query('name'));
```

> **警告**
> 在 Input bag 具有 `null` 值的键时，`new \Spiral\Http\Request\InputBag(['name' => null]);`，方法 `$input->query->has('name')` 将返回 `true`，但 `isset($input->query['name'])` 将返回 `false`。

### 输入标头

我们可以使用 `Spiral\Http\Request\InputManager` 中的 '**headers**' 输入包和 `header` 方法来访问输入标头。 `Spiral\Http\Request\HeadersBag` 有一些我们必须提及的补充：

*   `Spiral\Http\Request\HeadersBag` 将自动规范化请求的标头名称
*   默认情况下，"**get**" 方法将使用 ',' 串联标头值

```php
/** @var \Spiral\Http\Request\InputManager $sinput */

// Get all headers as array
dump($input->headers->all());

// Will be normalized into "Accept"
dump($input->headers->get('accept'));

// Return Accept header as array of values
dump($input->headers->get('accept', false));

dump($input->header('accept'));
```

### Cookie

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->cookies->all());

dump($input->cookie('name'));
```

### 服务器变量

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->server->all());

dump($input->server('name'));
```

> **注意**
> `Spiral\Http\Request\ServerBag` 将自动规范化所有请求的服务器值。这样就可以在不使用所有大写字母作为名称的情况下获取值：

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->server('SERVER_PORT'));

dump($input->server('server-port'));
```

### Post/Data 参数

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->data->all());

dump($input->data('name'));

// An alias
dump($input->post('name'));
```

### Post/Data 具有回退到 Query 参数的功能

如果要从 POST 数据中读取值，然后从 Query 中读取值，只需使用 `input` 方法。

```php
dump($input->input('name'));
```

### PSR7 请求属性

```php
dump($input->attributes->all());

dump($input->attribute('name'));
```

#### 上传文件

要获取上传文件的列表或单个文件，请使用 `files` 包和 `file` 方法。每个上传的文件实例都使用 `Psr\Http\Message\UploadedFileInterface` 表示，它是 PSR7 的一部分。

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($this->input->files->all());

dump($this->input->file('upload'));
```

> **注意**
> 根据 PSR，所有文件都组织成一个逻辑层次结构，这与 php 处理上传文件的默认方式不同。您可以使用点符号来访问嵌套的文件实例。

### 简化方法

除了数据方法和 InputBags 之外，`Spiral\Http\Request\InputManager` 还提供了一组方法来读取活动请求的各种属性。

```php
/** @var \Spiral\Http\Request\InputManager $input */

//Request Uri path, will always include leading /
dump($input->path());

//Active request Uri instance
dump($input->uri());

//GET, POST, PUT...
dump($input->method());

//Check if connection made over https
dump($input->isSecure());

//Check request headers to verify that request made over ajax
dump($input->isAjax());

//Check is request expects application/json as response (Accept: application/json)
dump($input->isJsonExpected());

//Receive client ip address (this method uses _SERVER value and may not be correct in some cases).
dump($input->remoteAddress());
```

要访问 `Spiral\Http\Request\InputBag` 而不使用 `__get`：

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->bag('data')->all());
```

### 添加新的输入包

使用 `Spiral\Bootloader\Http\HttpBootloader`，您可以添加自己的输入包。

例如，我们需要接收作为 `Symfony\Bridge\PsrHttpMessage\Factory\UploadedFile` 对象的上传文件。让我们创建一个输入包类。

```php
namespace App\Http\Request;

use Spiral\Http\Request\InputBag;
use Symfony\Bridge\PsrHttpMessage\Factory\UploadedFile;

final class FilesBag extends InputBag
{
    public function __construct(array $data, string $prefix = '')
    {
        foreach ($data as $name => $file) {
            $data[$name] = new UploadedFile($file, fn(): string => $this->getTemporaryPath());
        }

        parent::__construct($data, $prefix);
    }

    protected function getTemporaryPath(): string
    {
        return \tempnam(\sys_get_temp_dir(), \uniqid('symfony', true));
    }
}
```

之后，通过 `HttpBootloader` 中的 `addInputBag` 方法添加创建的 `FilesBag`。

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Http\Request\FilesBag;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\HttpBootloader;

class AppBootloader extends Bootloader
{
    public function init(HttpBootloader $http): void
    {
        $http->addInputBag('symfonyFiles', [
            'class'  => FilesBag::class,
            'source' => 'getUploadedFiles',
            'alias' => 'symfony-file'
        ]);
    }
}
```

## InputInterface

`Spiral\Http\Request\InputManager` 的方法没有 `get` 前缀。其原因是位于外部包 `spiral/filters` 中，该包需要通过 `Spiral\Filters\InputInterface` 提供数据源：

```php
namespace Spiral\Filters;

// ...

interface InputInterface
{
    public function withPrefix(string $prefix, bool $add = true): InputInterface;

    public function getValue(string $source, string $name = null);
}
```

您可以通过 `Spiral\Filters\InputInterface` 的短符号来调用 `Spiral\Http\Request\InputManager` 方法。这两种方法将产生相同的数据集。

```php app/src/Endpoint/Web/HomeController.php
use Spiral\Filters\InputInterface;
use Spiral\Http\Request\InputManager;

public function index(InputInterface $inputSource, InputManager $inputManager): void
{
    dump($inputManager->query('name'));
    dump($inputSource->getValue('query', 'name'));

    dump($inputManager->path());
    dump($inputSource->getValue('path'));
}
```

这种方法用于将传入数据映射到请求过滤器。

> **警告**
> 您必须激活 `Spiral\Bootloader\Security\FiltersBootloader` 才能访问 `Spiral\Filters\InputInterface`。

## 生成响应

您可以从控制器返回实例 `Psr\Http\Message\ResponseInterface`，它将直接发送给用户。

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Nyholm\Psr7\Response;
use Psr\Http\Message\ResponseInterface;

class HomeController 
{
    public function index(): ResponseInterface
    {
        $response = new Response(200);
        $response->getBody()->write("hello world");

        return $response;
    }
}
```

默认启用的 PSR-15 处理器为您提供了根据返回的字符串或输出缓冲区的内容自动生成响应的能力：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

class HomeController
{
    public function index(): string
    {
        return "hello world";
    }
}
```

与以下内容相同：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

class HomeController
{
    public function index(): void
    {
        echo "hello world";
    }
}
```

> **注意**
> 我们建议仅在开发过程中使用输出缓冲区来显示调试信息。坚持严格的返回类型。

## JSON 响应

默认的 PSR-15 还支持将转换为 JSON 的数组和 `JsonSerializable` 响应：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

class HomeController
{
    public function index(): array
    {
        return [
            'status' => 200,
            'data' => ['some' => 'json']
        ];
    }
}
```

## 响应工厂

从手动创建响应中抽象出来的正确方法是使用 `Psr\Http\Message\ResponseFactoryInterface`：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;

class HomeController
{
    public function index(ResponseFactoryInterface $responseFactory): ResponseInterface
    {
        $response = $responseFactory->createResponse(200);
        $response->getBody()->write("hello world");

        return $response;
    }
}
```

## ResponseWrapper

要生成更复杂的响应，请使用 `ResponseFactoryInterface` 包装器 `Spiral\Http\ResponseWrapper`，它添加了许多用于更简单的响应生成的方法：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Http\ResponseWrapper;

class HomeController
{
    public function index(ResponseWrapper $response): ResponseInterface
    {
        return $response->attachment(
            __FILE__,
            'controller.php'
        )->withAddedHeader('Key', 'value');
    }
}
```

您还可以通过 `PrototypeTrait` 和属性 `response` 访问包装器：

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    Use PrototypeTrait;

    public function index(): ResponseInterface
    {
        // temporary redirect
        return $this->response->redirect('https://google.com', 307);
    }
}
```

要创建 HTML 响应：

```php app/Interface/Controllers/HomeController.php
public function index(): ResponseInterface
{
    return $this->response->html('hello world');
}
```

要创建 `application/json` 响应：

```php app/Interface/Controllers/HomeController.php
public function index(): ResponseInterface
{
    return $this->response->json(
        ['something' => 123],
        200
    );
}
```

要发送附件：

```php app/Interface/Controllers/HomeController.php
public function index(): ResponseInterface
{
    return $this->response->attachment(__FILE__, 'name.php');
}
```

> **注意**
> 您还可以使用 `Psr\Http\Message\StreamInterface` 作为第一个参数，并指定您的 MIME 类型作为第三个选项。
