# Advanced — Attributes

Attributes in PHP are a way to add metadata to classes, properties, functions, constants, and parameters. This metadata
can be used to provide additional information about the element or to change the behavior of the element in certain
situations.

The `spiral/attributes` component provides a simple and consistent way to work with attributes in PHP.

**Benefits of using attributes in PHP application:**

- They provide a way to add metadata to elements in a way that is separate from the logic of the element.
- They can be used to change the behavior of elements in certain situations, such as providing additional validation for
  a property or changing the behavior of a function based on its attributes.
- They can be used to provide additional information about elements, such as providing documentation for a class or
  property.
- By utilizing the attributes in combination with interceptors, developers can implement Aspect-Oriented Programming
  practices within their codebase, resulting in benefits such as smaller, faster, and more easily testable code.

**The component also serves two very important purposes:**

- The ability to combine different types of metadata in one place. You shouldn't care if the developer is
  using [Doctrine Annotations](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/index.html)
  or [PHP Attributes](https://wiki.php.net/rfc/attributes_v2) added in PHP 8.
- The ability to read attributes in any version of the language. This means that you can use the PHP 8 Attributes right
  now, even if you are using PHP 7.2.

Component provides a metadata reader bridge allowing both modern
[PHP attributes](https://wiki.php.net/rfc/attributes_v2) and
[Doctrine annotations](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/index.html) to be used in
the same project.

> **Note:**
> This documentation uses the term "metadata" to refer to both "attributes" and "annotations".

## Installation

To install the component:

```terminal 
composer require spiral/attributes
```

### Framework Integration

To enable the component, you just need to add the `Spiral\Bootloader\Attributes\AttributesBootloader` class to the
bootloader list:

:::: tabs

::: tab Using method

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

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Attributes\AttributesBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

## Usage

The component provides a set of interfaces and classes for working with attributes.
The `Spiral\Attributes\ReaderInterface` provides a set of methods for reading attributes from classes, properties,
functions, constants, and parameters.

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

### Class Metadata

To read the class metadata, use the `$reader->getClassMetadata()` method. It receives as input the `ReflectionClass` of
the required class and returns a list of available metadata objects.

```php
$reflection = new ReflectionClass(User::class);

$attributes = $reader->getClassMetadata($reflection); 
// returns iterable<object>
```

The second optional argument `$name` of the method allows you to specify which specific metadata objects you want to
retrieve.

```php
$reflection = new ReflectionClass(User::class);

$attributes = $reader->getClassMetadata($reflection, Entity::class); 
// returns iterable<Entity>
```

To get one metadata object, you can use the method `$reader->firstClassMetadata()`.

```php
$reflection = new ReflectionClass(User::class);

$attribute = $reader->firstClassMetadata($reflection, Entity::class); 
// returns Entity|null
```

Since **v2.10.0**, supports read attributes from traits that are used in the class.

:::: tabs

::: tab Attributes

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

::: tab Annotations

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

### Property Metadata

To read the property metadata, use the `$reader->getPropertyMetadata()` method. It receives as input
the `ReflectionProperty` of the required property and returns a list of available metadata objects.

```php
$reflection = new ReflectionProperty(User::class, 'name');

$attributes = $reader->getPropertyMetadata($reflection); 
// returns iterable<object>
```

The second optional argument `$name` of the method allows you to specify which specific metadata objects you want to
retrieve.

```php
$reflection = new ReflectionProperty(User::class, 'name');

$attributes = $reader->getPropertyMetadata($reflection, Column::class); 
// returns iterable<Column>
```

To get one metadata object, you can use the method `$reader->firstPropertyMetadata()`.

```php
$reflection = new ReflectionProperty(User::class, 'name');

$column = $reader->firstPropertyMetadata($reflection, Column::class); 
// returns Column|null
```

### Function Metadata

To read the function metadata, use the `$reader->getFunctionMetadata()` method. It receives an argument of
the `ReflectionFunction` or `ReflectionMethod` type of the required function and returns a list of available metadata
objects.

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$attributes = $reader->getFunctionMetadata($reflection); 
// returns iterable<object>
```

The second optional argument `$name` of the method allows you to specify which specific metadata objects you want to
retrieve.

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$attributes = $reader->getPropertyMetadata($reflection, DTOGetter::class); 
// returns iterable<DTOGetter>
```

To get one metadata object, you can use the method `$reader->firstFunctionMetadata()`.

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$getter = $reader->firstFunctionMetadata($reflection, DTOGetter::class); 
// returns DTOGetter|null
```

### Constant Metadata

To read the class constant metadata, use the `$reader->getConstantMetadata()` method. It receives an argument of
the `ReflectionClassConstant` type of the required function and returns a list of available metadata objects.

```php
$reflection = new ReflectionClassConstant(Example::class, 'CONSTANT_NAME');

$attributes = $reader->getConstantMetadata($reflection); 
// returns iterable<object>
```

The second optional argument `$name` of the method allows you to specify which specific metadata objects you want to
retrieve.

```php
$reflection = new ReflectionClassConstant(Example::class, 'CONSTANT_NAME');

$attributes = $reader->getConstantMetadata($reflection, Deprecated::class); 
// returns iterable<Deprecated>
```

To get one metadata object, you can use the method `$reader->firstConstantMetadata()`.

```php
$reflection = new ReflectionClassConstant(Example::class, 'CONSTANT_NAME');

$getter = $reader->firstConstantMetadata($reflection, Deprecated::class); 
// returns Deprecated|null
```

### Parameter Metadata

To read the function/method parameter metadata, use the `$reader->getParameterMetadata()` method. It receives an
argument of the `ReflectionParameter` type of the required function and returns a list of available metadata objects.

```php
$reflection = new ReflectionParameter('send_email', 'email');

$attributes = $reader->getParameterMetadata($reflection); 
// returns iterable<object>
```

The second optional argument `$name` of the method allows you to specify which specific metadata objects you want to
retrieve.

```php
$reflection = new ReflectionParameter('send_email', 'email');

$attributes = $reader->getParameterMetadata($reflection, PreCondition::class); 
// returns iterable<PreCondition>
```

To get one metadata object, you can use the method `$reader->firstParameterMetadata()`.

```php
$reflection = new ReflectionParameter('send_email', 'email');

$getter = $reader->firstConstantMetadata($reflection, PreCondition::class); 
// returns PreCondition|null
```

## Create Annotations

> **Note**
> For details on using doctrine annotations, please see
>
the [doctrine documentation](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/index.html#create-an-annotation-class).

You should use "hybrid" syntax to create metadata classes that will work on any version of PHP.

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

In this case you can use this metadata class on any PHP version.

:::: tabs

::: tab Attributes

```php
#[MyEntityMetadata(table: 'users')] 
class User {}
```

:::

:::: tab Annotations

```php
/**
 * @MyEntityMetadata(table="users")
 */
class User {}
```

:::

::::

### Instantiation

The package supports different ways of instantiating attributes, but by default it uses Doctrine logic for
compatibility.

Let's say you use your metadata class as follows, passing in one field "`property`" with the string value "`value`".

:::: tabs

::: tab Attributes

```php
#[CustomMetadataClass(property: "value")]
class AnnotatedClass
{
}
```

:::

::: tab Annotations

```php
/** @CustomMetadataClass(property="value") */
class AnnotatedClass
{
}
```

:::

::::

In this case, the annotation class itself might look like this:

#### Doctrine Basic Instantiator

In this case, when declaring a metadata class, the attribute/annotation properties will be filled.

> **Note**
> See
> also [Doctrine Custom Annotations](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/custom.html#custom-annotation-classes)

```php
/** @Annotation */
#[\Attribute]
class CustomMetadataClass
{
    public $property;
}
```

#### Doctrine Constructor Instantiator

In the case of a constructor declaration, all data when using the metadata class will be passed to this constructor as
an array.

> **Note**
> See
> also [Doctrine Custom Annotations](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/custom.html#custom-annotation-classes)

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

#### Named Arguments (Interface Marker)

If you want to use named constructor parameters (see
also [PHP Manual — Named Arguments](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments)),
then you have to add an interface `Spiral\Attributes\NamedArgumentConstructorAttribute` to the metadata class that will
mark the required metadata class as one that takes named arguments.

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

#### Named Arguments (Metadata Marker)

Please note that using the previous method will require you to have a `spiral/attributes` package and you will not be
able to use these classes in other projects where this package is missing.

To solve this problem, you can use the metadata class, which will mean the same thing (the metadata class uses named
arguments), but does not directly implement the interface, and therefore does not require a `spiral/attributes` package
in the project.

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

## Drivers

The `Spiral\Attributes\Factory` encapsulates several implementations behind it and returns
a [selective reader](#selective-reader) implementation by default, which is suitable for most
cases. However, you can require a specific implementation if available on your platform and/or in your application.

```php
use Spiral\Attributes\Factory;

$reader = (new Factory())->create();
```

### Annotation Reader

> **Note**
> Please note that in order for this reader to be available in the application, you need to require
> "doctrine/annotations" component.

This reader implementation allows reading doctrine annotations.

```php
/** @ExampleAnnotation */
class Example {}

$reader = new \Spiral\Attributes\AnnotationReader();

$annotations = $reader->getClassMetadata(new ReflectionClass(Example::class));
// returns iterable<ExampleAnnotation>
```

### Attribute Reader

This reader implementation allows reading native PHP attributes on any PHP version.

```php
#[ExampleAttribute]
class Example {}

$reader = new \Spiral\Attributes\AttributeReader();

$attributes = $reader->getClassMetadata(new ReflectionClass(Example::class));
// returns iterable<ExampleAttribute>
```

### Selective Reader

The implementation automatically selects the correct reader based on the syntax used in the application. This behavior
is required if you use both syntax at the same time in the same project. For example, in the case of an already
working project, which is refactored and the syntax of annotations is translated into the modern syntax of attributes.

:::: tabs

::: tab Attributes

```php
#[ExampleAttribute]
class ClassWithAttributes {}
```

:::

::: tab Annotations

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

> **Note**
> When using both annotations and attributes in the same place, the behavior of this reader is non-deterministic.

### Merge Reader

The reader's implementation combines several syntaxes in one. This behavior is required if you are working with multiple
libraries at the same time that support either only the old or only the new syntax.

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

## Cache

Some implementations can slow things down to some extent because they read the metadata from scratch. This is especially
true when there is a lot of such data.

To optimize and speed up the work of readers, it is recommended to use the cache. The attribute package supports the
[PSR-6](https://www.php-fig.org/psr/psr-6/) and [PSR-16](https://www.php-fig.org/psr/psr-16/) specifications. To create
them, you need to use the corresponding classes.

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

You can also pass a cache implementation instance to the factory class.

```php
use Spiral\Attributes\Factory;

$reader = (new Factory)
    ->withCache($cacheDriver)
    ->create();

// Where $cacheDriver is PSR-6 or PSR-16 cache driver implementation
```
