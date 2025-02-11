# 测试 - 队列测试

当测试发送任务的代码时，通常你希望确保发送了特定的任务，但实际上并不希望运行该任务。这是因为任务的执行可以在单独的测试中进行测试。

Spiral 提供了一种方便的方式来测试任务发送的 [spiral/queue](../queue/configuration.md) 组件。您可以使用 `fakeQueue` 方法来阻止任务被发送到实际的队列。

`fakeQueue` 方法允许您用应用程序容器中的一个假的队列连接提供程序替换默认的队列连接提供程序，特别是 `Spiral\Testing\Queue\FakeQueueManager` 类。这个假的队列连接提供程序会创建假的队列，记录所有被推送到它的任务。

然后，您可以使用该类提供的断言方法，如 `assertPushed`, `assertPushedOnQueue`, `assertPushedTimes`, `assertNotPushed` 和 `assertNothingPushed`，来检查是否推送了特定的任务，以及它们被推送了多少次。这些方法允许您在无需实际运行任务本身的情况下，测试应用程序在调度任务方面的行为。

```php
use Spiral\Mailer\Message;
use Tests\TestCase;

final class UserServiceTest extends TestCase
{
    private \Spiral\Testing\Queue\FakeQueueManager $connection;
    private \Spiral\Testing\Queue\FakeQueue $queue;

    protected function setUp(): void
    {
        parent::setUp();
        $this->connection = $this->fakeQueue();
        $this->queue = $this->connection->getConnection('default');
    }

    public function testUserShouldBeRegistered(): void
    {
        // Perform user registration ...

        $this->queue->assertPushed('mail.job', function (array $data) {
            return $data['handler'] instanceof \Spiral\SendIt\MailJob
                && $data['options']->getQueue() === 'mail'
                && $data['payload']['foo'] === 'bar';
        });
    }
}
```

### 断言任务已被推送

`assertPushed` 方法可用于检查特定的任务是否被推送。您还可以向此方法传递一个闭包，它将用作一个 "真值测试" 来检查被推送的任务是否满足某些条件。

```php
$this->queue->assertPushed('mail.job', function (array $data) {
    return $data['handler'] instanceof \Spiral\SendIt\MailJob
        && $data['options']->getQueue() === 'mail'
        && $data['payload']['foo'] === 'bar';
});
```

### 断言任务被推送到特定队列

此外，您可以使用 `assertPushedOnQueue` 方法来检查任务是否被推送到特定的队列。

```php
$this->queue->assertPushedOnQueue('mail', 'mail.job', function (array $data) {
    return $data['handler'] instanceof \Spiral\SendIt\MailJob
        && $data['payload']['foo'] === 'bar';
});
```

### 断言任务被推送了特定次数

并且您可以使用 `assertPushedTimes` 方法来检查一个任务被推送的次数。

```php
$this->queue->assertPushedTimes('mail.job', 2);
```

### 断言任务未被推送

您还可以使用 `assertNotPushed` 方法来检查特定的任务是否未被推送到队列

```php
$this->queue->assertNotPushed('mail.job', function (array $data) {
    return $data['handler'] instanceof \Spiral\SendIt\MailJob
        && $data['options']->getQueue() === 'mail'
        && $data['payload']['foo'] === 'bar';
});
```

以及 `assertNothingPushed` 方法来检查是否没有任何任务被推送到队列。这些方法对于确保在不应该使用某些任务或队列时它们没有被使用非常有用。

```php
$this->queue->assertNothingPushed();
```
