# Centrifugo API

Spiral 提供了通过其 GRPC API 向 Centrifugo 发送各种命令的便捷方法。 Centrifugo 本身支持 GRPC API，允许使用压缩的二进制格式进行命令的通信。

这种方法利用了 HTTP/2 的强大功能，HTTP/2 是 GRPC 的底层传输协议。

> **注意**
> 要了解更多关于不同可用的 API 方法，请参考 Centrifugo 关于服务器 API 的文档，可以在 [Centrifugo 网站](https://centrifugal.dev/docs/server/server_api) 上找到。

## 用法

`RoadRunner\Centrifugo\CentrifugoApiInterface` 提供了允许与 Centrifugo 服务器轻松通信的一组方法。

您可以从容器中请求 `CentrifugoApiInterface` 实例。

**以下是一个如何执行此操作的示例：**

```php
final class SubscribeUserToChannelHandler
{
    public function __construct(
        private \RoadRunner\Centrifugo\CentrifugoApiInterface $api
    ) {
    }

    public function handle(User $user, Channel $channel): void
    {
        $this->api->subscribe($channel->getName(), $user->getId());
    }
}
```

## 可用的 API 方法

让我们来了解一下可用的方法：

### Publish (发布)

此方法用于将数据发布到特定频道。

**它接受以下参数：**

- `$channel`: 一个非空字符串，表示要将消息发布到的频道名称。
- `$message`: 一个字符串，表示要发布的 JSON 编码的消息。
- `$skipHistory`: 一个布尔值，指示是否跳过将消息添加到频道的历史记录。 默认为 `true`。
- `$tags`: 一个键值对数组，表示要附加到消息的附加元数据。 默认为一个空数组。

以下是使用此方法的一个示例：

```php
public function handle(User $user, Channel $channel): void
{
    $this->api->publish($channel->getName(), 'Hello world!');
}
```

> **更多信息**
> 请参考 [Centrifugo 文档](https://centrifugal.dev/docs/server/server_api#publish) 了解更多信息。

### Broadcast (广播)

此方法类似于 publish 方法，但允许将相同的数据发送到多个频道。

**它接受以下参数：**

- `$channels`: 一个非空字符串数组，表示要将消息广播到的频道名称。
- `$message`: 一个字符串，表示要广播的 JSON 编码的消息。
- `$skipHistory`: 一个布尔值，指示是否跳过将消息添加到频道的历史记录。 默认为 `true`。
- `$tags`: 一个键值对数组，表示要附加到消息的附加元数据。 默认为一个空数组。

以下是使用此方法的一个示例：

```php
public function handle(User $user, Channel ...$channels): void
{
    $this->api->broadcast(
      \array_map(fn (Channel $channel) => $channel->getName(), $channels),
      'Hello world!'
    );
}
```

> **更多信息**
> 请参考 [Centrifugo 文档](https://centrifugal.dev/docs/server/server_api#broadcast) 了解更多信息。

### Subscribe (订阅)

此方法允许将用户订阅到指定的频道。

**它接受以下参数：**

- `$channel`: 要订阅的频道名称。必须是一个非空字符串。
- `$user`: 正在订阅的用户的 ID。 必须是一个字符串。
- `$expireAt` (可选): 订阅的可选过期日期/时间，表示为 `\DateTimeInterface` 对象。如果未提供，则订阅不会过期。
- `$info` (可选): 包含在订阅中的其他信息数组。
- `$client` (可选): 用于标识进行订阅的客户端的可选 ID。
- `$data` (可选): 包含在订阅中的其他数据数组。
- `$session` (可选): 用于标识订阅的会话的可选 ID。

以下是使用此方法的一个示例：

```php
public function handle(User $user, Channel $channel): void
{
    $this->api->subscribe($channel->getName(), $user->getId(), ...);
}
```

> **更多信息**
> 请参考 [Centrifugo 文档](https://centrifugal.dev/docs/server/server_api#subscribe) 了解更多信息。

### Unsubscribe (取消订阅)

此方法允许取消用户对频道的订阅。

它接受以下参数：

- `$channel`: 要取消订阅的频道名称。
- `$user`: 要取消订阅频道的用户。
- `$client` (可选): 用户的客户端 ID。 如果不需要，此参数是可选的，可以设置为 null。
- `$session` (可选): 用户的会话 ID。 如果不需要，此参数是可选的，可以设置为 null。

以下是使用此方法的一个示例：

```php
public function handle(User $user, Channel $channel): void
{
    $this->api->unsubscribe($channel->getName(), $user->getId(), ...);
}
```

> **更多信息**
> 请参考 [Centrifugo 文档](https://centrifugal.dev/docs/server/server_api#unsubscribe) 了解更多信息。

### Disconnect (断开连接)

此方法通过用户的 ID 断开连接。

它接受以下参数：

- `$user`: 要断开连接的用户的 ID。
- `$client` (可选): 要断开连接的客户端的 ID。 如果未提供，则将断开与用户关联的所有客户端的连接。
- `$whitelist` (可选): 不应断开连接的客户端 ID 数组。
- `$session` (可选): 要断开连接的会话的 ID。 如果未提供，则将断开与用户关联的所有会话的连接。
- `$disconnect` (可选): 包含有关断开连接事件信息的 Disconnect 对象。

以下是使用此方法的一个示例：

```php
public function handle(User $user): void
{
    $this->api->disconnect($user->getId(), ...);
}
```

> **更多信息**
> 请参考 [Centrifugo 文档](https://centrifugal.dev/docs/server/server_api#disconnect) 了解更多信息。

### Presence (在线状态)

此方法用于检索连接到频道的活动客户端的列表。

它接受以下参数：

- `$channel`: 一个非空字符串，表示要从中检索客户端列表的频道名称。

以下是使用此方法的一个示例：

```php
public function handle(Channel $channel): void
{
   $result = $this->api->presence($channel->getName());
   // ...
}
```

> **更多信息**
> 请参考 [Centrifugo 文档](https://centrifugal.dev/docs/server/server_api#presence) 了解更多信息。

### Presence Stats (在线状态统计)

此方法用于检索短的频道在线状态信息 - 客户端数量和唯一用户数量（基于用户 ID）。

它接受以下参数：

- `$channel`: 一个非空字符串，表示频道名称。

以下是使用此方法的一个示例：

```php
public function handle(Channel $channel): void
{
   $stats = $this->api->presenceStats($channel->getName());
   // ...
}
```

> **更多信息**
> 请参考 [Centrifugo 文档](https://centrifugal.dev/docs/server/server_api#presence_stats) 了解更多信息。

### Channels (频道列表)

此方法用于检索活动频道列表，这些频道具有一个或多个活动订阅者。

它接受以下参数：

- `$pattern` (可选): 如果提供了非空字符串，则仅返回与该模式匹配的频道。

以下是使用此方法的一个示例：

```php
public function handle(Channel $channel): void
{
   $channels = $this->api->channels();
   // ...
}
```
