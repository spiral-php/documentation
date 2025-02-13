# 测试 — HTTP 测试

Spiral 的测试包提供了一种非常方便的方式来测试你的应用程序控制器。你可以使用 `Spiral\Testing\Http\FakeHttp` 类来向你的控制器发送请求并检查响应。

例如，在下面的例子中，我们可以看到对 `HomeController` 的测试。我们创建了一个 `FakeHttp` 实例，并用它向 `/` 端点发送了一个 `GET` 请求。然后，我们检查了响应是否为 `OK`。

```php tests/Feature/HomeControllerTest.php
namespace Tests\Feature;

use Spiral\Testing\Http\FakeHttp;
use Tests\TestCase;

final class HomeControllerTest extends TestCase
{
    private FakeHttp $http;

    protected function setUp(): void
    {
        parent::setUp();
        $this->http = $this->fakeHttp();
    }

    public function testIndex(): void
    {
        $response = $this->http->get('/');
        $response->assertOk();
    }
}
```

**这是一种非常直接的方式来测试你的控制器。**

## 发送请求

当你使用 `FakeHttp` 类发送请求时，它会直接发送到你的控制器，就像一个真实的请求一样，通过任何中间件和拦截器。你可以在你的测试中使用任何标准的 HTTP 方法，例如 `GET`, `POST`, `DELETE`, `PUT`。

`FakeHttp` 类不会返回一个真实的响应 `Psr\Http\Message\ResponseInterface` 对象，而是返回一个 `Spiral\Testing\Http\TestResponse` 对象。它提供了各种有用的断言，让你检查诸如响应状态、标头和正文之类的内容。这使得你可以轻松地测试你的应用程序如何响应不同的请求，并确保其按预期工作。

### 请求方法

`FakeHttp` 类提供了多种方法，用于向你的应用程序发送不同类型的请求。

#### GET

`get()` 方法用于发送 `GET` 请求。它接受多个参数，如 `uri`, `query`, `headers` 和 `cookies`。

```php
$http = $this->fakeHttp();
$response = $http->get(
    uri: '/users',
    query: ['sort' => 'desc'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
);
```

使用 `getJson` 方法发送一个带有以下标头的 `GET` 请求：

- `Accept: application/json`
- `Content-type: application/json`

```php
$http = $this->fakeHttp();
$response = $http->getJson(
    uri: '/users',
    query: ['sort' => 'desc'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
);
```

#### POST

```php
$http = $this->fakeHttp();
$response = $http->post(
    uri: '/user/1',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

> **注意**
> `data` 参数接受一个数组或 `Psr\Http\Message\StreamInterface` 对象。

使用 `postJson` 方法发送一个带有以下标头的 `POST` 请求：

- `Accept: application/json`
- `Content-type: application/json`

```php
$http = $this->fakeHttp();
$response = $http->postJson(
    uri: '/user/1',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

#### PUT

```php
$http = $this->fakeHttp();
$response = $http->put(
    uri: '/users',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

> **注意**
> `data` 参数接受一个数组或 `Psr\Http\Message\StreamInterface` 对象。

使用 `putJson` 方法发送一个带有以下标头的 `PUT` 请求：

- `Accept: application/json`
- `Content-type: application/json`

```php
$http = $this->fakeHttp();
$response = $http->putJson(
    uri: '/users',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

#### DELETE

```php
$http = $this->fakeHttp();
$response = $http->delete(
    uri: '/user/1',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

> **注意**
> `data` 参数接受一个数组或 `Psr\Http\Message\StreamInterface` 对象。

使用 `deleteJson` 方法发送一个带有以下标头的 `DELETE` 请求：

- `Accept: application/json`
- `Content-type: application/json`

```php
$http = $this->fakeHttp();
$response = $http->deleteJson(
    uri: '/user/1',
    data: ['foo' => 'bar'],
    headers: ['Content-type' => 'application/json'],
    cookies: ['token' => 'xxx-xxxx'],
    files: ['avatar' => ...],
);
```

### 请求标头

是的，你可以使用 `withHeaders()` 和 `withHeader()` 方法来为请求添加/设置默认自定义标头。

#### 设置一组标头

```php
$http = $this->fakeHttp();
$http->withHeaders(['Content-type' => 'application/json']);

$http->get('/users');
```

#### 设置单个标头

```php
$http = $this->fakeHttp();
$http->withHeader('Content-type', 'application/json');
$http->get('/users');
```

#### 设置授权标头

```php
$http = $this->fakeHttp();
$http->withAuthorizationToken(
    token: 'xxx-xxxx',
    type: 'Bearer' // 默认值是 'Bearer'
);
```

#### 刷新标头

此方法将刷新所有默认标头。

```php
$http = $this->fakeHttp();
$http->flushHeaders();
```

### 请求 Cookie

是的，你可以使用 `withCookies()` 和 `withCookie()` 方法为请求添加/设置默认自定义 Cookie。

#### 设置一组 Cookie

```php
$http = $this->fakeHttp();
$http->withCookies(['theme' => 'dark']);

$http->get('/users');
```

#### 设置单个 Cookie

```php
$http = $this->fakeHttp();
$http->withHeader('theme', 'dark');
$http->get('/users');
```

#### 刷新 Cookie

此方法将刷新所有默认 Cookie。

```php
$http = $this->fakeHttp();
$http->flushCookies();
```

### 会话/身份验证

Spiral 提供了一些很棒的工具来帮助你在测试 HTTP 请求时与会话进行交互。其中一个工具是 `withSession()` 方法。它允许你在向你的应用程序发送请求之前，将会话数据设置为特定的数组。当你需要为你的测试加载包含某些特定数据的会话时，这会非常方便。

例如，你可以像这样设置会话数据：

```php
$http = $this->fakeHttp();
$http->withSession(
    data: ['fav_color' => 'blue'],
    lifetime: 3600, // 默认值是 3600
    id: null // 默认值是 null
)->get('/users');
```

会话通常用于维护当前已通过身份验证的用户的状态。这就是为什么 Spiral 提供了一个 `withActor` 方法，它使得在测试期间将给定的用户身份验证为当前用户变得容易。

此方法允许你通过将用户对象的一个实例传递给该方法，快速而轻松地为测试设置经过身份验证的用户。

```php
$http = $this->fakeHttp();

$user = new User();
$http->withActor($user)->get('/profile');
```

当你调用 `withActor()` 方法时，一个 `Spiral\Testing\Auth\FakeActorProvider` 对象被绑定到容器，实现了 `Spiral\Auth\ActorProviderInterface`，并且这个对象将在每次调用 `getActor()` 方法时返回你传递的用户。这允许应用程序在测试期间访问已通过身份验证的用户，就像在正常操作期间一样。

## 测试响应

在向你的应用程序发送请求之后，你可以使用便捷的方法来测试响应。

### 可用的断言

#### assertHasHeader

断言响应具有给定的标头。

```php
// 断言响应具有名为 Content-Type 的标头。
$response->assertHasHeader(name: 'Content-type');

// 断言响应具有名为 Content-Type 且值是给定值的标头。
$response->assertHasHeader(name: 'Content-type', value: 'application/json');
```

#### assertHeaderMissing

断言响应没有给定的标头。

```php
$response->assertHeaderMissing(name: 'Content-type');
```

#### assertStatus

断言响应具有给定的状态码。

```php
$response->assertStatus(status: 200);
```

#### assertOk

断言响应具有 200 状态码。

```php
$response->assertOk();
```

#### assertCreated

断言响应具有 201 状态码。

```php
$response->assertCreated();
```

#### assertAccepted

断言响应具有 202 状态码。

```php
$response->assertAccepted();
```

#### assertNoContent

断言响应具有 204 状态码并且主体为空。

```php
$response->assertNoContent(
    status: 204 // 默认值是 204
);
```

#### assertNotFound

断言响应具有 404 状态码。

```php
$response->assertNotFound();
```

#### assertForbidden

断言响应具有 403 状态码。

```php
$response->assertForbidden();
```

#### assertUnauthorized

断言响应具有 401 状态码。

```php
$response->assertUnauthorized();
```

#### assertUnprocessable

断言响应具有 422 状态码。

```php
$response->assertUnauthorized();
```

#### assertBodySame

断言响应主体等于给定的值。

```php
$response->assertBodySame(needle: "<html>...</html>");
```

#### assertBodyNotSame

断言响应主体不等于给定的值。

```php
$response->assertBodyNotSame(needle: "<html>...</html>");
```

#### assertBodyContains

断言响应主体包含给定的字符串。

```php
$response->assertBodyContains(needle: "<div>...</div>");
```

#### assertCookieExists

断言响应具有给定的 Cookie。

```php
$response->assertCookieExists(key: 'theme');
```

#### assertCookieSame

断言响应具有给定的 Cookie，且该 Cookie 具有给定的值。

```php
$response->assertCookieSame(key: 'theme', value: 'dark');
```

#### assertCookieMissed

断言响应没有给定的 Cookie。

```php
$response->assertCookieMissed(key: 'theme');
```

## 测试文件上传

是的，`FakeHttp` 类提供了 `getFileFactory` 方法，该方法可用于为测试目的生成虚拟文件或图像。此方法返回 `Spiral\Testing\Http\FileFactory` 的一个实例，该实例具有创建不同类型文件的几种方法。

此功能使你可以轻松测试你的应用程序如何处理文件上传，并确保其功能正常。

```php tests/Feature/UserControllerTest.php
namespace Tests\Feature;

use Spiral\Testing\Http\FakeHttp;
use Tests\TestCase;

final class UserControllerTest extends TestCase
{
    private FakeHttp $http;

    protected function setUp(): void
    {
        parent::setUp();
        $this->http = $this->fakeHttp();
    }

    public function testUploadAvatar(): void
    {
        // Create a fake image 640x480
        $image = $http->getFileFactory()->createImage('avatar.jpg', 640, 480);

        $response = $http->post(uri: '/user/1', files: ['avatar' => $image]);
        $response->assertOk();
    }
}
```

### 创建一个假的图像

```php
$image = $http->getFileFactory()->createImage(
    filename: 'avatar.jpg',
    width: 640,
    height: 480
);
```

### 创建一个假的文件

```php
$file = $http->getFileFactory()->createFile(
    filename: 'foo.txt'
);
```

#### 带有文件大小

你可以设置文件大小（以千字节为单位）：

```php
// 创建一个大小为 100kb 的文件
$file = $http->getFileFactory()->createFile(
    filename: 'foo.txt',
    kilobytes: 100
);
```

#### 带有 MIME 类型

你可以设置 MIME 类型：

```php
// 创建一个大小为 100kb 的文件
$file = $http->getFileFactory()->createFile(
    filename: 'foo.txt',
    mimeType: 'text/plain'
);
```

#### 带有内容

你可以设置一个带有特定内容的文件：

```php
$file = $http->getFileFactory()->createFileWithContent(
    filename: 'foo.txt',
    content: 'Hello world',
    mimeType: 'text/plain'
);
```
