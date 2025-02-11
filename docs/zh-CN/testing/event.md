# 测试 - 事件测试

Spiral 提供了一种方便的方式来测试 [spiral/event](../advanced/events.md) 组件。在测试调度事件的代码时，您可能希望阻止事件的监听器实际执行。您可以使用 `fakeEventDispatcher` 方法来实现这一点。

```php
private \Spiral\Testing\Events\FakeEventDispatcher $events;
    
protected function setUp(): void
{
    parent::setUp();
    $this->events = $this->fakeEventDispatcher();
}
```

它允许您用一个假的（模拟的）事件调度器替换应用程序容器中的默认事件调度器。这个假的事件调度器 `Spiral\Testing\Events\FakeEventDispatcher` 记录所有被调度的事件，并提供断言方法，您可以使用这些方法来检查是否调度了特定的事件以及调度的次数。

该方法允许您将一个事件类数组作为参数传递。当您传递特定的事件类列表时，假的事件调度器将仅记录并为那些特定事件提供断言方法。其他被调度的事件将由默认的事件调度器处理。当您只想测试代码中调度某些事件的特定部分，而不是应用程序中调度的所有事件时，此功能很有用。

```php
private \Spiral\Testing\Events\FakeEventDispatcher $events;
    
protected function setUp(): void
{
    parent::setUp();
    $this->events = $this->fakeEventDispatcher([UserRegistered::class]);
}
```

在执行您想要测试的代码之后，您可以使用 `assertDispatched`，`assertDispatchedTimes`，`assertNotDispatched` 和 `assertNothingDispatched` 方法来检查哪些事件被调度了。

```php
use Tests\TestCase;

final class UserServiceTest extends TestCase
{
    private \Spiral\Testing\Events\FakeEventDispatcher $events;

    protected function setUp(): void
    {
        parent::setUp();
        $this->events = $this->fakeEventDispatcher();
    }


    public function testUserShouldBeRegistered(): void
    {
        // Perform user registration ...

        $this->events->assertDispatched(UserRegistered::class);
    }
}
```

此外，您还可以将一个闭包传递给 `assertDispatched` 或 `assertNotDispatched` 方法来检查事件是否满足某个条件。例如，您可以检查事件的参数是否等于某个值。如果至少有一个事件满足该条件，那么断言将成功。

### 断言事件已调度

```php
// 断言一个或多个事件已被调度
$this->events->assertDispatched(SomeEvent::class);

// 基于真值测试回调断言一个或多个事件已被调度
$this->events->assertDispatched(
    UserRegistered::class, 
    static function(UserRegistered $event): bool {
        return $event->username === 'john_smith';
    }
);
```

### 断言事件被调度的特定次数

```php
$this->events->assertDispatchedTimes(UserRegistered::class, 1);
```

### 断言事件未被调度

```php
$this->events->assertNotDispatched(UserRegistered::class);

$this->events->assertNotDispatched(
    UserRegistered::class, 
    static function(UserRegistered $event): bool {
        return $event->username === 'john_smith';
    }
);
```

### 断言没有事件被调度

```php
$this->events->assertNothingDispatched();
```

### 获取所有已调度的事件

获取匹配真值测试回调的所有事件。

```php
$events = $this->events->dispatched(UserRegistered::class);

// 或者

$events =  $this->events->dispatched(
    UserRegistered::class, 
    static function(UserRegistered $event): bool {
        return $event->someParam === 100;
    }
);
```
