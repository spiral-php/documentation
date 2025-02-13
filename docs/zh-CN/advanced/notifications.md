# 高级 — 通知

Spiral 拥有一个非常酷的通知系统，允许你通过不同的渠道向用户发送各种消息。你可以通过 [spiral-packages/notifications](https://github.com/spiral-packages/notifications) 包发送电子邮件、短信、Slack 消息、推送通知等等。它由一个名为 Symfony Notifier 的组件提供支持，并且超级易于使用。

你只需要安装该包，在你的应用程序中注册它，然后就可以开始使用了！

## 安装

要安装该包，你可以使用 composer 命令：

```terminal
composer require spiral-packages/notifications
```

安装该包后，你需要将 Bootloader 注册到你的应用程序的 Bootloader 列表中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Notifications\Bootloader\NotificationsBootloader::class,
        // ...
    ];
}
```

在 [框架 — Bootloader](../framework/bootloaders.md) 部分阅读更多关于 Bootloader 的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Notifications\Bootloader\NotificationsBootloader::class,
    // ...
];
```

在 [框架 — Bootloader](../framework/bootloaders.md) 部分阅读更多关于 Bootloader 的信息。
:::

::::

完成这些步骤后，你将完全将通知包集成到你的应用程序中。

## 配置

为了充分利用该包的功能，你需要在配置文件中设置各种渠道、传输和策略。

该文件位于 `app/config/notifications.php`，允许你指定通知的发送方式和位置，以及可能需要的任何其他选项。

这是一个配置文件示例：

```php app/config/notifications.php
use Symfony\Component\Notifier\Channel\BrowserChannel;
use Symfony\Component\Notifier\Channel\ChatChannel;
use Symfony\Component\Notifier\Channel\EmailChannel;
use Symfony\Component\Notifier\Channel\PushChannel;
use Symfony\Component\Notifier\Channel\SmsChannel;

return [
    'channels' => [
        'nexmo_sms' => [
            'type' => 'sms',
            'transport' => 'nexmo',
        ],
        'default_email' => [
            'type' => 'email',
            'transport' => 'smtp',
        ],
        'roundrobin_email' => [
            'type' => 'email',
            'transport' => ['smtp', 'smtp_1'], // will be used roundrobin algorithm
        ],
        'chat/slack' => [
            'type' => 'chat',
            'transport' => 'slack',
        ],
    ],

    'transports' => [
        'nexmo' => 'nexmo://KEY:SECRET@default?from=FROM',
        'smtp' => 'smtp://user:pass@smtp.example.com:25',
        'smtp_1' => 'smtp://user:pass@smtp.example.com:25',
        'slack' => 'slack://TOKEN@default?channel=CHANNEL'
    ],

    'policies' => [
        'urgent' => ['sms', 'chat/slack', 'email'],
        'high' => ['chat/slack', 'push/firebase'],
    ],

    'queueConnection' => env('NOTIFICATIONS_QUEUE_CONNECTION', 'sync'),

    'typeAliases' => [
        'browser' => BrowserChannel::class,
        'chat' => ChatChannel::class,
        'email' => EmailChannel::class,
        'push' => PushChannel::class,
        'sms' => SmsChannel::class,
    ],
];
```

### 渠道类型

该包支持各种通知渠道，每个渠道都有其独特的性能和集成选项。

#### 这些渠道包括：

- **短信渠道 (SMS channel)**：允许你向电话号码发送短信。它可以与 Twilio 等提供商集成以发送消息。
- **聊天渠道 (Chat channel)**：将通知发送到 Slack 和 Telegram 等聊天服务。
- **电子邮件渠道 (Email channel)**：与 Symfony Mailer 集成以发送电子邮件通知。
- **浏览器渠道 (Browser channel)**：使用 flash 消息向用户的浏览器发送通知。
- **推送渠道 (Push Channel)**：允许你向移动设备和 Web 浏览器发送推送通知。

> **了解更多 (See more)**
> 在官方 [Symfony 文档](https://symfony.com/doc/current/notifier.html) 中阅读更多关于渠道类型的信息。

在配置文件中，可以在 `typeAliases` 部分注册所有可用于发送通知的渠道。此部分是一个数组，其中 **key** 表示渠道的类型，**value** 表示将处理该渠道的类。

`typeAliases` 数组中的键必须与配置文件 `channels` 部分中定义的 `type` 匹配，以便系统知道为每个渠道使用哪个类。

### 传输 (Transports)

`transports` 部分定义了可用于发送通知的各种服务，例如 nexmo、smtp、slack 等。每个传输都有一个唯一的键，该值是连接字符串，其中包含连接到服务所需的凭据。

例如：

```php
'transports' => [
    'nexmo' => 'nexmo://KEY:SECRET@default?from=FROM',
    'smtp' => 'smtp://user:pass@smtp.example.com:25',
    'smtp_1' => 'smtp://user:pass@smtp.example.com:25',
    'slack' => 'slack://TOKEN@default?channel=CHANNEL'
],
```

> **注意 (Note)**
> 你可以按照 [链接](https://symfony.com/doc/current/notifier.html#channels-chatters-texters-email-browser-and-push) 查看所有可用传输的完整列表。

### 渠道 (Channels)

在配置文件中，你可以根据你的应用程序的需要注册任意数量的渠道。每个渠道都在一个数组中定义，其中键是渠道的名称。

让我们看一个例子：

```php app/config/notifications.php
'email' => [
    'type' => 'email',
    'transport' => 'smtp',
],
```

- `email` 键是渠道的名称，你可以在你的通知类中使用它来路由通知。
- `type` 键是发送通知的渠道的类型。
- `transport` 键指定将用于发送通知的传输。如果将该值设置为一个数组，该包将使用循环算法来选择传输。这意味着该包将循环遍历传输数组，为一个通知使用一个传输，然后为下一个通知使用下一个传输，依此类推。

例如，如果你有以下配置：

```php app/config/notifications.php
'roundrobin_email' => [
  'type' => 'email',
  'transport' => ['smtp', 'smtp_1'],
],
```

### 策略 (Policies)

`policies` 部分定义了不同的通知策略，这些策略指定了应该为不同类型的通知使用哪些渠道。

```php app/config/notifications.php
'policies' => [
    'urgent' => ['sms', 'chat/slack', 'email'],
    'high' => ['chat/slack', 'push/firebase'],
],
```

例如，`urgent` 策略指定应使用 `SMS`、`chat/slack` 和 `email` 通知。

## 使用

要使用该包发送通知，你需要同时拥有一个接收者和一个通知。

### 接收者 (Recipient)

接收者应该实现 `Symfony\Component\Notifier\Recipient\RecipientInterface` 接口。

如果接收者应该接收短信通知，他们还应该实现 `Symfony\Component\Notifier\Recipient\SmsRecipientInterface` 接口，该接口定义了特定于短信通知的其他方法。类似地，对于电子邮件通知，接收者应该实现 `Symfony\Component\Notifier\Recipient\EmailRecipientInterface` 接口。

这确保了该包拥有通过正确渠道向正确接收者发送通知所需的所有必要信息。

这是一个用户类的示例，该类可以作为通知的接收者：

```php
use Symfony\Component\Notifier\Recipient\RecipientInterface;
use Symfony\Component\Notifier\Recipient\SmsRecipientInterface;

final class User implements RecipientInterface, SmsRecipientInterface
{
    // ...

    public function getPhone(): string
    {
        return '+8(000)000-00-00';
    }
}
```

### 通知 (Notification)

一个通知类应该扩展 `Symfony\Component\Notifier\Notification\Notification` 类，该类提供了通知类应该具有的基本方法。

> **了解更多 (See more)**
> 在 [官方文档](https://symfony.com/doc/current/notifier.html#creating-sending-notifications) 中阅读更多关于通知类的信息。

```php
use Symfony\Component\Notifier\Notification\Notification;
use Symfony\Component\Notifier\Notification\SmsNotificationInterface;
use Symfony\Component\Notifier\Message\SmsMessage;

class UserBannedNotification extends Notification implements SmsNotificationInterface
{
    public function getChannels(RecipientInterface $recipient): array
    {
        if ($recipient instanceof SmsRecipientInterface) {
            return ['nexmo_sms'];
        }
        
        return ['chat/slack'];
    }

    public function asSmsMessage(SmsRecipientInterface $recipient, string $transport = null): ?SmsMessage
    {
        return SmsMessage::fromNotification($this, $recipient);
    }
}
```

`getChannels()` 方法允许你指定通知应该发送到哪些渠道。在此示例中，通知将发送到 SMS 和聊天渠道。

你可以定义 `getImportance()` 而不是 `getChannels()`。该方法允许你指定通知的紧急程度，在这种情况下，它被设置为 `urgent`。此重要性级别可以在配置文件的 `policies` 部分中定义。

```php
use Symfony\Component\Notifier\Notification\Notification;
use Symfony\Component\Notifier\Notification\SmsNotificationInterface;
use Symfony\Component\Notifier\Message\SmsMessage;

class UserBannedNotification extends Notification implements SmsNotificationInterface
{
    public function getImportance(): string
    {
        return 'urgent';
    }

    public function asSmsMessage(SmsRecipientInterface $recipient, string $transport = null): ?SmsMessage
    {
        return SmsMessage::fromNotification($this, $recipient);
    }
}
```

### 发送通知 (Sending a notification)

创建了通知类和接收者类后，就可以发送通知了。

```php
use Symfony\Component\Notifier\NotifierInterface;

final class UserBanService {

    public function __construct(
        private readonly UserRepository $repository
        private readonly NotifierInterface $notifier
    ) {}

    public function handle(string $userUuid): void
    {
        $user = $this->repository->findByPK($userUuid);

        $this->notifier->send(
            new UserBannedNotification(subject: 'Your profile banned for activity that violates rules'),
            $user
        );
    }
}
```

你也可以通过队列发送通知：

```php
$this->notifier->sendQueued(
    new UserBannedNotification(subject: 'Your profile banned for activity that violates rules'),
    $user
);
```

> **注意 (Note)**
> 队列通知将通过通知配置中的 `queueConnection` 发送。

## 自定义通知传输 (Custom notification transport)

在某些情况下，你可能需要使用 `symfony/notifier` 组件未提供的自定义传输，在这种情况下，你可以使用 `Spiral\Notifications\NotificationTransportRegistryInterface` 接口注册自定义传输。

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Notifications\NotificationTransportRegistryInterface;
use Spacetab\SmsaeroNotifier\SmsaeroTransportFactory;

class MyBootloader extends Bootloader
{
    public function boot(NotificationTransportRegistryInterface $registry): void
    {
        $registry->registerTransport(new SmsaeroTransportFactory());
    }
}
```
