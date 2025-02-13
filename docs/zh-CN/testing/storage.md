# 测试 — Storage 测试

Spiral 提供了一种便捷的方式来测试 [spiral/storage](../advanced/storage.md) 组件。

`fakeStorage` 方法允许您创建一个模拟真实存储行为的假存储，但实际上不会将任何文件发送到云端。这样，您就可以测试文件上传，而不必担心意外地将真实文件发送到云端。

```php
use Tests\TestCase;

final class UserServiceTest extends TestCase
{
    private \Spiral\Testing\Storage\FakeBucket $bucket;

    protected function setUp(): void
    {
        parent::setUp();
        $this->bucket = $this->fakeStorage()->bucket('avatars');
    }


    public function testUserShouldBeRegistered(): void
    {
        // Perform user registration ...

        $this->bucket->assertCreated('avatars/john_smith.jpg');
    }
}
```

然后，您可以使用类提供的断言方法，例如 `assertExists`, `assertCreated`, `assertNotExist`, `assertNotCreated` 等，来检查特定文件是否存在于存储桶中。

### 断言文件存在

`assertExists` 方法断言指定文件存在于存储桶中。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertExists('image.jpg');
```

### 断言文件不存在

`assertNotExist` 方法断言指定文件不存在于存储桶中。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertNotExist('image.jpg');
```

### 断言文件已创建

`assertCreated` 方法断言指定文件已在存储桶中创建。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertCreated('image.jpg');
```

### 断言文件未创建

`assertNotCreated` 方法断言指定文件未在存储桶中创建。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertNotCreated('image.jpg');
```

### 断言文件已删除

`assertDeleted` 方法断言指定文件已从存储桶中删除。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertDeleted('image.jpg');
```

### 断言文件未删除

`assertNotDeleted` 方法断言指定文件未从存储桶中删除。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertNotDeleted('image.jpg');
```

### 断言文件已移动

`assertMoved` 方法断言指定文件已从一个位置移动到另一个位置。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertMoved('file.txt', 'folder/file.txt');
```

### 断言文件未移动

`assertNotMoved` 方法断言指定文件未从一个位置移动到另一个位置。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertNotMoved('file.txt', 'folder/file.txt');
```

### 断言文件已复制

`assertCopied` 方法断言指定文件已从一个位置复制到另一个位置。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertCopied('file.txt', 'folder/file.txt');
```

### 断言文件未复制

`assertNotCopied` 方法断言指定文件未从一个位置复制到另一个位置。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertNotCopied('file.txt', 'folder/file.txt');
```

### 断言文件可见性已更改

`assertVisibilityChanged` 方法断言指定文件的可见性已更改。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertVisibilityChanged('file.txt');
```

### 断言文件可见性未更改

`assertVisibilityNotChanged` 方法断言指定文件的可见性未更改。

```php
$uploads = $storage->bucket('uploads');
$uploads->assertVisibilityNotChanged('file.txt');
```
