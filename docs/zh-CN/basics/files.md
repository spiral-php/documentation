# 基础知识 — 文件和目录

框架提供了一个简单的组件 `spiral/files` 来处理文件系统。

## 目录注册器

大多数 Spiral 组件依赖于目录注册器而不是硬编码的路径。 注册器使用 `Spiral\Boot\DirectoriesInterface` 表示。

> **了解更多**
> 在[入门 — 目录结构](../start/structure.md)章节中阅读更多关于应用程序目录结构的内容。

你可以在应用程序的入口点 (`app.php`) 配置应用程序特定的目录：

```php app.php
$app = \App\Application\Kernel::create(
    directories: [
        'root' => __DIR__,
        'uploadDir' => __DIR__ . '/upload'
    ]
)->run();
```

或者使用 Bootloader:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\DirectoriesInterface;

final class AppBootloader extends Bootloader
{
    public function boot(DirectoriesInterface $dirs): void
    {
        $dirs->set(
            'uploadDir',
            $dirs->get('root') . '/upload'
        );
    }
}
```

访问目录路径：

```php
use Spiral\Boot\DirectoriesInterface;

final class UploadService {
    public function __construct(
        private readonly DirectoriesInterface $dirs
    ) {}
    
    public function store(UploadedFile $file) {
        $filePath = $this->dirs->get('uploadDir') . $file->getFilename();
        // ...
    }
}
```

> **注意**
> 你也可以在你的配置文件中使用简短的函数 `directory`。 请注意，此函数在框架外部无法工作，因为它依赖于全局容器范围。

如果你希望目录名称始终受特定方法的约束，你可以创建一个帮助类，例如 `AppDirectories`。

<details>
  <summary>点击展开示例。</summary>

```php app/src/Application/AppDirectories.php
namespace App\Application;

use Spiral\Boot\DirectoriesInterface;

final class AppDirectories
{
    public function __construct(
        private readonly DirectoriesInterface $directories
    ) {
    }

    /**
     * 应用程序根目录。
     * @return non-empty-string
     */
    public function getRoot(?string $path = null): string
    {
        return $this->buildPath('root', $path);
    }

    /**
     * 应用程序目录。
     * @return non-empty-string
     */
    public function getApp(?string $path = null): string
    {
        return $this->buildPath('app', $path);
    }

    /**
     * 公共目录。
     * @return non-empty-string
     */
    public function getPublic(?string $path = null): string
    {
        return $this->buildPath('public', $path);
    }

    /**
     * 运行时目录。
     * @return non-empty-string
     */
    public function getRuntime(?string $path = null): string
    {
        return $this->buildPath('runtime', $path);
    }

    /**
     * 运行时缓存目录。
     * @return non-empty-string
     */
    public function getCache(?string $path = null): string
    {
        return $this->buildPath('cache', $path);
    }

    /**
     * 供应商库目录。
     * @return non-empty-string
     */
    public function getVendor(?string $path = null): string
    {
        return $this->buildPath('vendor', $path);
    }

    /**
     * 配置目录。
     * @return non-empty-string
     */
    public function getConfig(?string $path = null): string
    {
        return $this->buildPath('config', $path);
    }

    /**
     * 资源目录。
     * @return non-empty-string
     */
    public function getResources(?string $path = null): string
    {
        return $this->buildPath('resources', $path);
    }

    private function buildPath(string $key, ?string $path = null): string
    {
        return \rtrim($this->directories->get($key), '/') . ($path ? '/' . \ltrim($path, '/') : '');
    }
}
```
</details>

## 文件

使用 `Spiral\Files\FilesInterface` 组件来处理文件系统：

```php
use Spiral\Files\FilesInterface;

final class FileService
{   
    public function __construct(
        private readonly FilesInterface $files,
        private readonly AppDirectories $dirs
    ) {}

    public function getRootFiles()
    {
        // 从根目录递归地获取所有文件
        dump(
            $files->getFiles($dirs->getRoot())
        );
    }
}
```

你也可以使用原型属性 `files` 访问此实例：

```php
use Spiral\Prototype\Traits\PrototypeTrait;

final class FileService
{
    use PrototypeTrait;
   
    public function __construct(
        private readonly AppDirectories $dirs
    ) {}
    
    public function store()
    {
        dump($this->files->exists(__FILE__)); // true
    }
}
```

> **了解更多**
> 在 [基础知识 — 原型](../basics/prototype.md)章节中阅读更多关于原型属性的内容。

### 创建目录

要确保给定目录存在，请使用方法 `ensureDirectory`，第二个参数接受访问权限：

```php
public function store()
{
    $this->files->ensureDirectory(
        $dirs->get('customDir'),
        FilesInterface::READONLY // or FilesInterface::RUNTIME for editable dirs and files
    );
}
```

检查目录是否存在：

```php
dump($files->isDirectory(__DIR__));
```

### 删除目录

删除目录及其内容：

```php
$files->deleteDirectory('custom');
```

仅删除目录内容：

```php
$files->deleteDirectory('custom', true);
```

### 文件统计信息

检查文件是否存在：

```php
dump($files->exists(__FILE__)); // bool
```

获取文件创建/更新时间：

```php
dump($files->time(__FILE__)); // unix timestamp
```

获取文件 MD5 值：

```php
dump($files->md5('filename'));
```

获取文件名扩展名：

```php
dump($files->extension(__FILE__)); // without leading "."
```

检查路径是否为文件：

```php
dump($files->isFile(__DIR__));
```

获取文件大小：

```php
dump($files->size(__DIR__));
```

### 权限

获取文件/目录权限：

```php
dump($files->getPermissions(__FILE__)); // int
```

设置文件权限：

```php
$files->setPermissions(__FILE__, 0777)
```

使用常量控制文件模式：

| 常量                 | 值   |
|----------------------|------|
| FilesInterface::READONLY | 644  |
| FilesInterface::RUNTIME  | 666  |

### 移动和复制

将文件从一个路径复制到另一个路径：

```php
$files->copy('old-path', 'new-path');
```

移动文件：

```php
$files->move('old-path', 'new-path');
```

### 临时文件

*创建*一个临时文件名：

```php
dump($files->tempFilename());
```

创建带有特定扩展名的临时文件名：

```php
dump($files->tempFilename('php'));
```

创建位于特定位置的临时文件名：

```php
dump($files->tempFilename('php', __DIR__));
```

## 读写操作

该组件提供了多种方法，以原子方式操作文件内容（无需获取文件资源）：

### 写入/创建文件

将内容写入文件（独占锁）：

```php
$files->write('filename', 'data');
```

写入/创建文件并确保正确的访问模式：

```php
$files->write('filename', 'data', 0777);
```

检查并自动创建文件目录：

```php
$files->write('filename', 'data', 0777, true);
```

> **注意**
> 如果文件不可写，请确保处理 `Spiral\Files\Exception\WriteErrorException` 异常。

### 追加内容

追加文件内容：

```php
$files->append('filename', 'data');
```

追加并确保文件模式：

```php
$files->append('filename', 'data', 0777);
```

追加/创建文件并确保目标目录存在：

```php
$files->append('filename', 'data', 0777, true);
```

### 触摸

触摸文件并在缺失时创建它：

```php
$files->touch('filename');
```

触摸文件并确保文件模式：

```php
$files->touch('filename', 0777);
```

### 读取文件

读取文件内容：

```php
dump($files->read('filename'));
```

> **注意**
> 当找不到文件时，请确保处理 `Spiral\Files\Exception\FileNotFoundException` 异常。
