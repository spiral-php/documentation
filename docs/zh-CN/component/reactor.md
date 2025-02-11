# 组件 — 代码生成

你可以使用组件 `spiral/reactor` 通过流畅的声明式封装器生成 PHP 类代码。该组件基于 `nette/php-generator`。

## 安装

安装该组件：

```terminal
composer require spiral/reactor
```

> **注意**
> 请注意，spiral/framework >= 2.7 已经包含了这个组件。

## 类声明

要声明一个类，使用 `Spiral\Reactor\ClassDeclaration`。

```php
use Spiral\Reactor\ClassDeclaration;

$class = new ClassDeclaration('MyClass');

dump($class->render()); // or dump((string) $class);
```

输出：

```php
class MyClass
{
}
```

你可以直接从类中访问大部分声明。

### 属性

要渲染类属性：

```php
$class = new ClassDeclaration('MyClass');

$class->addProperty('property', 'default')
    ->setProtected()
    ->setReadOnly()
    ->setType('string')
    ->setComment(['My property.', '@var string'])
    ->addAttribute('SomeAttribute');

dump((string) $class);
```

输出：

```php
class MyClass
{
    /**
     * My property.
     * @var string
     */
    #[SomeAttribute]
    protected readonly string $property = 'default';
}
```

### 常量

要渲染常量：

```php
$class = new ClassDeclaration('MyClass');

$class->addConstant('MY_CONSTANT', 'default')
    ->setPublic()
    ->setFinal()
    ->addAttribute('SomeAttribute')
    ->setComment(['My constant']);

dump((string) $class);
```

输出：

```php
class MyClass
{
    /** My constant */
    #[SomeAttribute]
    final public const MY_CONSTANT = 'default';
}
```

### 特质

要添加特质声明：

```php
$class = new ClassDeclaration('MyClass');

$class->addTrait(PrototypeTrait::class);

dump((string) $class);
```

输出：

```php
class MyClass
{
    use Spiral\Prototype\Traits\PrototypeTrait;
}
```

### 接口和扩展

要实现给定的接口或扩展一个基类：

```php
use Spiral\Reactor\ClassDeclaration;
use Cycle\ORM\Select\Repository;

$class = new ClassDeclaration('MyClass');

$class
    ->addImplement(\Countable::class)
    ->setExtends(Repository::class);

dump((string) $class);
```

输出：

```php
class MyClass extends Cycle\ORM\Select\Repository implements Countable
{
}
```

### 方法

要生成一个类方法：

```php
$class = new ClassDeclaration('MyClass');

$class->addMethod('ping')
    ->setPublic()
    ->setComment('My method')
    ->setReturnType('string')
    ->setReturnNullable()
    ->setFinal()
    ->setBody('return $a;')
    ->addAttribute('SomeAttribute')
        ->addParameter('a', null)
        ->setType('string')
        ->setNullable(true);

dump((string) $class);
```

输出：

```php
class MyClass
{
    /**
     * My method
     */
    #[SomeAttribute]
    final public function ping(?string $a = null): ?string
    {
        return $a;
    }
}
```

## 命名空间

要在特定的命名空间内创建一个类：

```php
use Spiral\Reactor\Partial\PhpNamespace;

$namespace = new PhpNamespace('App\\Some');
$namespace->addClass('MyClass')

dump((string) $namespace);
 ```

输出：

```php
namespace App\Some;

class MyClass
{
}
```

## 接口声明

要声明一个接口，使用 `Spiral\Reactor\InterfaceDeclaration`。

```php
$interface = new InterfaceDeclaration('MyInterface');
$interface
    ->addExtend(\Countable::class)
    ->addComment('My interface')
    ->addMethod('someMethod')
        ->setPublic()
        ->setReturnType('int');

dump((string) $interface);
```

输出：

```php
/**
 * My interface
 */
interface MyInterface extends Countable
{
    public function someMethod(): int;
}
```

## 枚举声明

要声明枚举，使用 `Spiral\Reactor\EnumDeclaration`。

```php
$enum = new EnumDeclaration('MyEnum');

$enum->addCase('First', 'first');
$enum->addCase('Second', 'second');

$enum
    ->setType('string')
    ->addConstant('FOO', 'bar')
    ->addComment('Description of enum')
    ->addAttribute('SomeAttribute');
$enum
    ->addMethod('getCase')
    ->setReturnType('string')
    ->addBody('return self::First->value;');

dump((string) $enum);
```

输出：

```php
/**
 * Description of enum
 */
#[SomeAttribute]
enum MyEnum: string
{
    public const FOO = 'bar';

    case First = 'first';
    case Second = 'second';

    public function getCase(): string
    {
        return self::First->value;
    }
}
```

## 函数声明

要声明一个全局函数，使用 `Spiral\Reactor\FunctionDeclaration`。

```php
$function = new FunctionDeclaration('myFunction');
$function
    ->addBody('return \'Hello world\';')
    ->setReturnType('string')
    ->addAttribute('SomeAttribute')
    ->addComment('Some function');

dump((string) $function);
```

输出：

```php
/**
 * Some function
 */
#[SomeAttribute]
function myFunction(): string
{
    return 'Hello world';
}

```

## 特质声明

要声明一个特质，使用 `Spiral\Reactor\TraitDeclaration`。

```php
$trait = new TraitDeclaration('MyTrait');
$trait
    ->setComment('Some trait')
    ->addMethod('myMethod')
        ->setPublic()
        ->setReturnType('void');

dump((string) $trait);
```

输出：

```php
/**
 * Some trait
 */
trait MyTrait
{
    public function myMethod(): void
    {
    }
}
```

## 文件

你可以在一个文件中收集类、接口、特质、全局函数和枚举：

```php
$namespace = new PhpNamespace('MyNamespace');
$namespace
    ->addUse(\Countable::class)
    ->addUse(Repository::class, 'Repo') // with alias
    ->addUseFunction('count');

$class = $namespace->addClass('MyClass');
$class
    ->addImplement(\Countable::class)
    ->addMethod('count')
        ->setReturnType('int')
        ->addBody('return 1;');

$file = new FileDeclaration();
$file
    ->setStrictTypes()
    ->addNamespace($namespace);

dump((string) $file);
```

输出：

```php
namespace MyNamespace;

use Countable;
use Cycle\ORM\Select\Repository as Repo;
use function count;

class MyClass implements Countable
{
    public function count(): int
    {
        return 1;
    }
}
```
