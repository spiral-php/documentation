# 测试 - Mailer 测试

Spiral 有一种方便的测试 [spiral/sendit](../component/sendit.md) 组件的方法。一个有用的
特性是使用一个假邮件发送器，防止实际的电子邮件被发送。这很有用，因为发送电子邮件
通常与你正在测试的代码无关。

`fakeMailer()` 方法用一个假的邮件发送器
`Spiral\Testing\Mailer\FakeMailer` 替换 `Spiral\Mailer\MailerInterface`。它提供了有用的断言方法来检查是否发送了电子邮件，
发送了多少次以及电子邮件包含什么。

这是一个例子，说明你如何在用户注册过程中使用这个方法：

```php
use Spiral\Mailer\Message;
use Tests\TestCase;

final class UserServiceTest extends TestCase
{
    private \Spiral\Testing\Mailer\FakeMailer $mailer;

    protected function setUp(): void
    {
        parent::setUp();
        $this->mailer = $this->fakeMailer();
    }

    public function testRegisterUser(): void
    {
        // Perform user registration ...

        $this->mailer->assertSent(WelсomeMessage::class);
    }
}
```

`assertSent()` 方法提供了一种方法来检查是否发送了特定的邮件，并检查它是否包含某些
信息。你可以向它传递一个闭包来指定一个邮件必须满足的条件，以便断言成功。

```php
use Spiral\Mailer\Message;

$this->mailer->assertSent(
    WelcomeMessage::class,
    static fn (Message $message) => \in_array('user@site.com', $message->getTo())
);
```

例如，它被用来检查是否发送了 `WelcomeMessage` 类的邮件，并传递了一个闭包，该闭包
检查收件人电子邮件地址是否为 `user@site.com`。如果至少发送了一个 `WelcomeMessage` 类的邮件，
并且收件人电子邮件地址为 `user@site.com`，则断言将成功。

此外，还有 `assertSentTimes()` 方法。它可以用来断言一个特定的邮件被发送了
一定次数。它接受两个参数：邮件的类名和它应该被发送的预期次数。

```php
$this->mailer->assertSentTimes(WelcomeMessage::class, 2);
```

是的，此外，`FakeMailer` 还提供了 `assertNotSent()` 和 `assertNothingSent()` 方法。这些方法可以
用来断言没有发送特定的邮件，或者根本没有发送任何邮件。

`assertNotSent()` 方法用于断言没有发送特定的邮件。它接受与 `assertSent()` 方法相同的闭包，但它断言闭包应该对所有已发送的邮件失败。

```php
$this->mailer->assertNotSent(
    WelcomeMessage::class,
    static fn (Message $message) => \in_array('user@site.com', $message->getTo())
);
```

另一方面，`assertNothingSent()` 方法断言在测试用例期间没有发送任何邮件。
如果你想确保在你的测试期间没有发送不需要的邮件，这很有用。

```php
$this->mailer->assertNothingSent();
```
