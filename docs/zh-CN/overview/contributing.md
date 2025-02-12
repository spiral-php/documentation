# 概述 — 贡献指南

您有兴趣帮助 Spiral 吗？这是一个开源项目，需要像您这样的开发人员的帮助来保持其运行并使其变得更好。有很多方法可以参与进来。

## 拉取请求

贡献框架的一种方式是在 GitHub 上提交拉取请求。如果您有修复或改进，可以提交一个[拉取请求](https://github.com/spiral/framework/pulls)，维护人员将对其进行审查。在提交拉取请求时，您应该注意以下几点要求：

### 声明严格类型

该框架要求所有 PHP 文件在文件顶部使用 `declare(strict_types=1);` 指令来声明[严格类型](https://www.php.net/manual/en/language.types.declarations.php#language.types.declarations.strict)。这有助于确保代码是类型安全的，并降低了出现错误和错误的风险。

以下是此指令应该如何使用的示例：

```php
<?php

declare(strict_types=1);

namespace App;

// ...
```

### 遵循 PSR-12 编码标准

确保您的代码遵循 [PSR-12](https://www.php-fig.org/psr/psr-12/) 编码标准。这是一组关于代码应该如何格式化和构建的指南，以便它保持一致且易于阅读。

#### 检查代码

```terminal
./vendor/bin/php-cs-fixer fix --config=.php-cs-fixer.dist.php -vvv --dry-run --using-cache=no
```

#### 自动修复代码风格

```terminal
./vendor/bin/php-cs-fixer fix --config=.php-cs-fixer.dist.php -vvv --using-cache=no
```

> **注意**
> 该包将强制使用 `CL` 结尾。

### 包含测试

向 Spiral 提交代码时需要记住的重要一点是包含测试。这有助于确保您提交的代码按预期工作，并且不会引起任何新问题，而且它使维护人员更容易验证您的代码是否正确。

#### 运行测试

```terminal
./vendor/bin/phpunit
```

### 使用 Psalm

在您提交任何代码之前，您必须使用 [Psalm](https://psalm.dev/) 来检查是否有任何错误或问题。这确保了您的代码是正确的，并遵循了最佳实践。Spiral 在检查代码时使用 Psalm 的错误级别，**"4"**。这是一个简单的步骤，以确保您的贡献达到标准。

#### 运行 Psalm

```terminal
./vendor/bin/psalm --no-cache
```

### 保持简单

在贡献给 Spiral 时，需要记住的另一件事是尽量保持您的代码尽可能简单和直接。维护人员更喜欢易于理解和实现的解决方案，而不是过于复杂的解决方案。因此，在提交拉取请求时，尝试考虑解决手头问题的最简单方法，并以这种方式呈现它。这将有助于您的拉取请求更快地被接受。

## 支持问题

如果您有任何问题或需要建议，请随时加入我们的 [Discord](https://discord.gg/V6EK4he) 频道，以获得框架维护人员和社区成员的支持。

<a href="https://discord.gg/V6EK4he"><img src="https://img.shields.io/badge/discord-chat-magenta.svg"></a>

## 问题

如果您在使用 Spiral 时遇到任何问题或安全漏洞，请报告它们。维护人员非常重视这些问题，并将尽最大努力尽快解决它们。您可以通过在 `spiral/framework` GitHub 存储库中打开一个[问题](https://github.com/spiral/framework/issues)来报告问题或漏洞。

## 商业支持

需要注意的是，Spiral 和所有相关组件都由 [Spiral Scout](https://spiralscout.com/) 维护。

如果您想支持 Spiral 的开发，您可以成为 [赞助商](https://github.com/sponsors/roadrunner-server)。

此外，如果您需要商业支持，您可以联系 Spiral Scout 团队，邮箱地址为 [team@spiralscout.com](mailto:team@spiralscout.com)。他们将很乐意帮助您解决您可能遇到的任何问题。

## 许可

Spiral 及其组件将无限期地保留在 [MIT 许可证](/license.md) 下。
