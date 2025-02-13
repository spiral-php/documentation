# 安全性 — 数据加密

默认情况下，Web 和 GRPC 应用程序框架都默认启用了 Encryper 组件。要在其他构建中安装 Encrypter，请执行：

```terminal
composer require spiral/encrypter
```

> **注意**
> 请注意，spiral/framework >= 2.6 已经包含了这个组件。

您必须注册 `Spiral\Bootloader\Security\EncrypterBootloader` 这个 Bootloader 以激活该组件。

## 应用密钥

加密组件是基于 [defuse/php-encryption](https://github.com/defuse/php-encryption) 的，它需要一个由您的应用程序提供的加密密钥。

默认情况下，`EncrypterBootloader` 会从环境变量 `ENCRYPTER_KEY` 加载 Base64 编码的密钥。

```terminal
php app.php encrypt:key -m .env
```

> **注意**
> Encrypter 用于保护您的 cookie 值，更改密钥将自动使所有已发布的 cookie 无效。

## 用法

您可以通过 `Spiral\Encrypter\EncrypterInterface` 在应用程序中使用 Encrypter：

```php
/**
 * 不可变类，负责加密服务。
 */
interface EncrypterInterface
{
    /**
     * 创建并使用新密钥生成加密器实例。
     *
     * @throws EncrypterException
     */
    public function withKey(string $key): EncrypterInterface;

    /**
     * 加密密钥值。以 Base64 字符串格式返回。
     */
    public function getKey(): string;

    /**
     * 加密数据。使用 encrypt() 方法加密。
     *
     * @param mixed $data
     *
     * @throws EncrypterException
     */
    public function encrypt($data): string;

    /**
     * 解密负载字符串。负载应该通过 encrypt() 方法生成。
     *
     * @return mixed
     *
     * @throws DecryptException
     * @throws EncrypterException
     */
    public function decrypt(string $payload);
}
```

您也可以在您的应用程序中使用 `encrypter` 属性，它可以通过 `Spiral\Encrypter\EncrypterInterface` 访问:

```php
/**
 * Immutable class responsible for encryption services.
 */
interface EncrypterInterface
{
    /**
     * 创建并使用新密钥生成加密器实例。
     *
     * @throws EncrypterException
     */
    public function withKey(string $key): EncrypterInterface;

    /**
     * 加密密钥值。以 Base64 字符串格式返回。
     */
    public function getKey(): string;

    /**
     * 加密数据。使用 encrypt() 方法加密。
     *
     * @param mixed $data
     *
     * @throws EncrypterException
     */
    public function encrypt($data): string;

    /**
     * 解密负载字符串。负载应该通过 encrypt() 方法生成。
     *
     * @return mixed
     *
     * @throws DecryptException
     * @throws EncrypterException
     */
    public function decrypt(string $payload);
}
```
