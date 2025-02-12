# 高级特性 — 特性（Attributes）

PHP 中的特性（Attributes）是一种为类、属性、函数、常量和参数添加元数据的方式。这些元数据可用于提供关于元素的附加信息，或在特定情况下更改元素的行为。

`spiral/attributes` 组件提供了一种在 PHP 中使用特性的简单且一致的方式。

**在 PHP 应用程序中使用特性的好处：**

- 它们提供了一种将元数据添加到元素的方式，这种方式独立于元素的逻辑。
- 它们可以用于在特定情况下更改元素的行为，例如为属性提供额外的验证，或根据函数的特性更改其行为。
- 它们可以用于提供关于元素的附加信息，例如为类或属性提供文档。
- 通过结合使用特性和拦截器，开发者可以在其代码库中实现面向方面编程（Aspect-Oriented Programming）实践，从而带来更小、更快、更容易测试的代码。

**该组件还服务于两个非常重要的目的：**

- 能够在同一个地方组合不同类型的元数据。你无需关心开发者使用的是 [Doctrine 注解](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/index.html) 还是 PHP 8 中添加的 [PHP 特性](https://wiki.php.net/rfc/attributes_v2)。
- 能够在任何版本的语言中读取特性。这意味着即使你使用的是 PHP 7.2，你也可以立即使用 PHP 8 的特性。

该组件提供了一个元数据读取器桥接器，允许在同一个项目中使用现代的 [PHP 特性](https://wiki.php.net/rfc/attributes_v2) 和 [Doctrine 注解](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/index.html)。

> **注意：**
> 本文档使用术语“元数据”来同时指代“特性”和“注解”。

## 安装

要安装该组件：

```terminal 
composer require spiral/attributes
```

### 框架集成

要启用该组件，你只需要将 `Spiral\Bootloader\Attributes\AttributesBootloader` 类添加到引导程序列表中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Attributes\AttributesBootloader::class,
        // ...
    ];
}
```

在 [框架 — 引导程序](../framework/bootloaders.md) 部分阅读关于引导程序的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Attributes\AttributesBootloader::class,
    // ...
];
```

在 [框架 — 引导程序](../framework/bootloaders.md) 部分阅读关于引导程序的更多信息。
:::

::::

## 使用

该组件提供了一组用于处理特性的接口和类。
`Spiral\Attributes\ReaderInterface` 提供了从类、属性、函数、常量和参数中读取特性的一组方法。

```php
interface ReaderInterface
{
    public function getClassMetadata(\ReflectionClass $class, string $name = null): iterable;
    public function getPropertyMetadata(\ReflectionProperty $property, string $name = null): iterable;
    public function getFunctionMetadata(\ReflectionFunctionAbstract $function, string $name = null): iterable;
    public function getConstantMetadata(\ReflectionClassConstant $constant, string $name = null): iterable;
    public function getParameterMetadata(\ReflectionParameter $parameter, string $name = null): iterable;
    
    public function firstClassMetadata(\ReflectionClass $class, string $name): ?object;
    public function firstPropertyMetadata(\ReflectionProperty $property, string $name): ?object;
    public function firstFunctionMetadata(\ReflectionFunctionAbstract $function, string $name): ?object;
    public function firstConstantMetadata(\ReflectionClassConstant $constant, string $name): ?object;
    public function firstParameterMetadata(\ReflectionParameter $parameter, string $name): ?object;
}
```

### 类元数据

要读取类元数据，请使用 `$reader->getClassMetadata()` 方法。它接收作为输入的所需类的 `ReflectionClass`，并返回可用元数据对象的列表。

```php
$reflection = new ReflectionClass(User::class);

$attributes = $reader->getClassMetadata($reflection); 
// returns iterable<object>
```

该方法的第二个可选参数 `$name` 允许你指定要检索哪些特定的元数据对象。

```php
$reflection = new ReflectionClass(User::class);

$attributes = $reader->getClassMetadata($reflection, Entity::class); 
// returns iterable<Entity>
```

要获取一个元数据对象，你可以使用方法 `$reader->firstClassMetadata()`。

```php
$reflection = new ReflectionClass(User::class);

$attribute = $reader->firstClassMetadata($reflection, Entity::class); 
// returns Entity|null
```

从 **v2.10.0** 版本开始，支持从类中使用的 traits 读取特性。

:::: tabs

::: tab 特性

```php
#[Cycle\Entity]
class Entity {
    use TsTrait;
}

#[Behavior\CreatedAt]
#[Behavior\UpdatedAt]
trait TsTrait
{
    #[Cycle\Column(type: 'datetime')]
    private DateTimeImmutable $createdAt;

    #[Cycle\Column(type: 'datetime', nullable: true)]
    private ?DateTimeImmutable $updatedAt = null;
}
```

:::

::: tab 注解

```php
/**
 * @Cycle\Entity
 */
class Entity {
    use TsTrait;
}

/**
 * @Behavior\CreatedAt
 * @Behavior\UpdatedAt
 */
trait TsTrait
{
    #[Cycle\Column(type: 'datetime')]
    private DateTimeImmutable $createdAt;

    #[Cycle\Column(type: 'datetime', nullable: true)]
    private ?DateTimeImmutable $updatedAt = null;
}
```

:::

::::

### 属性元数据

要读取属性元数据，请使用 `$reader->getPropertyMetadata()` 方法。它接收作为输入的所需属性的 `ReflectionProperty`，并返回可用元数据对象的列表。

```php
$reflection = new ReflectionProperty(User::class, 'name');

$attributes = $reader->getPropertyMetadata($reflection); 
// returns iterable<object>
```

该方法的第二个可选参数 `$name` 允许你指定要检索哪些特定的元数据对象。

```php
$reflection = new ReflectionProperty(User::class, 'name');

$attributes = $reader->getPropertyMetadata($reflection, Column::class); 
// returns iterable<Column>
```

要获取一个元数据对象，你可以使用方法 `$reader->firstPropertyMetadata()`。

```php
$reflection = new ReflectionProperty(User::class, 'name');

$column = $reader->firstPropertyMetadata($reflection, Column::class); 
// returns Column|null
```

### 函数元数据

要读取函数元数据，请使用 `$reader->getFunctionMetadata()` 方法。它接收一个 `ReflectionFunction` 或 `ReflectionMethod` 类型的参数，代表所需的函数，并返回可用元数据对象的列表。

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$attributes = $reader->getFunctionMetadata($reflection); 
// returns iterable<object>
```

该方法的第二个可选参数 `$name` 允许你指定要检索哪些特定的元数据对象。

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$attributes = $reader->getPropertyMetadata($reflection, DTOGetter::class); 
// returns iterable<DTOGetter>
```

要获取一个元数据对象，你可以使用方法 `$reader->firstFunctionMetadata()`。

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$getter = $reader->firstFunctionMetadata($reflection, DTOGetter::class); 
// returns DTOGetter|null
```

### 常量元数据

要读取类常量元数据，请使用 `$reader->getConstantMetadata()` 方法。它接收一个 `ReflectionClassConstant` 类型的参数，代表所需的函数，并返回可用元数据对象的列表。

```php
$reflection = new ReflectionClassConstant(Example::class, 'CONSTANT_NAME');

$attributes = $reader->getConstantMetadata($reflection); 
// returns iterable<object>
```

该方法的第二个可选参数 `$name` 允许你指定要检索哪些特定的元数据对象。

```php
$reflection = new ReflectionClassConstant(Example::class, 'CONSTANT_NAME');

$attributes = $reader->getConstantMetadata($reflection, Deprecated::class); 
// returns iterable<Deprecated>
```

要获取一个元数据对象，你可以使用方法 `$reader->firstConstantMetadata()`。

```php
$reflection = new ReflectionClassConstant(Example::class, 'CONSTANT_NAME');

$getter = $reader->firstConstantMetadata($reflection, Deprecated::class); 
// returns Deprecated|null
```

### 参数元数据

要读取函数/方法参数元数据，请使用 `$reader->getParameterMetadata()` 方法。它接收一个 `ReflectionParameter` 类型的参数，代表所需的函数，并返回可用元数据对象的列表。

```php
$reflection = new ReflectionParameter('send_email', 'email');

$attributes = $reader->getParameterMetadata($reflection); 
// returns iterable<object>
```

该方法的第二个可选参数 `$name` 允许你指定要检索哪些特定的元数据对象。

```php
$reflection = new ReflectionParameter('send_email', 'email');

$attributes = $reader->getParameterMetadata($reflection, PreCondition::class); 
// returns iterable<PreCondition>
```

要获取一个元数据对象，你可以使用方法 `$reader->firstParameterMetadata()`。

```php
$reflection = new ReflectionParameter('send_email', 'email');

$getter = $reader->firstConstantMetadata($reflection, PreCondition::class); 
// returns PreCondition|null
```

## 创建注解

> **注意**
> 关于使用 doctrine 注解的详细信息，请参阅
> [doctrine 文档](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/index.html#create-an-annotation-class)。

你应该使用“混合”语法来创建元数据类，以便在任何版本的 PHP 上都能使用。

```php
/**
 * @Annotation
 * @Target({ "CLASS" })
 */
#[\Attribute(\Attribute::TARGET_CLASS)]
class MyEntityMetadata
{
    public string $table;
}
```

在这种情况下，你可以在任何 PHP 版本上使用这个元数据类。

:::: tabs

::: tab 特性

```php
#[MyEntityMetadata(table: 'users')] 
class User {}
```

:::

:::: tab 注解

```php
/**
 * @MyEntityMetadata(table="users")
 */
class User {}
```

:::

::::

### 实例化

该包支持不同的实例化特性方式，但默认情况下它使用 Doctrine 逻辑来实现兼容性。

假设你像这样使用你的元数据类，传递一个字段“`property`”，其字符串值为“`value`”。

:::: tabs

::: tab 特性

```php
#[CustomMetadataClass(property: "value")]
class AnnotatedClass
{
}
```

:::

::: tab 注解

```php
/** @CustomMetadataClass(property="value") */
class AnnotatedClass
{
}
```

:::

::::

在这种情况下，注解类本身可能如下所示：

#### Doctrine 基本实例化器

在这种情况下，当声明一个元数据类时，特性/注解属性将被填充。

> **注意**
> 另请参阅
> [Doctrine 自定义注解](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/custom.html#custom-annotation-classes)

```php
/** @Annotation */
#[\Attribute]
class CustomMetadataClass
{
    public $property;
}
```

#### Doctrine 构造函数实例化器

在构造函数声明的情况下，使用元数据类的所有数据都将作为数组传递给此构造函数。

> **注意**
> 另请参阅
> [Doctrine 自定义注解](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/custom.html#custom-annotation-classes)

```php
/** @Annotation */
#[\Attribute]
class CustomMetadataClass
{
    public function __construct(array $properties)
    {
        // $properties = [ "property" => "value" ]
    }
}
```

#### 命名参数（接口标记）

如果你想使用命名构造函数参数（另请参阅
[PHP 手册 — 命名参数](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments)），
那么你必须将接口 `Spiral\Attributes\NamedArgumentConstructorAttribute` 添加到元数据类中，该接口将标记所需的元数据类为一个接受命名参数的类。

```php
/** @Annotation */
#[\Attribute]
class CustomMetadataClass implements \Spiral\Attributes\NamedArgumentConstructorAttribute
{
    public function __construct($property)
    {
        // $property = "value"
    }
}
```

#### 命名参数（元数据标记）

请注意，使用先前的方法将需要你拥有一个 `spiral/attributes` 包，并且你将无法在缺少此包的其他项目中使用这些类。

为了解决这个问题，你可以使用元数据类，这将意味着相同的事情（元数据类使用命名参数），但不会直接实现接口，因此不需要项目中的 `spiral/attributes` 包。

```php
/**
 * @Annotation
 * @Spiral\Attributes\NamedArgumentConstructor
 */
#[\Attribute]
#[\Spiral\Attributes\NamedArgumentConstructor]
class CustomMetadataClass
{
    public function __construct($property)
    {
        // $property = "value"
    }
}
```

## 驱动程序

`Spiral\Attributes\Factory` 封装了几个实现，并且默认情况下返回一个 [选择性读取器](#选择性读取器) 实现，该实现适用于大多数情况。但是，如果你的平台和/或应用程序上提供，你可以要求一个特定的实现。

```php
use Spiral\Attributes\Factory;

$reader = (new Factory())->create();
```

### 注解读取器

> **注意**
> 请注意，为了使此读取器在应用程序中可用，你需要引入 "doctrine/annotations" 组件。

此读取器实现允许读取 doctrine 注解。

```php
/** @ExampleAnnotation */
class Example {}

$reader = new \Spiral\Attributes\AnnotationReader();

$annotations = $reader->getClassMetadata(new ReflectionClass(Example::class));
// returns iterable<ExampleAnnotation>
```

### 特性读取器

此读取器实现允许在任何 PHP 版本上读取原生 PHP 特性。

```php
#[ExampleAttribute]
class Example {}

$reader = new \Spiral\Attributes\AttributeReader();

$attributes = $reader->getClassMetadata(new ReflectionClass(Example::class));
// returns iterable<ExampleAttribute>
```

### 选择性读取器

该实现会自动根据应用程序中使用的语法选择正确的读取器。如果同时在同一个项目中使用两种语法，则需要此行为。例如，在已经运行的项目中，将注解语法转换为现代的特性语法。

:::: tabs

::: tab 特性

```php
#[ExampleAttribute]
class ClassWithAttributes {}
```

:::

::: tab 注解

```php
/** @ExampleAnnotation */
class ClassWithAnnotations {}
```

:::

::::

```php
$reader = new \Spiral\Attributes\Composite\SelectiveReader([
    new \Spiral\Attributes\AnnotationReader(),
    new \Spiral\Attributes\AttributeReader(),
]);

$annotations = $reader->getClassMetadata(new ReflectionClass(ClassWithAnnotations::class));
// returns iterable<ExampleAnnotation>

$attributes = $reader->getClassMetadata(new ReflectionClass(ClassWithAttributes::class));
// returns iterable<ExampleAttribute>
```

> **注意**
> 当在同一位置使用注解和特性时，此读取器的行为是不确定的。

### 合并读取器

该读取器的实现将几种语法合并为一个。如果你同时使用多个库，并且这些库只支持旧语法或新语法，则需要此行为。

```php
/** @DoctrineAnnotation */
#[NativeAttribute]
class ExampleClass {}

$reader = new \Spiral\Attributes\Composite\MergeReader([
    new \Spiral\Attributes\AnnotationReader(),
    new \Spiral\Attributes\AttributeReader(),
]);

$metadata = $reader->getClassMetadata(new ReflectionClass(ExampleClass::class));
// returns iterable { DoctrineAnnotation, NativeAttribute }
```

## 缓存

某些实现可能会在某种程度上减慢速度，因为它们会从头开始读取元数据。当有大量此类数据时，尤其如此。

为了优化和加速读取器的工作，建议使用缓存。特性包支持 [PSR-6](https://www.php-fig.org/psr/psr-6/) 和 [PSR-16](https://www.php-fig.org/psr/psr-16/) 规范。要创建它们，你需要使用相应的类。

```php
use Spiral\Attributes\Psr6CachedReader;
use Spiral\Attributes\Psr16CachedReader;
use Spiral\Attributes\AttributeReader;

$psr6reader = new Psr6CachedReader(
    new AttributeReader(),
    new SomePsr6CacheImplementation() // Any PSR-6 cache implementation
);

$psr16reader = new Psr16CachedReader(
    new AttributeReader(),
    new SomePsr6CacheImplementation() // Any PSR-16 cache implementation
);
```

你还可以将缓存实现实例传递给工厂类。

```php
use Spiral\Attributes\Factory;

$reader = (new Factory)
    ->withCache($cacheDriver)
    ->create();

// Where $cacheDriver is PSR-6 or PSR-16 cache driver implementation
```
