# 测试 — 数据库填充

当你构建使用数据库的应用程序时，确保数据库正常工作非常重要。这意味着检查它是否以预期的方式存储、更改和返回数据。但有时，测试数据库可能会很棘手，甚至有点无聊。你可能需要编写大量复杂的命令，并非常小心地添加或删除数据。

这就是 Spiral 的数据库测试工具派上用场的地方。它们使这项工作变得更容易、更快捷。Spiral 在一个名为 [spiral-packages/database-seeder](https://github.com/spiral-packages/database-seeder) 的软件包中提供了一组特殊的工具。它旨在帮助你轻松地测试数据库。

## Database Seeder 工具提供什么

1. **轻松测试：** 使用 Spiral，你不需要处理复杂的命令。这些工具使用简单，这意味着你可以更容易地编写和理解你的测试。

2. **重置数据库的不同方法：** 测试完某些东西后，你需要再次清理你的数据库以进行下一个测试。Spiral 有不同的方法可以做到这一点，例如 Transaction（事务）、Migration（迁移）、Refresh（刷新）和 SqlFile（Sql 文件）方法。每种方法都有其自己的工作方式，因此你可以选择最适合你的测试的方法。

3. **Seeder（填充器）和 Factory（工厂）：** 这些就像用测试数据填充数据库的快捷方式。这些数据看起来就像你在应用程序中使用的真实数据。你可以使用这些工具快速设置测试所需的数据。

4. **检查你的数据库：** 在你对数据库进行操作之后，你想确保它工作正常。Spiral 的工具允许你检查数据是否存在，以及你的数据库结构是否正确。

无论你有多少经验，Spiral 的数据库测试工具对任何开发人员都非常有用。它们有助于确保你的数据库正在做它应该做的事情，这对于你的应用程序正常运行非常重要。

## 安装

要安装该软件包，请运行以下命令：

```terminal
composer require spiral-packages/database-seeder --dev
```

> **注意**
> 重要的是要注意 `--dev` 标志用于开发和测试环境。

安装完软件包后，你需要从该软件包注册 Bootloader（启动加载器）。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders（框架 - 启动加载器）](../framework/bootloaders.md) 部分阅读更多关于 Bootloader 的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders（框架 - 启动加载器）](../framework/bootloaders.md) 部分阅读更多关于 Bootloader 的信息。
:::

::::

我们还建议你将以下内容添加到你的 `composer.json` 文件中。这允许你将你的 Factory（工厂）和 Seeder（填充器）与应用程序的其余代码分开。

```json
{
  // ...
  "autoload": {
    "psr-4": {
      "Database\\": "app/database/"
    }
  }
}
```

完成这些步骤后，你已经安装了该软件包并注册了必需的 Bootloader，因此你现在可以在你的应用程序中使用该软件包。

## 设置数据库测试环境

### 环境变量

在 Spiral 环境中运行单元测试时，提高效率的一个关键步骤是正确配置环境变量。具体来说，在你的 `phpunit.xml` 文件中将 `CYCLE_SCHEMA_CACHE` 和 `TOKENIZER_CACHE_TARGETS` 设置为 `true` 可以显著提高测试速度。这些设置可以防止不必要的重复操作，例如每次测试运行时的目录扫描和 Cycle ORM 缓存重建。

要应用这些优化，请相应地修改你的 phpunit.xml 文件。以下是设置它的一个示例：

```xml phpunit.xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit>
    <php>
        <env name="CYCLE_SCHEMA_CACHE" value="true"/>
        <env name="TOKENIZER_CACHE_TARGETS" value="true"/>
        <!-- ... 其他配置 ... -->
    </php>
</phpunit>
```

当你测试你的数据库时，一个关键步骤是正确设置你的环境。这意味着为每个测试准备你的数据库，然后在测试后将事情恢复原样。在 Spiral 中有几种策略可以帮助你做到这一点。让我们看看使用简单的测试用例作为示例，使用不同的方法来设置和重置你的数据库进行测试。

### 1. Transaction Strategy (事务策略)

它在运行第一个测试时为你的数据库设置测试环境。然后，对于每个测试，它都在一个事务中完成所有操作——一种受保护的空间。测试完成后，它会撤消在事务中发生的所有操作（回滚）。假设你正在测试向你的数据库添加一个新用户。事务策略将在一个事务中添加此用户，然后在测试后删除此新用户的任何痕迹，就好像从未添加过一样。

这是最快的方式，并且非常适合大多数测试。当你有很多不改变数据库结构但只是添加、更改或删除数据的测试时，它特别好用。

**这是一个使用它的测试用例的示例：**

```php
declare(strict_types=1);

namespace Tests;

use Spiral\DatabaseSeeder\Database\Traits\{
    DatabaseAsserts, Helper, ShowQueries, Transactions
};

abstract class DatabaseTestCase extends TestCase
{
    use Transactions,
        Helper,
        DatabaseAsserts,
        ShowQueries;
}
```

让我们检查一下使用的每个 trait：

- `Transactions` trait（事务 trait）：它管理将每个测试用例包装在数据库事务中。当测试开始时，它会启动一个事务。一旦测试完成，无论它通过还是失败，事务都会回滚。这确保了每个测试都是隔离的，并且在测试之后数据库保持不变，这使得测试彼此独立。它还处理在测试之前运行数据库迁移，设置必要的数据库模式。
- `Helper` trait（助手 trait）：它提供了一组用于与数据库交互的助手方法。它包括简化各种数据库操作的实用程序函数，使你更容易在测试中执行常见任务，而无需编写重复的代码。
- `DatabaseAsserts` trait（数据库断言 trait）：它提供了一系列断言，你可以使用它们来验证数据库的状态，例如检查记录是否存在，验证记录计数等。这对于确保你的数据库操作产生预期的结果至关重要。
- `ShowQueries` trait（显示查询 trait）：这对于调试特别有用。当你启用此 trait 时，它会将测试期间执行的所有 SQL 查询记录到终端。这可以帮助你了解测试期间数据库中发生的情况，并解决出现的任何问题。

### 2. Migration Strategy (迁移策略)

这种方法在每次测试之前使用迁移（准备数据库的步骤）来设置你的数据库。测试后，它会回滚这些迁移。它比 Transaction Strategy（事务策略）慢，但如果你需要测试数据库结构本身的更改，它会很有用。

**示例：** 如果你正在测试在你的数据库中创建一个新表，那么 Migration Strategy（迁移策略）将为测试创建此表，然后在之后将其删除。

**这是一个使用它的测试用例的示例：**

```php
declare(strict_types=1);

namespace Tests;

use Spiral\DatabaseSeeder\Database\Traits\{
    DatabaseAsserts, DatabaseMigrations, Helper, ShowQueries
};

abstract class DatabaseTestCase extends TestCase
{
    use DatabaseMigrations,
        Helper,
        DatabaseAsserts,
        ShowQueries;
}
```

- `DatabaseMigrations` trait（数据库迁移 trait）：它管理在每次测试之前运行数据库迁移，并在测试之后回滚它们。这确保了每个测试都是隔离的，并且在测试之后数据库保持不变，这使得测试彼此独立。它还处理在测试之前运行数据库迁移，设置必要的数据库模式。

### 3. Refresh Strategy (刷新策略)

此策略在每次测试后使用 `Spiral\DatabaseSeeder\Database\Cleaner` 清理你的数据库。如果你的数据库已经具有你需要的结构，并且你只是在测试数据更改，那么它很有用。它比 Transaction Strategy（事务策略）慢。

**这是一个使用它的测试用例的示例：**

```php
declare(strict_types=1);

namespace Tests;

use Spiral\DatabaseSeeder\Database\Traits\{
    DatabaseAsserts, Helper, RefreshDatabase, ShowQueries
};

abstract class DatabaseTestCase extends TestCase
{
    use RefreshDatabase,
        Helper,
        DatabaseAsserts,
        ShowQueries;
}
```

- `RefreshDatabase` trait（刷新数据库 trait）：它管理每次测试后清理数据库表（除了 `migrations` 表）。

### 4. SqlFile Strategy (Sql 文件策略)

这种方法使用一个 SQL 文件为你的数据库设置必要的结构和数据，以进行测试。当你有一个可以从文件中轻松加载的复杂设置时，它很方便，但它比 Transaction Strategy（事务策略）慢。

**示例：** 如果你需要一个包含许多表和数据的特定设置，SqlFile Strategy（Sql 文件策略）允许你从预先准备好的 SQL 文件中加载所有这些内容。

**这是一个使用它的测试用例的示例：**

```php
declare(strict_types=1);

namespace Tests;

use Spiral\DatabaseSeeder\Database\Traits\{
    DatabaseAsserts, Helper, DatabaseFromSQL, ShowQueries
};

abstract class DatabaseTestCase extends TestCase
{
    use DatabaseFromSQL,
        Helper,
        DatabaseAsserts,
        ShowQueries;
   
   protected function getPrepareSQLFilePath(): string
   {
       return __DIR__ . '/database/prepare.sql';
   }

    protected function getDropSQLFilePath(): string
    {
        return __DIR__ . '/database/prepare.sql';
    }
}
```

正如你所看到的，有很多方法可以设置和重置你的数据库以进行测试。你可以选择最适合你需求的方法。设置好数据库后，就可以开始定义你的 Factories（工厂）和 Seeders（填充器）了。

## 定义实体工厂

要定义一个填充工厂，你应该创建一个扩展 `Spiral\DatabaseSeeder\Factory\AbstractFactory` 类的类。

这是一个定义 `User` 实体的填充工厂的示例：

```php
namespace Database\Factory;

use App\Entity\User;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;
use Spiral\DatabaseSeeder\Factory\FactoryInterface;

/**
 * @implements FactoryInterface<User>
 */
final class UserFactory extends AbstractFactory
{
    public function entity(): string
    {
        return User::class;
    }
    
    public function makeEntity(array $definition): User
    {
        return new User(
            username: $definition['username']
        );
    }

    public function definition(): array
    {
        return [
            'firstName' => $this->faker->firstName(),
            'lastName' => $this->faker->lastName(),
            'birthday' => \DateTimeImmutable::createFromMutable($this->faker->dateTime()),
            'comments' => CommentFactory::new()->times(3)->make(), // 可以使用其他 factories（工厂）。
            // 小心，不允许循环依赖！
        ];
    }
}
```

> **注意**
> 从 **v2.4.0** 开始，你可以像上面的例子那样，为 Factory（工厂）定义 `@implements FactoryInterface<...>` 注释的返回类型。有了这个注释，你的 IDE 现在将提供建议性的自动完成选项，使代码交互更直观，更不易出错。
> ![image](https://github.com/spiral-packages/database-seeder/assets/773481/2da1fcf3-9214-4188-8993-1b4eaa65ac98)

`entity` 方法应该返回 Factory（工厂）负责创建的目标实体的完全限定类名。在某些情况下，Factory（工厂）可能会使用 `entity` 方法返回的类名来创建一个实体的新实例，而无需调用其构造函数。相反，它可以使用反射通过使用来自 `definition` 方法的数据直接设置实体的属性。

`makeEntity` 方法允许你通过其构造函数控制创建实体的过程。该方法将一个定义数组作为参数，该数组由 `definition` 方法生成。

`definition` 方法是你可以定义实体的所有属性的地方。该方法应返回一个数组，其中键是属性名称，值是应该为这些属性设置的值。这些值可以是硬编码的，也可以使用 [Faker](https://fakerphp.github.io/) 库生成，该库提供了各种虚假数据生成方法，例如姓名、地址、电话号码等。此方法负责生成将传递给 makeEntity 方法以构造实体对象或用于直接设置属性的定义数组。

## 使用实体 Factory（工厂）

可以通过调用 Factory（工厂）类的 `new` 方法来创建一个 Factory（工厂）：

```php
use Database\Factory\UserFactory;

$factory = UserFactory::new();
```

这将创建一个 Factory（工厂）的新实例。它提供了几个用于生成实体的有用方法。

你还可以将一个定义数组传递给 Factory（工厂）类的 new 方法。

```php
use Database\Factory\UserFactory;

$factory = UserFactory::new(['admin' => true]);
```

当你有一组通用的定义要在多个 Factory（工厂）中使用时，或者当你想要为将在特定情况下被覆盖的属性设置默认值时，这是一个有用的功能。

### 创建实体

`create` 方法创建实体数组，将它们存储在数据库中，并返回它们以供代码进一步使用。

```php
$users = $factory->times(10)->create();
```

`createOne` 方法创建一个实体，将其存储在数据库中，并返回它以供代码进一步使用。

```php
$user = $factory->createOne();
```

### 制作实体

`make` 方法创建实体数组并返回它们以供代码进一步使用，但不会将它们存储在数据库中。

```php
$users = $factory->times(10)->make();
```

`makeOne` 方法创建一个实体并返回它以供代码进一步使用，但不会将它存储在数据库中。

```php
$user = $factory->makeOne();
```

### 实体状态

`state` 方法允许开发人员轻松地为实体定义特定状态。它可用于在创建实体时在其上设置一组特定的属性。闭包返回的值将替换定义数组中的相应值，允许开发人员以特定方式轻松更改实体的状态。

> **注意**
> 它不是破坏性的，它只会更新在返回数组中传递的属性，而不会从定义数组中删除任何属性。

```php
$factory->state(fn(\Faker\Generator $faker, array $definition) => [
    'admin' => $faker->boolean(),
])->times(10)->create();
```

除了 `state` 方法之外，还有 `entityState` 方法。此方法允许开发人员使用该实体的可用方法更改实体对象的状态。它将一个闭包作为参数，该闭包应该接受实体作为参数，并应该返回修改后的实体。这允许开发人员充分利用其实体的面向对象特性，并使用已经在实体上定义的方法来更改其状态。

```php
$factory->entityState(static function(User $user) {
    return $user->markAsDeleted();
})->times(10)->create();
```

`state` 和 `entityState` 方法可以在 Factory（工厂）类中使用，以创建用于创建具有特定状态的实体的附加方法。通过创建这些附加方法，开发人员可以创建更具表现力和可读性的代码，并使其更容易理解代码的意图。

```php
final class UserFactory extends AbstractFactory
{
    // ....

    public function admin(): self
    {
        return $this->state(fn(\Faker\Generator $faker, array $definition) => [
            'admin' => true,
        ]);
    }
    
    public function fromCity(string $city): self
    {
        return $this->state(fn(\Faker\Generator $faker, array $definition) => [
            'city' => $city,
        ]);
    }
    
    public function deleted(): self
    {
        return $this->entityState(static function (User $user) {
            return $user->markAsDeleted();
        });
    }
    
    public function withBirthday(\DateTimeImmutable $date): self
    {
        return $this->entityState(static function (User $user) use ($date) {
            $user->birthday = $date;
            return $user;
        });
    }
}
```

使用这些方法的示例：

```php
$factory->admin()
    ->fromCity('New York')
    ->deleted()
    ->withBirthday(new \DateTimeImmutable('2010-01-01 00:00:00'))
    ->times(10)
    ->create();
```

Factory（工厂）可以在你的功能测试用例中使用，以在数据库中创建实体，而无需使用填充器。这在你想要为功能测试创建一组特定的测试数据时，或者当你想要使用一组特定的数据测试你的应用程序的行为时，非常有用。

这是一个在功能测试用例中使用 Factory（工厂）的示例：

```php
final class UserServiceTest extends DatabaseTestCase
{
    // ...

    public function testDeleteUser(): void
    {
        $user = UserFactory::new()
            ->fromCity('New York')
            ->createOne();

        $this->userService->delete(
            uuid: $user->uuid,
        );

        $this->assertEntity(User::class)->where([
            'uuid' => (string)$user->uuid,
            'deleted_at' => ['!=' => null],
        ])->assertExists();
    }
}
```

## 填充

该软件包提供了使用测试数据填充数据库的能力。为此，开发人员可以创建一个 Seeder（填充器）类并将其扩展自 `Spiral\DatabaseSeeder\Seeder\AbstractSeeder` 类。Seeder（填充器）类应该实现 `run` 方法，该方法返回一个生成器，其中包含要存储在数据库中的实体。

```php
namespace Database\Seeder;

use Spiral\DatabaseSeeder\Seeder\AbstractSeeder;
use Spiral\DatabaseSeeder\Attribute\Seeder;

#[Seeder]
final class UserTableSeeder extends AbstractSeeder
{
    public function run(): \Generator
    {
        foreach (UserFactory::new()->times(100)->make() as $user) {
            yield $user;
        }
        
        foreach (UserFactory::new()->admin()->times(10)->make() as $user) {
            yield $user;
        }
        
        foreach (UserFactory::new()->admin()->deleted()->times(1)->make() as $user) {
            yield $user;
        }
    }
}
```

> **注意**
> 不要忘记将 `#[Seeder]` 属性添加到你的 Seeder（填充器）类中。此属性用于向软件包注册 Seeder（填充器）类。如果你没有添加此属性，则不会注册 Seeder（填充器）类，并且不会执行 Seeder（填充器）。

Seeder（填充器）主要用于使用用于你的阶段服务器的测试数据填充数据库，为开发人员和利益相关者提供一组一致的数据以进行测试和使用。

它们在测试具有许多不同实体及其之间关系的复杂应用程序时特别有用。通过使用 Seeder（填充器）使用测试数据填充数据库，你可以确保你的测试针对一组一致且定义良好的数据运行，这可以帮助使你的测试更可靠，并且不易出现不稳定情况。

### 运行 Seeder（填充器）

使用以下命令运行 Seeder（填充器）：

```terminal
php app.php db:seed
```

这将执行所有已注册的 Seeder（填充器）类，并将测试数据插入到数据库中。

## 断言

该软件包提供了几个额外的断言方法，使用了 `Spiral\DatabaseSeeder\Database\Traits\DatabaseAsserts` trait。这些方法可用于测试执行某些操作后数据库的状态。

### 实体断言

可以使用 `assertEntity` 方法断言数据库中实体的状态。它获取实体类名，并提供用于测试数据库实体的方法：

#### `assertExists()`

检查数据库中是否存在该实体。

```php
$this->assertEntity(User::class)->where([
    'uuid' => (string)$user->uuid,
    'deleted_at' => ['!=' => null],
])->assertExists();
```

#### `assertMissing()`

检查数据库中不存在该实体。

```php
$this->assertEntity(User::class)->where([
    'uuid' => (string)$user->uuid,
])->assertMissing();
```

#### `assertCount(int $total)`

检查数据库中是否存在指定数量的实体项目。

```php
$this->assertEntity(User::class)->where([
    'uuid' => (string)$user->uuid,
])->assertCount(1);
```

#### `assertEmpty()`

检查数据库中是否没有实体项目。

```php
$this->assertEntity(User::class)->assertEmpty();
```

#### `where(array $where)`

允许你指定数据库中实体的条件。

```php
$this->assertEntity(User::class)->where([
    'uuid' => (string)$user->uuid,
])->...;
```

#### `count()`

返回数据库中实体项目的数量。

```php
$total = $this->assertEntity(User::class)->where([
    'deleted_at' => ['!=' => null],
])->count();
```

#### `select(\Closure $closure)`

允许你对数据库进行复杂的查询。

```php
$this->assertEntity(User::class)->select(function(\Cycle\ORM\Select $select) {
    $select->where('deleted_at', '!=', null);
})->...;
```

#### `withoutScope`

允许你禁用实体的全局范围。

```php
$this->assertEntity(User::class)->withoutScope()->...;
```

#### `withScope(ScopeInterface $scope)`

允许你启用实体的全局范围。

```php
$this->withScope(new SoftDeletedScope())->...;
```

### 表断言

#### `select(\Closure $closure)`

允许你对数据库进行复杂的查询。

```php
$this->assertTable('users')->select(function(\Cycle\Database\Query\SelectQuery $query) {
    $query->where('deleted_at', '!=', null);
})->...;
```

#### `where(array $where)`

允许你指定数据库中表的条件。

```php
$this->assertTable('users')->where([
    'id' => 5,
])->assertCount(1);
```

#### `assertRecordExists()`

允许你检查数据库中是否存在记录。

```php
$this->assertTable('users')->where([
    'id' => 5,
])->assertRecordExists();
```

#### `assertCountRecords(int $total)`

允许你检查数据库中是否存在指定数量的记录。

```php
$this->assertTable('users')->where([
    'id' => 5,
])->assertCountRecords(2);
```

#### `countRecords()`

返回数据库中记录的数量。

```php
$total = $this->assertTable('users')->where([
    'deleted_at' => ['!=' => null],
])->countRecords();
```

#### `assertEmpty()`

允许你检查数据库中是否没有记录。

```php
$this->assertTable('users')->where([
    'id' => 5,
])->assertEmpty();
```

#### `assertExists()`

允许你检查数据库中是否存在表。

```php
$this->assertTable('users')->assertExists();
```

#### `assertColumnExists(string $column)`

允许你检查表中是否存在列。

```php
$this->assertTable('users')->assertColumnExists('id');
```

#### `assertColumnMissing(string $column)`

允许你检查表中是否缺少列。

```php
$this->assertTable('users')->assertColumnMissing('id');
```

#### `assertColumnSame(...)`

断言表中列与指定的特征匹配，包括类型、大小、精度、刻度、可空性和默认值。此方法特别适用于验证表的详细模式。

```php
$assertion = $this->assertTable('users');

$assertion->assertColumnSame(
     column: 'id', 
     type: 'uuid', 
     size: 64,
     ...
);
```

#### `assertIndexPresent(array $columns)`

断言表中指定列上存在索引。这对于确保必要的索引（可能会影响性能和唯一性约束）就位至关重要。

```php
$this->assertTable('users')->assertIndexPresent(['email']);
```

#### `assertForeignKeyPresent(array $columns)`

检查是否存在与指定的表列相关的外键。这对于验证数据库模式中的关系完整性约束很重要。

```php
$this->assertTable('users')->assertForeignKeyPresent(['org_id']);
```

#### `assertPrimaryKeyExists(string ...$columns)`

断言指定的列是表主键的一部分。这对于确保主键的正确配置至关重要，主键对于表中记录的标识和完整性至关重要。

```php
$this->assertTable('users')->assertPrimaryKeyExists('id');
```

## 控制台命令

该软件包提供了几个控制台命令，用于快速创建 Factory（工厂）、Seeder（填充器）。这些命令可以轻松地快速创建用于填充和测试你的应用程序的必要类，而无需手动创建文件或确保包含必要的样板代码。

### 创建 Factory（工厂）

`Spiral\DatabaseSeeder\Console\Command\FactoryCommand` 控制台命令用于创建 Factory（工厂）。Factory（工厂）的名称作为参数传递。

例如，要创建一个名为 `UserFactory` 的 Factory（工厂），该命令将是：

```terminal
php app.php create:factory UserFactory
```

### 创建 Seeder（填充器）

`Spiral\DatabaseSeeder\Console\Command\SeederCommand` 控制台命令用于创建 Seeder（填充器）。Seeder（填充器）的名称作为参数传递。

例如，要创建一个名为 `UserSeeder` 的 Seeder（填充器），该命令将是：

```terminal
php app.php create:seeder UserSeeder
```
