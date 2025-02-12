# 高级 — 邮件发送器

Spiral 提供了一个由 symfony/mailer 组件驱动的简单电子邮件 API。

> **注意**
> 默认情况下，该组件在 [application bundle](https://github.com/spiral/app) 中可用。

邮件通过 "transport"（传输器）发送。开箱即用，您可以通过在 `.env` 文件中配置 DSN 来通过 SMTP 传送邮件（user、pass 和 port 参数是可选的）。

## 安装

要启用该组件，您需要将 `Spiral\SendIt\Bootloader\MailerBootloader` 类添加到 bootloaders 列表中。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\SendIt\Bootloader\MailerBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloaders 的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\SendIt\Bootloader\MailerBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloaders 的信息。
:::

::::

此 bootloader 配置基本绑定、默认设置和用于发送邮件的队列。它还将自动注册 `Spiral\SendIt\Bootloader\BuilderBootloader`，该 bootloader 注册了 `spiral/views` 组件，并提供了使用 **Stempler** 模板引擎创建邮件模板的能力。

## 配置

:::: tabs

::: tab 环境变量
您可以通过 `.env` 变量配置该组件：

```dotenv
MAILER_DSN=smtp://username:password@example.com:25
MAILER_FROM=John Smith

MAILER_QUEUE_CONNECTION=roadrunner
MAILER_QUEUE=emails
```

:::

::: tab 配置文件
您也可以通过 `app/config/mailer.php` 文件配置该组件。

这是一个配置文件示例：

```php
return [
    /**
     * -------------------------------------------------------------------------
     *  Transport setup
     * -------------------------------------------------------------------------
     *
     * The MAILER_DSN isn't a real address: it's a convenient format that offloads most of the configuration work to mailer.
     * Supported formats are compatible with the symfony/mailer component.
     */
    'dsn' => env('MAILER_DSN'),

    /**
     * -------------------------------------------------------------------------
     *  From
     * -------------------------------------------------------------------------
     *
     * Instead of calling ->from() on each Email you create, you can configure this value globally.
     */
    'from' => env('MAILER_FROM'),

    /**
     * -------------------------------------------------------------------------
     *  Configuration queue connection
     * -------------------------------------------------------------------------
     *
     * This section allows you to configure the `queue` and the `driver` for sending emails.
     * The `queueConnection` with the value `sync` allows you to send emails without using a queue.
     */
    'queue' => env('MAILER_QUEUE'), // won't be used for sync queue
    'queueConnection' => env('MAILER_QUEUE_CONNECTION', 'sync'),
];

```

:::

::::

### DSN

`MAILER_DSN` 参数用于配置传送邮件的传输器。该组件使用 `symfony/mailer` 组件发送邮件，并且支持多种内置传输器，包括 `SMTP`、`sendmail` 和 PHP `mail()` 函数。

> **注意**
> DSN (Data Source Name) 是一个字符串，用于指定要使用的传输器，并且可以包含诸如 user、password 和 port 等选项。 这是一个例子：`smtp://username:password@example.com:25`

| DSN 协议 | 示例                                | 描述                                                                                                                                                                                                                                                        |
|--------------|----------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| smtp         | `smtp://user:pass@smtp.example.com:25` | 邮件发送器使用 SMTP 服务器发送邮件                                                                                                                                                                                                                |
| sendmail     | `sendmail://default`                   | 邮件发送器使用本地 sendmail 二进制文件发送邮件                                                                                                                                                                                                      |
| native       | `native://default`                     | 邮件发送器使用 sendmail 二进制文件和在 `php.ini` 的 `sendmail_path` 设置中配置的选项。 在 Windows 主机上，如果未配置 `sendmail_path`，邮件发送器将回退到 `php.ini` 的 `smtp` 和 `smtp_port` 设置。 |

> **注意**
> 您可以在官方 [Symfony 文档](https://symfony.com/doc/current/mailer.html#using-built-in-transports) 中找到更多关于 `symfony/mailer` 组件支持的不同类型的传输器及其配置方法的信息。

### 自定义邮件传输器

除了 `symfony/mailer` 组件支持的内置传输器之外，Spiral 还允许您使用自定义传输器。

要使用自定义传输器，您需要创建一个实现 `Symfony\Component\Mailer\Transport\TransportInterface` 接口的类，该接口是 `symfony/mailer` 组件的一部分。此类应处理使用所需传输器发送邮件的逻辑。

假设您有多个 DSN 传输器，并且您想使用 Round Robin 算法来选择其中一个来发送邮件。

这是一个如何使用自定义传输器的示例：

```php
use Spiral\Boot\Bootloader\Bootloader;
use Symfony\Component\Mailer\Transport\TransportInterface;
use Symfony\Component\Mailer\Transport\RoundRobinTransport;
use Symfony\Component\Mailer\Transport;

class AppBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        \Spiral\SendIt\Bootloader\MailerBootloader::class,
    ];
    
    protected const SINGLETONS = [
        TransportInterface::class => [self::class, 'initTransport'],
    ];
    
    public function initTransport(): TransportInterface
    {
        $transports = [];
        
        $dsns = [
            'smtp://...',
            'smtp://...',
            // ...
        ];

        foreach($dsns as $dsn) {
            $transports[] = Transport::fromDsn($dsn);
        }

        return new RoundRobinTransport($transports);
    }
}
```

> **注意**
> 重要的是要注意，创建和配置自定义传输器可能很复杂，并且可能需要深入了解 Symfony Mailer 组件及其 Transport 接口

### 队列

发送邮件可能是一项耗时的任务，如果您的应用程序发送大量邮件，则可能会降低应用程序的整体性能。 缓解此问题的一种方法是使用队列作业发送邮件。

<details>
    <summary>点击查看使用队列作业发送邮件的优势</summary>

- **提高性能：** 通过将发送邮件的任务转移到队列作业，您可以防止主应用程序在等待发送邮件时被阻塞。 这可以提高您的应用程序的整体性能。
- **可扩展性：** 如果您的应用程序需要发送大量邮件，使用队列作业允许您扩展工作进程的数量以处理负载，而不会影响主应用程序的性能。
- **重试机制：** 如果发送邮件时发生错误，作业可以设置为在稍后或经过一定次数的尝试后重试。 这对于处理临时故障（例如电子邮件服务器宕机）很有用。
- **优先级：** 根据邮件优先级，您可以对队列进行排序并首先处理最重要的邮件。
- **日志记录：** 队列作业提供了一个日志记录机制，用于跟踪邮件的状态（已发送、失败、重试）。

</details>

Spiral 提供了一种简便的方法来设置用于发送邮件的队列连接和管道。

在 `.env` 文件中，您可以通过设置 `MAILER_QUEUE_CONNECTION` 和 `MAILER_QUEUE` 变量来配置用于发送邮件的队列连接和管道。

您也可以在 `app/config/mailer.php` 配置文件中配置邮件队列。

> **另请参阅**
> 在 [Queue — Installation and Configuration](../queue/configuration.md) 部分阅读更多关于队列连接配置的信息。

## 用法

该组件提供了使用 `Stempler` 视图来编写内容丰富的邮件模板的功能：

```html

<extends:sendit:builder subject="I'm afraid I can't do that"/>
<use:bundle path="sendit:bundle"/>

<email:attach path="{{ $attachment }}" name="attachment.txt"/>

<block:html>
    <p>I'm sorry, {{ $name }}!</p>
    <p>
        <email:image path="path/to/image.png"/>
    </p>
</block:html>
```

使用方法：

```php
use Spiral\Mailer\MailerInterface;
use Spiral\Mailer\Message;

public function send(MailerInterface $mailer): void
{
    $mailer->send(new Message(
        'template.dark.php', 
        'email@domain.com',
        [
            'name' => 'Dave',
            'attachment' => __FILE__,
        ]
    ));
}
```

### 延迟发送消息

该组件允许延迟发送消息。延迟时间使用 `setDelay` 方法设置：

```php
use Spiral\Mailer\Message;

$message = new Message('test', 'email@domain.com');
$message->setDelay(new \DateTimeImmutable('+ 60 minute'));
// or
$message->setDelay(new \DateInterval('PT60S'));
// or
$message->setDelay(100);
```

## 自定义邮件传输器

在某些情况下，您可能需要使用 `symfony/mailer` 组件未提供的自定义邮件，在这种情况下，您可以使用 `Spiral\SendIt\TransportRegistryInterface` 接口注册自定义传输器。

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\SendIt\TransportRegistryInterface;
use Symfony\Component\Mailer\Transport\c;

class AppBootloader extends Bootloader 
{
    public function boot(TransportRegistryInterface $registry): void
    {
        $registry->registerTransport(new SendmailTransportFactory(...));
    }
}
```

## 事件

| 事件                              | 描述                                          |
|------------------------------------|------------------------------------------------------|
| Spiral\SendIt\Event\MessageSent    | 该事件将在发送消息`之后`触发。 |
| Spiral\SendIt\Event\MessageNotSent | 如果无法发送消息，则会触发该事件。 |

> **注意**
> 要了解有关调度事件的更多信息，请参阅我们文档中的 [Events](../advanced/events.md) 部分。
