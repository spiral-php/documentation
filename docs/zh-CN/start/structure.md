# 入门 — 目录结构

Spiral 框架不对您的应用程序施加任何目录结构限制，因此您可以灵活地组织文件和目录。然而，它提供了一个推荐的结构，可以作为起点。此结构也可以轻松修改。

## 目录

默认的目录结构可以通过 Kernel 类的 `mapDirectories` 方法进行控制。默认情况下，所有应用程序目录将基于 `root` 使用以下模式进行计算：

| 目录      | 值                  |
|-----------|------------------------|
| root      | **由用户设置**        |
| app       | **root**/app           |
| config    | **app**/config         |
| resources | **app**/resources      |
| runtime   | **root**/runtime       |
| cache     | **root**/runtime/cache |
| public    | **root**/public        |
| vendor    | **root**/vendor        |

某些组件将声明它们自己的目录，例如：

| 组件             | 目录      | 值              |
|-------------------|------------|--------------------|
| spiral/views      | views      | **app**/views      |
| spiral/translator | locale     | **app**/locale     |
| spiral/migrations | migrations | **app**/migrations |

## 初始化目录

您可以在 `app.php` 文件中设置 `root` 目录或任何其他目录的值。

```php
$app = \App\Application\Kernel::create(
    directories: ['root' => __DIR__]
)->run();
```

例如，如果您想将 `runtime` 目录的位置更改为系统的临时目录，您可以这样做：

```php
$app = \App\Application\Kernel::create(
    directories: [
        'root' => __DIR__, 
        'runtime' => \sys_get_temp_dir()
    ]
)->run();
```

您可以使用 `Spiral\Boot\DirectoriesInterface` 类访问应用程序中各种目录的路径。此接口提供了访问通过 `mapDirectories` 方法定义的各种目录的路径的方法。

以下是如何使用 `DirectoriesInterface` 类访问 `runtime` 目录路径的示例：

```php
use Spiral\Boot\DirectoriesInterface;

final class UploadService {
    public function __construct(
        private readonly DirectoriesInterface $dirs
    ) {}
    
    public function store(UploadedFile $file) {
        $filePath = $this->dirs->get('runtime') . 'uploads/' . $file->getFilename();
        // ...
    }
}
```

您也可以在全局 IoC 作用域内使用 `directory` 函数（配置文件、控制器、服务代码）。

```php app/config/cache.php
return [
    'storages' => [
        'file' => [
            'path' => directory('runtime') . 'cache',
        ],   
    ],
];
```

## 命名空间

默认情况下，所有骨架应用程序都使用 `App` 根命名空间，指向 `app/src` 目录。您可以在 `composer.json` 中将基础命名空间更改为任何想要的命名空间：

```json composer.json
{
  "autoload": {
    "psr-4": {
      "App\\": "app/src/"
    }
  }
}
```

## 应用程序结构

我们下面提供的结构是许多 PHP 应用程序中使用的通用结构，并且可以作为您项目的起点。通过遵循此结构，您可以以逻辑和可维护的方式组织您的代码，从而随着时间的推移更容易地构建和扩展您的应用程序。当然，您可能需要进行调整以适应项目的特定需求，但是此结构为大多数应用程序提供了坚实的基础。

```
- Endpoint
    - Web
        - ...
        - Filter
            - ...
        - Middleware
            - ...
        - Interceptor
            - ...
        - DataGrid
            - ...
        - routes.php
    - Console
        - Interceptor
            - ...
        - ...
    - RPC
        - Interceptor
            - ...
        - ...
    - Temporal
        - Workflow
            - ...
        - Activity
            - ...
    - Centrifugo
        - Interceptor
        - ...
- Application
    - Bootloader
        - ...
    - Exception
        - SomeException.php
        - Renderer
            - ViewRenderer.php
    - Kernel.php
- Domain
    - User
        - Entity
            - User.php
        - Service
            - StoreUserService.php
        - Repository
            - UserRepositoryInterface.php
        - Exception
            - UserNotFoundException.php
- Infrastructure
    - Persistence
        - CycleUserRepository.php
    - CycleORM
        - Typecaster
            - UuidTypecast.php
    - Interceptor
        - LogInterceptor.php
```

#### 这是对该结构中的目录和文件的简要说明：

- **Endpoint**: 此目录包含应用程序的入口点，包括 HTTP 端点（在 Web 子目录中）、命令行界面（在 Console 子目录中）和 gRPC 服务（在 RPC 子目录中）。

- **Application**: 此目录包含应用程序的核心，包括启动应用程序的 Kernel 类、将服务注册到容器的 Bootloader 类，以及包含异常处理逻辑的 Exception 目录。

- **Domain**: 此目录包含您的领域逻辑，按子域组织。例如，User 模型的 Entity、用于存储新用户的 Service、用于从数据库获取用户的 Repository，以及用于处理用户相关错误的 Exception。

- **Infrastructure**: 此目录包含应用程序的基础设施代码，包括用于数据库相关代码的 Persistence 目录、用于 ORM 相关代码的 CycleORM 目录以及用于全局拦截器的 Interceptor 目录。

<hr>

## 接下来是什么？

现在，通过阅读一些文章深入了解基础知识：

* [Kernel 和 Environment](../framework/kernel.md)
* [文件和目录](../basics/files.md)
