# 组件 — 存储与云分发

Spiral 通过其 `spiral/storage` 和 `spiral/distribution` 组件提供了一个全面的文件存储和分发解决方案。`spiral/storage` 组件利用 Flysystem PHP 软件包的功能，提供强大的存储抽象，并提供用于处理本地文件系统和 Amazon S3 的便捷驱动程序。`spiral/distribution` 组件与 `spiral/storage` 组件集成，负责为通过存储组件存储的资源生成公共 HTTP 链接。

## 存储

`spiral/storage` 组件得益于 Frank de Jonge 出色的 [Flysystem](https://github.com/thephpleague/flysystem) PHP 软件包，提供了强大的存储抽象。存储组件的集成提供了用于处理本地文件系统和 Amazon S3 的简单驱动程序。更棒的是，在本地开发机器和生产服务器之间切换这些存储选项非常简单，因为每个系统的 API 保持不变。

> **注意**
> 与经典文件系统不同，存储组件提供了一个 API，该 API 提供了写入文件、检查其存在、读取和获取此文件的公共地址的操作。 不提供（经典文件系统拥有的）所有用于处理目录或文件列表的操作。

### 安装

要安装该组件：

```terminal
composer require spiral/storage
```

确保将 `Spiral\Storage\Bootloader\StorageBootloader` 添加到应用程序的引导程序列表中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Storage\Bootloader\StorageBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Storage\Bootloader\StorageBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::::

### 配置

默认情况下，存储配置文件位于 `app/config/storage.php`。 此文件允许配置服务器和特定存储，也称为“桶（buckets）”。 您可以在 `servers` 部分配置您的应用程序将使用的服务器，而在 `buckets` 部分可以指定服务器将使用的存储。

例如，一个包含一个服务器和两个存储的最简单配置可能如下所示：

```php app/config/storage.php
return [

    /**
     * -------------------------------------------------------------------------
     *  默认存储桶名称
     * -------------------------------------------------------------------------
     *
     * 在此处，您可以指定希望默认用于所有存储操作的存储桶。
     *
     */

    'default' => env('STORAGE_SERVER', 'uploads'),

    /**
     * -------------------------------------------------------------------------
     *  存储服务器
     * -------------------------------------------------------------------------
     *
     * 您的应用程序配置了每个服务器。
     * 当然，下面显示了自定义 Spiral 支持的每个可用服务器的示例，以简化开发。
     *
     */

    'servers' => [
        'static' => [
            'adapter' => 'local',
            'directory' => __DIR__ . '/../../runtime/static',
        ],

        's3' => [
            'adapter' => 's3', // or "s3-async"
            'region' => env('S3_REGION'),
            'bucket' => env('S3_BUCKET'),
            'key' => env('S3_KEY'),
            'secret' => env('S3_SECRET'),
        ],
    ],

    /**
     * -------------------------------------------------------------------------
     *  存储桶
     * -------------------------------------------------------------------------
     *
     *  这里是使用上面服务器设置的特定桶（或存储）的列表。
     *  此列表中每个“server”部分必须引用上面列表中的有效服务器名称。
     *
     *  在这种情况下，设置列表也是一个使用示例。您可以
     *  根据需要自由更改桶的数量和设置类型。
     *
     */

    'buckets' => [
        'uploads' => [
            'server' => 'static',
            'prefix' => 'upload',
        ],

        'images' => [
            'server' => 'static',
            'prefix' => 'img',
        ],

        'videos' => [
            'server' => 's3',
        ],
    ],
];
```

> **注意**
> 请注意，此配置仅在与 Spiral Framework 一起使用时可用。

#### 手动配置（框架之外）

仅当组件单独安装在框架之外时，才需要使用这种方式。

首先，您需要创建一个存储实例，其中将存储所有桶。 之后，您可以通过所需的名称从该实例添加和获取任意桶。

```php
$storage = new \Spiral\Storage\Storage();

$storage->add('example', \Spiral\Storage\Bucket::fromAdapter(
    new \League\Flysystem\Local\LocalFilesystemAdapter(__DIR__ . '/path/to/directory')
));

$file = $storage->bucket('example')
    ->write('file.txt', 'content');
```

您可能已经注意到，您可以使用现有的 [flysystem adapters](https://flysystem.thephpleague.com/v2/docs/adapter/local/) 来创建桶。 只需安装您想要的那个，然后使用 `Bucket::fromAdapter()` 方法将其添加到存储中。

#### 本地服务器

顾名思义，本地服务器位于本地文件系统中（与应用程序的可执行代码位于同一位置）。

您已经看到了本地服务器设置的示例，但是，为了简化它们，已经特别删除了某些可选部分。 现在，让我们看一下这种类型的服务器的完整配置，保留所有可能的配置部分。

<details>
  <summary>单击以显示配置示例。</summary>

```php app/config/storage.php
return [
    'servers' => [
        'profiles' => [
            //
            // 服务器类型名称。对于本地服务器，此值必须是
            // 字符串值 "local"。
            //
            'adapter' => 'local',

            //
            // 您的文件将被存储到的本地目录的必需路径。
            //
            'directory'  => '/app/storage/user-profiles',

            //
            // 可见性映射。在这里，您可以设置文件的默认可见性，
            // 以及对应于某种可见性的文件和目录的权限。
            //
            // 可见性值只能是 "private" 或 "public"。
            //
            'visibility' => [
                'public'  => ['file' => 0644, 'dir' => 0755],
                'private' => ['file' => 0600, 'dir' => 0700],

                'default' => 'public',
            ],
        ],
    ],

    'buckets' => [
        'bucket' => [
            //
            // 与现有本地服务器的关系。请注意，所有进一步的
            // 桶选项仅适用于此（即配置文件）服务器
            // 类型。
            //
            'server' => 'profiles',

            //
            // 对于使用本地服务器的桶，您可以添加目录
            // 前缀。在这种情况下，文件的实际物理路径将
            // 如下所示："/app/storage/user-profiles/avatars"，其中
            //  "/example/directory" 字符串是物理目录
            //  在服务器中指定，而 "avatars" - 文件的前缀
            //  在桶中指定的目录。
            //
            'prefix' => 'avatars',
        ]
    ]
];
```

</details>

RoadRunner 插件提供了 [Fileserver 插件](https://roadrunner.dev/docs/http-static#file-server-plugin)，它通过使用 URL 前缀促进了从各种本地存储提供文件服务。 此功能可以对特定文件和目录的可见性和访问进行细粒度的控制。

例如，您可以将 URL 前缀 http://127.0.0.1:10101/avatars 映射到 `/app/storage/user-profiles/avatars` 目录，以提供对静态资产的公共访问。

以下是 Fileserver 插件的配置示例：

```yaml .rr.yaml
fileserver:
  address: 127.0.0.1:10101
  calculate_etag: true
  weak: false
  stream_request_body: true
  serve:
    - prefix: "/avatars"
      root: "/app/storage/user-profiles/avatars"
```

#### S3 服务器

这种类型的服务器旨在通过 S3 协议与外部分布式文件系统进行交互。 S3 是与 [Amazon](https://s3.console.aws.amazon.com/s3/home) 服务器通信的主要协议。 除了 Amazon S3 本身之外，还有免费的替代方案，您可以在自己的服务器上安装和使用，例如 [Minio Server](https://docs.min.io/)。

要设置这种类型的服务器，您将需要一个现有的存储桶和个人身份验证数据。 此外，为了完整起见，以下示例将包含可选且具有默认值的参数。

请注意，为了与这种类型的服务器交互，您必须安装两个可用软件包中的一个（或两者）。 您必须使用 Composer 安装 `league/flysystem-aws-s3-v3` 或 `league/flysystem-async-aws-s3` 软件包。

```terminal
composer require league/flysystem-aws-s3-v3 ^2.0
// OR
composer require league/flysystem-async-aws-s3 ^2.0
```

在配置过程中，您应该在服务器的“adapter”部分指定将使用哪个软件包。
`"s3"` 值对应于 `league/flysystem-aws-s3-v3` 软件包，而 `"s3-async"` 值对应于 `league/flysystem-async-aws-s3` 软件包。

<details>
  <summary>单击以显示配置示例。</summary>

```php app/config/storage.php
return [
    'servers' => [
        'local' => [
            //
            // 服务器类型名称。对于 S3 服务器，此值必须是字符串
            // 值 "s3" 或 "s3-async"。
            //
            'adapter' => 's3',

            //
            // S3 区域的必需字符串键，如 "eu-north-1"。
            //
            // 可以在此处找到区域在“Amazon S3”页面上：
            //  - https://s3.console.aws.amazon.com/s3/home
            //
            'region' => env('S3_REGION'),

            //
            // S3 API 版本的可选键。
            //
            'version' => env('S3_VERSION', 'latest'),

            //
            // S3 存储桶的必需键。
            //
            // 可以在此处找到存储桶名称在“Amazon S3”页面上：
            //  - https://s3.console.aws.amazon.com/s3/home
            //
            'bucket' => env('S3_BUCKET'),

            //
            // S3 凭证密钥的必需键，如 "AAAABBBBCCCCDDDDEEEE"。
            //
            // 可以在此处找到凭证密钥在“安全凭证”页面上：
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            'key' => env('S3_KEY'),

            //
            // S3 凭证私钥的必需键。
            // 这必须是私钥字符串值或私钥文件的路径。
            //
            // 标识符也可以在此处的“安全凭证”页面上找到：
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            'secret' => env('S3_SECRET'),

            //
            // S3 凭证令牌的可选键。
            //
            'token' => env('S3_TOKEN', null),

            //
            // S3 凭证过期时间的可选键。
            //
            'expires' => env('S3_EXPIRES', null),

            //
            // S3 文件的可选键。默认情况下，可见性为“public”。
            //
            'visibility' => env('S3_VISIBILITY', 'public'),

            //
            // 对于使用 S3 服务器的桶，您可以添加一个目录
            // 前缀。
            //
            'prefix' => '',

            //
            // S3 API 终端节点 URI 的可选键。 使用 Amazon 以外的服务器时，需要此值。
            //
            'endpoint' => env('S3_ENDPOINT', null),

            //
            // S3 可选的附加选项。
            // 例如，需要选项 “use_path_style_endpoint” 才能使用
            //  Minio S3 服务器。
            //
            // 注意：此“options”部分自 framework >= 2.8.5 起可用
            // 另请参见 https://github.com/spiral/framework/issues/416
            //
            'options' => [
                'use_path_style_endpoint' => true,
            ]
        ],
    ],

    'buckets' => [
        'bucket' => [
            //
            // 与现有 S3 服务器的关系。请注意，所有进一步的桶
            // 选项仅适用于此（即 s3 或 s3-async）服务器
            // 类型。
            //
            'server' => 's3',

            //
            // 特定桶类型的可见性值。虽然可以
            // 指定服务器范围的可见性，您还可以覆盖此值
            //  用于特定桶类型。
            //
            'visibility' => env('S3_VISIBILITY', 'public'),

            //
            // 如果您想使用另一个使用主存储桶
            // 服务器设置，您可以通过指定
            // 适当的配置键。
            //
            'bucket' => env('S3_BUCKET', null),

            //
            // 如果新存储桶位于不同的区域，您也可以
            // 覆盖此值。
            //
            'region' => env('S3_REGION', null),

            //
            // 类似的事情可以在目录前缀中使用
            // 某个桶必须指向其他根目录。
            //
            'prefix' => 'custom-directory',
        ]
    ]
];
```

</details>

#### 自定义服务器

在某些情况下，标准适配器可能不够用，在这种情况下，您可能需要指定自己的适配器。 您也可以使用
您的配置文件来配置自定义适配器。

在这种情况下，适配器部分必须引用 `League\Flysystem\FilesystemAdapter` 实现，而 “options” 部分将包含一个传递给该适配器构造函数的参数数组。

<details>
  <summary>单击以显示配置示例。</summary>

```php app/config/storage.php
return [
    'servers' => [
        'custom' => [
            //
            // 包含适配器类名的服务器类型名称。
            //
            'adapter' => \Custom\FlysystemAdapter::Class,

            //
            // 适配器的构造函数参数。
            //
            'options' => [
                // ...
            ]
        ],
    ],

    'buckets' => [
        'bucket' => [
            //
            // 与自定义服务器的关系。
            //
            'server' => 'custom'
        ]
    ]
];
```

</details>

### 用法

最后，在熟悉了组件支持的服务器类型后，我们可以继续
使用它们。

存储体系结构假定对相同操作有 3 个不同的访问级别：Storage、Bucket 和 File。在
这三个级别中的每一个级别上，您都可以对文件进行操作，但区别在于您必须将多少数据传输到特定的方法。 在最高的“Storage”级别，您必须传递有关桶和文件的信息。 在“Bucket”级别，您只需传递有关文件的信息。 最后，在“File”级别，您将不必传递任何附加信息。

在实践中，它将是这样的。 让我们尝试在应用程序的某个控制器中以 3 种不同的方式创建一个文件 `example.txt`。

```php
use Spiral\Storage\StorageInterface;

class UploadController
{
    public function createFile(StorageInterface $storage): array
    {
        $result = [];
        
        // 1. Storage 级别
        $result[] = $storage->create('bucket://example.txt');

        // 2. Bucket 级别
        $result[] = $storage->bucket('bucket')
            ->create('example.txt');

        // 3. File 级别
        $result[] = $storage->bucket('bucket')
            ->file('example.txt')
            ->create();

        return $result;
    }
}
```

这些方法之间的区别如下：

- 在使用存储时，您应该以类似 URI 的格式 `[BUCKET_NAME]://[FILE_NAME]` 传递文件名。
  在使用存储的所有方法中，这是第一个字符串参数。

- 在使用存储桶的所有方法中，这是任意格式的文件名。另请注意，前导斜杠不会以任何方式影响文件位置，并且名称 `file.txt` 和 `/file.txt` 将完全
  相同。

- 在使用特定文件的所有方法中，不需要其他参数。

每种方法都有其优点和缺点。只需使用您最喜欢的即可。

由于在上面的示例中我们正在使用一个控制器，所以同时我们将使用一个替代依赖项，
这适用于简单的文件保存情况，不需要许多存储桶的实现。

在这种情况下，将选择系统中默认使用的桶。您可以在
配置的 `'default'` 部分中定义桶。

```php
use Spiral\Storage\BucketInterface;
use Psr\Http\Message\ServerRequestInterface;

class UploadingController
{
    public function upload(ServerRequestInterface $request, BucketInterface $bucket): string
    {
        /** @var \Psr\Http\Message\UploadedFileInterface $file */
        foreach ($request->getUploadedFiles() as $i => $file) {
            $bucket->write("file-{$i}.txt", $file->getStream());
        }

        return \count($request->getUploadedFiles()) . ' files uploaded';
    }
}
```

从每个级别，您可以引用子级。

**从存储：**

- `$storage->bucket('[bucket-name]'): BucketInterface`
- `$storage->file('[bucket-name]://[file-name]'): FileInterface`

**从存储桶：**

- `$bucket->file('[file-name]'): FileInterface`

在了解了在存储的不同级别使用相同方法的选项后，我们
可以继续描述可用的可能性。让我们开始！

#### 创建和写入

如果要创建文件，可以使用两种可用方法之一：`create()` 或 `write()`。第一个创建一个
空文件，如果它未创建，而第二个允许您另外在那里写入任意 `string` 或 `resource` 流内容。

例如，创建文件的代码可能如下所示。

```php
// 从桶创建文件
$file = $bucket->create('file.txt');

// 从桶创建具有字符串内容的文件
$file = $bucket->write('file.txt', 'message');

// 从桶创建具有资源流内容的文件
$file = $bucket->write('file.txt', fopen(__DIR__ . '/local/file.txt', 'rb+'));
```

#### 复制和移动

要复制文件，请使用 `copy()` 方法，该方法包含一个必需的参数，即新文件的名称，以及一个可选参数 - 应将文件复制到的桶。如果未指定第二个参数，则编译桶将与原始桶相同。

`move()` 方法与使用 `copy()` 方法完全相似，但它不是复制而是移动文件。

```php
$backup = $firstBucket->copy('from.txt', 'backup.txt');

$moved  = $firstBucket->move('backup.txt', 'to.txt', $secondBucket);
```

> 请注意，当文件从具有私有权限的桶移动（或复制）到具有公共权限的分布时，文件的可见性也将更改为与该桶相符的可见性。

#### 删除

要删除文件，只需使用 `delete()` 方法即可。此方法接受一个可选的布尔参数，该参数表示删除
文件所在位置的空目录。

```php
$bucket->delete('file.txt');
```

#### 读取内容

有两种方法可以读取现有文件的内容：使用 `getContents()` 和 `getStream()` 方法。第一个方法返回文件的字符串内容，第二个方法返回用于处理流数据的资源流。

```php
$string = $bucket->getContents('text.txt');

$resource = $bucket->getStream('music.mp3');
```

#### 存在性检查

要检查文件的存在性，请使用 `exists()` 方法，如果文件存在于存储桶中，则该方法返回布尔值 `true`，如果不存在，则返回 `false`。

```php
$isExists = $bucket->exists('file.txt');
```

#### 文件大小

要获取有关文件大小的信息，请使用 `getSize()` 方法，该方法返回现有文件的大小（以字节为单位）。

```php
$bytes = $bucket->getSize('file.txt');
```

#### 上次修改时间

要检索有关文件上次修改日期的信息，请使用 `getLastModified()` 方法，该方法以 UNIX 时间戳格式返回时间。

```php
$timestamp = $bucket->getLastModified('file.txt');
```

#### MIME 类型

要获取文件 MIME 类型，请使用 `getMimeType()` 方法，该方法返回文件的字符串 MIME 类型表示。

```php
$mime = $bucket->getMimeType('file.txt');
```

#### 可见性

除了文件是否存在等特征外，还有文件可见性。 您可以使用 `getVisibility()` 方法获取
有关文件可见性的信息，并使用 `setVisibility()` 方法更新可见性。

这些方法对已在 `Spiral\Storage\Visibility` 类似于枚举的接口中定义的常量进行操作。
因此，文件可见性控制的示例代码将如下所示。

```php
$visibility = $bucket->getVisibility('file.txt');

// 如果文件是“private”，则发布它
if ($visibility === Visibility::VISIBILITY_PRIVATE) {
    $bucket->setVisibility('file.txt', Visibility::VISIBILITY_PUBLIC);
}
```

> **注意**
> 如果使用位于 Windows OS 上的桶，此功能可能无法运行。

#### URI 发布

存储组件最初仅提供用于处理任意文件系统的操作。 但是，
除了存储之外，某些系统还允许 HTTP 组织对这些文件的访问点，以便通过浏览器接收其内容。

存储组件允许您使用
[分发组件](../component/distribution.md)添加这些文件的公共地址的任意 URI 解析器。

要配置解析器，您应该熟悉此组件的配置。 之后，对于一个
特定的桶，只需添加一个包含指向特定分发解析器的链接的部分。

每个桶都可以指定分发。

```php
return [
    // ...
    'buckets' => [
        'uploads' => [
            'server' => '...',

            //
            // + 添加与现有分发的关联。
            //
            'distribution' => 'NAME_OF_DISTRIBUTION'
        ],
    ],
];
```

要使用此 URI 解析器，只需调用 `toUri()` 方法。 如果您需要任何其他分发，可以在 `toUriFrom(...)` 方法中显式指定它。

> **注意**
> 与其他方法不同，此方法只能在特定文件上调用。

```php
use Spiral\Storage\BucketInterface;
use Spiral\Distribution\UriResolverInterface;

class UriController
{
    // 使用默认的 URI 解析器
    public function getUri(BucketInterface $bucket): string
    {
        return (string)$bucket
            ->file('picture.jpg')
            ->toUri();
    }

    // 使用另一个 URI 解析器
    public function getAnotherUri(BucketInterface $bucket, UriResolverInterface $resolver): string
    {
        return (string)$bucket
            ->file('picture.jpg')
            ->toUriFrom($resolver);
    }
}
```

某些生成器可能接受其他选项。 这种参数可以传递给 `toUri([...$arguments])`
或 `toUriFrom($resolver, [...$arguments])` 方法。 例如，如果您创建一个指向 CloudFront 的链接，您可以
另外指定此链接的过期时间。

> **另请参阅**
> 您可以在分发部分的相应页面上阅读有关可能附加参数的更多信息。

```php
$uri = $file->toUri(new \DateInterval('PT30S'));
// 并且
$uri = $file->toUriFrom($resolver, new \DateInterval('PT30S'));
```

## 分发

`spiral/distribution` 组件负责提供任意资源的公共 HTTP 链接。 在大多数
情况下，这将与网站本身的地址相同，但是，在某些情况下，资源可能位于外部服务器上，例如 [Amazon CloudFront](https://aws.amazon.com/cloudfront/) 或其他一些 CDN。 在这些情况下，
生成资源的公共链接需要使用提供商的特定 API，或为
使用的 CDN 编写自己的代码。 该组件使这种交互更容易，并提供了一些内置驱动程序，用于生成指向外部供应商的 URI。

### 安装

使用 Composer 安装该组件：

```terminal
composer require spiral/distribution
```

要启用该组件，您只需要将 `Spiral\Distribution\Bootloader\DistributionBootloader` 类添加到
引导程序列表中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Distribution\Bootloader\DistributionBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Distribution\Bootloader\DistributionBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::::

### 配置

默认情况下，分发配置文件位于 `app/config/distribution.php`。

```php app/config/distribution.php
<?php

return [

    /**
     * -------------------------------------------------------------------------
     *  默认分发解析器名称
     * -------------------------------------------------------------------------
     *
     * 在此处，您可以指定希望在
     * 默认情况下用于所有 URI 生成的解析器。 当然，您可以使用
     * 多个解析器，同时使用分发库。
     *
     */

    'default' => env('DISTRIBUTION_RESOLVER', 'local'),

    /**
     * -------------------------------------------------------------------------
     *  分发解析器
     * -------------------------------------------------------------------------
     *
     *  以下是为您的应用程序配置的每个解析器。
     *  当然，下面显示了自定义 Spiral 支持的每个可用分发的示例，
     *  以简化开发。
     *
     */

    'resolvers' => [
        'local' => [
            'type' => 'static',
            'uri'  => env('APP_URL', 'http://localhost')
        ],

        'cloudfront' => [
            'type' => 'cloudfront',
            'key' => env('AWS_CF_KEY'),
            'domain' => env('AWS_CF_KEY'),
            'private' => env('AWS_CF_PRIVATE_KEY'),
        ],

        's3' => [
            'type' => 's3',
            'region' => env('S3_REGION'),
            'bucket' => env('S3_BUCKET'),
            'key' => env('S3_KEY'),
            'secret' => env('S3_SECRET'),
        ],
    ],

];
```

> **警告**
> 提供的配置示例特定于 Spiral Framework，并且不适用于该框架之外。配置选项和结构将根据所使用的特定框架或系统而有所不同。

#### 手动配置（框架之外）

仅当组件单独安装在框架之外时，才需要使用这种方式。

首先，您需要创建一个管理器实例，其中将存储所有 URI 解析器。 之后，您可以通过所需的名称从该实例添加和
获取任意解析器。

```php
$manager = new \Spiral\Distribution\Manager();

$manager->add('resolver-name', new CustomResolver());

$manager->resolver('resolver-name'); // object(CustomResolver)
```

之后，您可以在其中添加您自己的管理器，或组件提供的管理器，例如“static”。

```php
use Nyholm\Psr7\Uri;
use Spiral\Distribution\Manager;
use Spiral\Distribution\Resolver\StaticResolver;

$manager = new Manager();
$manager->add('local', new StaticResolver(new Uri('https://static.example.com')));
```

### 用法

配置好组件后，您就可以开始使用它了。

如果您正在使用 Spiral 应用程序，则管理器已配置好。您可以从
[容器](../container/overview.md) 获取它，或者通过 [依赖注入](../container/overview.md#dependency-injection) 获取它。

```php
use Spiral\Distribution\DistributionInterface;

class FilesController
{
    public function showImage(DistributionInterface $dist): string
    {
        $resolver = $dist->resolver('local');

        return (string)$resolver->resolve('example/image.jpg');
    }
}
```

如果需要“default”配置部分中定义的默认解析器，则不需要
获取整个管理器实例。您可以立即从容器中获取您想要的解析器。

```php
use Spiral\Distribution\UriResolverInterface;

class FilesController
{
    public function showImage(UriResolverInterface $resolver): string
    {
        return (string)$resolver->resolve('example/image.jpg');
    }
}
```

您可能已经注意到，在上面的示例中获取解析器后，`resolve()` 方法被用于指向文件的相对路径。它将一个字符串值作为参数，并返回
[PSR-7 `Psr\Http\Message\UriInterface`](https://www.php-fig.org/psr/psr-7/) 的实现。

```php
$uri = $resolver->resolve('path/to/file.txt');
//
// 预期：
//  object(Psr\Http\Message\UriInterface)
//
```

> **注意**
> 某些解析器在获取链接时支持其他选项，
> 例如：`$cloudfront->resolve('path/to/file.txt', expiration: new \DateInterval('PT60S'));`

#### 静态 URI 解析器

这种类型的解析器通过将传递的文件链接添加到解析器配置中指定的 URI 的末尾来生成资源的地址。

要配置这种类型的解析器，您只需要指定两个必需的字段。

```php app/config/distribution.php
return [
    // ...
    'resolvers' => [
        // ...
        'local' => [
            //
            // 解析器类型的必需键。
            // 对于静态解析器，它必须包含“static”字符串值。
            //
            'type' => 'static',

            //
            // 静态服务器 URL 的必需键。
            //
            'uri'  => env('APP_URL', 'http://localhost')
        ],
    ]
];
```

与用于在 [url generator](../http/routing.md#url-generation) 路由组件中为页面生成地址的类似方法不同，链接可以是任意的，并且可以在旨在提供静态内容的单独服务器上进行配置。

这样，如果将任意文件字符串传递给 `resolve()` 方法，您将收到指向该文件的物理 http 链接。 如果基本 uri 定义为“`http://localhost`”，结果将如下所示：

```php
/** @var \Spiral\Distribution\Resolver\StaticResolver $resolver */
$resolver = $manager->resolver('local');

echo $resolver->resolve('path/to/file.txt');
//
// 预期：
//  string(33) "http://localhost/path/to/file.txt"
//
```

#### CloudFront URI 解析器

CloudFront 是一项流行的静态分发服务，与 Amazon 服务结合使用。 要使用它，您必须
使用 Composer 安装 `aws/aws-sdk-php` 软件包。

```terminal
composer require aws/aws-sdk-php ^3.0
```

在 AWS 服务中注册并创建静态服务器之后，[您将收到](https://console.aws.amazon.com/cloudfront/home) 设置的参数。 此外，
您将需要“私钥文件”和“访问密钥 ID”，您可以在“[安全凭证](https://console.aws.amazon.com/iam/home#/security_credentials)”页面上的“CloudFront 密钥对”选项卡上找到它们。

要配置此解析器，只需在配置部分中指定连接参数：

```php app/config/distribution.php
return [
    // ...
    'resolvers' => [
        // ...
        'cloudfront' => [
            //
            // 解析器类型的必需键。
            // 对于 CloudFront，它必须包含“cloudfront”字符串值。
            //
            'type' => 'cloudfront',

            //
            // CloudFront 访问密钥 ID 的必需键。
            // 这必须包含字符串值，如“AAAABBBBCCCCDDDDEEEE”。
            //
            // 可以在您的个人“安全凭证”页面上找到标识符，网址为：
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            'key' => env('AWS_CF_KEY'),
            
            //
            // CloudFront 私钥的必需键。
            // 这必须是私钥字符串值或私钥文件的路径。
            //
            // 标识符也可以在此处的“安全凭证”页面上找到：
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            // 请注意，您只能在生成期间下载私钥文件！
            //
            'private' => env('AWS_CF_PRIVATE_KEY'),

            //
            // CloudFront 域名所需的密钥。
            // 这必须包含字符串值，如“example.cloudfront.net”。
            //
            // 可以在此处找到域在“CloudFront 分发”页面上：
            //  - https://console.aws.amazon.com/cloudfront/home
            //
            'domain' => env('AWS_CF_DOMAIN'),
            
            //
            // CloudFront 文件前缀的可选键。
            // 这必须包含类似“path/to/directory”的字符串。在这种情况下，
            // 每次生成 URL 时都会添加此前缀。
            //
            'prefix' => env('AWS_CF_PREFIX'),
        ],
    ]
];
```

如果您决定自己创建解析器，可以使用传递给
用于处理 CloudFront 服务的解析器的构造函数的相同设置。

```php
//
// 在构造函数中使用 PHP 8 命名参数是为了清晰起见
//
$cloudfront = new \Spiral\Distribution\Resolver\CloudFrontResolver(
    keyPairId: 'AAAABBBBCCCCDDDDEEEE',
    privateKey: \file_get_contents(__DIR__ . '/path/to/key.pem'),
    domain: 'example.cloudfront.net',
    prefix: 'path/to/files'
);

$url = $cloudfront->resolve(...);
```

CloudFront 解析器接收作为 `resolve()` 方法的第一个参数的应该生成公共
地址的文件链接，并且作为第二个可选参数，该链接的生存期（过期时间）。

过期时间可以以多种格式指定。 它可以是：

- PHP 内置 [`\DateInterval` 对象](https://www.