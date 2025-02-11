# 组件 — 数据网格

利用 Spiral 的强大功能，`spiral/data-grid` 组件为开发人员提供了一个简化的解决方案，可以根据用户规范自动生成 Cycle 和 DBAL 的 `SELECT` 查询。

> **警告：**
> Spiral 默认不提供 Cycle ORM。要使用此组件，你需要安装 `spiral/cycle-bridge` 组件。你可以在 [基础知识 — 数据库和 ORM](../basics/orm.md) 章节中找到关于 Cycle ORM Bridge 的更多信息。

#### 什么时候应该考虑使用数据网格组件？

1.  **RESTful API 开发：** 想象一下，你拥有构建动态 API 的能力，这些 API 可以根据用户请求过滤、排序和管理关系数据，而无需编写繁琐且重复的查询代码。

2.  **数据可视化平台：** 如果你的应用程序需要以表格格式呈现庞大的数据集，并具有按列排序、过滤记录和分页结果等功能，那么数据网格组件是你以最小的 effort 就能实现这一目标的门票。

3.  **内容管理系统 (CMS)：** 对于经常根据用户角色、偏好或搜索条件提取和显示数据的平台，此组件可以提高效率，使数据检索快速而直接。

4.  **电子商务平台：** 想象一下用户按类别过滤产品、按价格排序或分页浏览数千种商品的情况。数据网格组件可以使这些操作流畅且用户友好。

**通过利用数据网格组件，开发人员可以在应用程序层之间实现清晰的界限：**

-   **自动查询处理：** 无需用详细的查询指令来干扰你的界面层，只需定义用户驱动的规范（例如过滤或排序），然后让组件自动构建必要的特定于域的查询。

-   **统一的数据表示：** 一旦域层提取或处理数据，该组件就可以标准化数据的呈现方式，无论其来源或复杂性如何。

-   **解耦的输入源：** 该组件处理多个输入源的通用性确保了域层对请求的来源保持不可知，无论是 HTTP、控制台还是 gRPC。

-   **可扩展的网格模式：** 根据你的域结构设计网格模式，并让界面层根据用户输入动态调整它们，从而保持数据结构和呈现逻辑之间的明确界限。

## 安装

要安装该组件：

```terminal
composer require spiral/data-grid-bridge spiral/cycle-bridge
```

在 Cycle bootloaders 之后，在你的应用程序中激活 bootloader `Spiral\DataGrid\Bootloader\GridBootloader`：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\DataGrid\Bootloader\GridBootloader::class,
        // ...
    ];
}
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 章节中阅读更多关于 bootloaders 的内容。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\DataGrid\Bootloader\GridBootloader::class,
    // ...
];
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 章节中阅读更多关于 bootloaders 的内容。
:::

::::

## 快速开始

安装后，在 `app/config/dataGrid.php` 配置文件中为你的数据源设置 writers。

**以下是 Cycle ORM Bridge 的基本设置：**

```php app/config/dataGrid.php
return [
    'writers' => [
        \Spiral\Cycle\DataGrid\Writer\QueryWriter::class,
        \Spiral\Cycle\DataGrid\Writer\PostgresQueryWriter::class,
        \Spiral\Cycle\DataGrid\Writer\BetweenWriter::class,
    ],
];
```

> **注意**
> 正如你所看到的，我们可以为不同的规范注册多个 writers。

## 使用

该组件提供了两个基本的抽象 — **网格工厂（Grid Factory）** 和 **网格模式（Grid Schema）**。

### 网格模式（Grid Schema）

将模式（Schema）视为一组规则，用于根据用户的请求获取数据的方式。

**以下是网格模式的简单示例：**

```php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\StringValue;

$schema = new GridSchema();

// 用户分页：将结果限制为每页 10 个
$schema->setPaginator(new PagePaginator(10));

// 排序选项：按 id 排序
$schema->addSorter('id', new Sorter('id'));

// 过滤选项：按名称查找匹配用户输入
$schema->addFilter('name', new Like('name', new StringValue()));
```

简而言之，通过模式（Schema），开发人员可以自定义在获取数据时的过滤和排序选项。

为了更简洁的设置，你可以扩展 `GridSchema` 并在其构造函数中设置所有内容，如下例所示：

```php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Value\StringValue;

class UserSchema extends GridSchema
{
    public function __construct()
    {
        // 用户分页：将结果限制为每页 10 个
        $this->setPaginator(new PagePaginator(10));

        // 排序选项：按 id 排序
        $this->addSorter('id', new Sorter('id'));

        // 过滤选项：按名称查找匹配用户输入
        $this->addFilter('name', new Like('name', new StringValue()));
    }
}
```

### 网格工厂（Grid Factory）

网格工厂（Grid Factory）是你的网格模式（grid schema）和你想检索的实际数据之间的链接。以下代码示例演示了如何使用 Cycle ORM 仓库（Repository）将模式（schema）连接到数据：

```php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\GridFactoryInterface;

$schema = new UserSchema();

$factory = $container->get(GridFactoryInterface::class);
$users = $container->get(\App\Database\UserRepository::class);

/** @var Spiral\DataGrid\GridInterface $result */
$result = $factory->create($users->select(), $schema);

// 获取精炼的数据
print_r(iterator_to_array($result));
```

你也可以设置默认的规范：

```php
/** @var Spiral\DataGrid\GridFactory $factory */
$factory = $factory->withDefaults([
    GridFactory::KEY_SORT     => ['id' => 'desc'],
    GridFactory::KEY_FILTER   => ['name' => 'Antony'],
    GridFactory::KEY_PAGINATE => ['page' => 3, 'limit' => 100]
]);
```

> **注意**
> 因为 `withDefaults` 方法是不可变的，调用它不会更改原始的 Grid Factory。相反，它会给你一个带有指定默认值的新实例。

如何应用这些规范：

-   要从第二页选择用户，请使用 `POST` 或 `QUERY` 数据打开页面，如：`?paginate[page]=2`
-   要激活 `like` 过滤器：`?filter[name]=antony`
-   按 `ASC` 或 `DESC` 排序：`?sort[id]=desc`
-   获取总值的计数：`?fetchCount=1`

最后，最后一个代码示例展示了一个示例控制器在将所有内容放在一起时可能是什么样子：

```php app/src/Endpoint/Web/Controller/UserController.php
use Spiral\DataGrid\GridInterface;
use Spiral\DataGrid\GridFactoryInterface;
use App\Database\UserRepository;

class UserController
{
    #[Route('/users')]
    public function index(UserSchema $schema, GridFactoryInterface $factory, UserRepository $users): array
    {
        /** @var GridInterface $result */
        $result = $factory->create($users->select(), $schema);

        $values = [];

        foreach ([
            GridInterface::FILTERS,
            GridInterface::SORTERS,
            GridInterface::COUNT,
            GridInterface::PAGINATOR
        ] as $key) {
             $values[$key] = $result->getOption($key);
        }

        return [
            'users' => iterator_to_array($result),
            'grid' => [
                'values' => $values,
            ],
        ];
    }
}
```

## 输入源

Grid Factory 从 `Spiral\DataGrid\InputInterface` 获取其数据。默认情况下，它通过 `Spiral\Http\Request\InputManager` 从 HTTP 请求中获取此数据。

然而，该组件的优点在于其灵活性。它不仅适用于 HTTP 请求。你可以使其与命令行提示符、gRPC 请求或其他来源一起使用。只需为你的所选数据源使用 `InputInterface`，并为你的应用程序的需求设置 DataGrids。

**有两种主要方法可以切换数据的来源：**

1.  使用 `GridFactory::withInput` 进行本地更改： 适用于临时更改，例如在测试期间。这是一个使用数组作为输入的示例：

```php
use Spiral\DataGrid\Input\ArrayInput;

/** @var Spiral\DataGrid\GridFactory $factory */
$factory = $factory->withInput(new ArrayInput([
    'name' => 'antony',
    'id' => 'desc'
]));

/** @var Spiral\DataGrid\GridInterface $result */
$result = $factory->create($users->select(), $schema);
```

> **注意**
> 因为 `withInput` 方法是不可变的，调用它不会更改原始的 Grid Factory。相反，它会给你一个带有指定输入的新实例。

2.  **通过容器设置全局输入源：** 在这里，你正在更改整个应用程序的输入源。

为了说明，这就是你修改 Grid Factory 以从控制台获取输入的方式：

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Boot\Bootloader\Bootloader;

class AppBootloader extends Bootloader
{
    protected const SINGLETONS = [
        \Spiral\DataGrid\InputInterface::class => ConsoleInput::class,
    ];
}
```

## 网格 writers

在 Data Grid 中，我们有 **Schema** 来概述应该如何管理数据，以及 **Factory** 来定义这些数据的来源。 除此之外，我们还有 **Writers**。

他们的工作？根据用户输入更改数据。

**可以这样理解他们：**

如果你有一本书中的数据，并使用铅笔（writer）添加、修改或擦除内容，那么这支铅笔就是网格 writer。Spiral 具有用于 Cycle ORM 的 writer。但很酷的是，你可以为其他系统制作你自己的铅笔，比如 Doctrine 集合。

### 如何创建你自己的 Writer

想制作你自己的铅笔（或 writer）吗？请按照以下步骤操作：

1.  使用 `Spiral\DataGrid\WriterInterface` 作为你的指南。

**这是一个例子：**

```php app/src/Application/Schema/DoctrineCollectionWriter.php
<?php

declare(strict_types=1);

namespace App\Application\Schema;

use Doctrine\Common\Collections\Collection;
use Doctrine\Common\Collections\Criteria;
use Doctrine\Common\Collections\Expr\Comparison;
use Spiral\DataGrid\Compiler;
use Spiral\DataGrid\Specification\Filter\Equals;
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Sorter\AbstractSorter;
use Spiral\DataGrid\SpecificationInterface;
use Spiral\DataGrid\WriterInterface;

final class DoctrineCollectionWriter implements WriterInterface
{
    public function write(mixed $source, SpecificationInterface $specification, Compiler $compiler): mixed
    {
        // 如果数据不在集合中，则原样返回。
        if (!$source instanceof Collection) {
            return $source;
        }

        // 准备一组用于更改数据的规则。
        $criteria = null;

        // 如果更改与排序有关。
        if ($specification instanceof AbstractSorter) {
            $orders = [];
            foreach ($specification->getExpressions() as $field) {
                $orders[$field] = ($specification->getValue() === AbstractSorter::ASC)
                    ? Criteria::ASC
                    : Criteria::DESC;
            }

            if ($orders !== []) {
                $criteria = (new Criteria())->orderBy($orders);
            }
        // 如果更改与匹配确切值有关。
        } elseif ($specification instanceof Equals) {
            $expr = new Comparison($specification->getExpression(), Comparison::EQ, $specification->getValue());
            $criteria = (new Criteria())->where($expr);
        // 如果更改与检查数据是否包含特定文本有关。
        } elseif ($specification instanceof Like) {
            $criteria = new Criteria(
                Criteria::expr()->contains($specification->getExpression(), $specification->getValue())
            );
        } else {
            // ...等等。
            return null;
        }

        // 如果有任何更改，则应用更改。
        if ($criteria !== null) {
            $source = $source->matching($criteria)->getValues();
        }

        return $source;
    }
}
```

如果 writer 返回 `null`，编译器将认为 writer 不知道如何处理规范，并且如果所有 writers 都返回 `null`，编译器将抛出异常 `Spiral\DataGrid\Exception\CompilerException`。本质上，这是系统说“嘿，这里有些不对劲。没有任何 writer 完成他们的工作”的方式。

**为什么它很重要**

想象一下，你正尝试更新数据库中的记录。你已经为系统提供了一组关于如何进行此更新的规则。你希望出现两种结果之一：要么成功更新了记录，要么更新参数出现问题。

`CompilerException` 充当反馈机制。Spiral 不会在静默失败并让开发人员挠头的情况下，明确地提醒他们没有任何 writers 做出任何更改。这种反馈对于调试和确保数据完整性非常宝贵。

2.  在 `app/config/dataGrid.php` 配置文件中注册你的 writer。

我们需要在 `writers` 部分注册 writer。writers 的顺序很重要，因为编译器将按照它们注册的相同顺序使用它们。

```php app/config/dataGrid.php
return [
    'writers' => [
        \App\Application\Schema\DoctrineCollectionWriter::class,
    ],
];
```

就这样！现在你可以尝试将 Doctrine 集合传递给网格工厂，看看它如何工作。

## 计算条目

如果你需要使用复杂的函数来计算条目，你可以通过 `withCounter` 方法传递一个可调用的函数：

```php
/** @var Spiral\DataGrid\GridFactory $factory */
$factory = $factory->withCounter(static function ($select): int {
    return count($select) * 2;
});
```

> **注意**
> 这是一个简单的示例，但此函数可能在处理带有 joins 的复杂 SQL 请求时非常有用。

## 分页规范

### 页面分页器规范

这是一个简单的页码+限制分页：

```php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;

$schema = new GridSchema();
$schema->setPaginator(new PagePaginator(10, [25, 50, 100, 500]));
// ...
```

从用户输入来看，这种分页器接受一个包含两个键的数组，`limit` 和 `page`。
如果设置了限制，则应将其显示在 `allowedLimits` 构造函数参数中。

```php
use Spiral\DataGrid\Specification\Pagination\PagePagination;

$paginator = new PagePaginator(10, [25, 50, 100, 500]);

$paginator->withValue(['limit' => 123]); // 不会应用
$paginator->withValue(['limit' => 50]);  // 将应用
$paginator->withValue(['limit' => 100]); // 将应用

$paginator->withValue(['limit' => 100, 'page' => 2]);
```

在后台，该分页器将 `limit` 和 `page` 转换为 `Limit` 和 `Offset` 规范。你可以自由编写自己的分页器，例如基于游标的分页器（例如：`lastID`+`limit`）。

## 排序器规范

排序器是携带排序方向的规范。对于可以应用方向的排序器，你可以传递以下值之一：

-   `1`，`'1'`，`'asc'`，`SORT_ASC` 用于升序
-   `-1`，`'-1'`，`'desc'`，`SORT_DESC` 用于降序

现在，以下规范可用于网格：

*   [有序排序器](#sorter-specifications-ordered-sorters)
*   [方向性排序器](#sorter-specifications-directional-sorter)
*   [排序器](#sorter-specifications-sorter)
*   [排序器集](#sorter-specifications-sorter-set)

### 有序排序器

`AscSorter` 和 `DescSorter` 包含应该以升序（或降序）排序顺序应用的表达式：

```php
use Spiral\DataGrid\Specification\Sorter;

$ascSorter = new Sorter\AscSorter('first_name', 'last_name');
$descSorter = new Sorter\DescSorter('first_name', 'last_name');
```

### 方向性排序器

此排序器包含 2 个独立的排序器，每个用于升序和降序。通过接收 `withValue` 的顺序，我们将获得其中一个排序器：

```php
use Spiral\DataGrid\Specification\Sorter;

$sorter = new Sorter\DirectionalSorter(
    new Sorter\AscSorter('first_name'),
    new Sorter\DescSorter('last_name')
);

// 将按 first_name asc 排序
$ascSorter = $sorter->withDirection('asc');

// 将按 last_name desc 排序
$descSorter = $sorter->withDirection('desc');
```

> **注意**
> 你可以在两个排序器中使用不同的字段集进行排序。
> 如果你具有相同的字段集，请改用 [排序器](#sorter-specifications-sorter-specification)。

### 排序器

如果你的两个方向的排序字段相同，则这是方向性排序器的排序器包装器：

```php
use Spiral\DataGrid\Specification\Sorter;

$sorter = new Sorter\Sorter('first_name', 'last_name');

// 将按 first_name 和 last_name asc 排序
$ascSorter = $sorter->withDirection('asc');

// 将按 first_name 和 last_name desc 排序
$descSorter = $sorter->withDirection('desc');
```

### 排序器集

这只是一种将排序器组合成一个集合的方式，传递方向将应用于整个集合：

```php
use Spiral\DataGrid\Specification\Sorter;

$sorter = new Sorter\SorterSet(
    new Sorter\AscSorter('first_name'),
    new Sorter\DescSorter('last_name'),
    new Sorter\Sorter('email', 'username')
    // ...
);

// 将按 first_name、email 和 username asc 排序，也按 last_name desc 排序
$ascSorter = $sorter->withDirection('asc');

// 将按 last_name、email 和 username desc 排序，也按 first_name asc 排序
$descSorter = $sorter->withDirection('desc');
```

## 过滤规范

过滤器是携带值的规范。值可以直接通过构造函数传递。在这种情况下，过滤器值是固定的，将按原样应用。

```php
use Spiral\DataGrid\Specification\Filter;

// name 应该为 'Antony'
$filter = new Filter\Equals('name', 'Antony');

// name 仍然是 'Antony'
$filter = $filter->withValue('John');
```

如果将 `ValueInterface` 传递给构造函数，你可以使用 `withValue()` 方法。然后，它将检查传入的值是否与 `ValueInterface` 类型匹配，并将进行转换。

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// price 尚未定义
$filter = new Filter\Equals('price', new Value\NumericValue());

// 该值将转换为 int，并且 price 应该等于 7
$filter = $filter->withValue('7');

// 此值不适用，因为它不是数字
$filter = $filter->withValue([123]);
```

现在，以下规范可用于网格：

*   [全部](#filter-specifications-all)
*   [任何](#filter-specifications-any)
*   [(不)等于](#filter-specifications-not-equals)
*   [比较 gt/gte lt/lte](#filter-specifications-compare)
*   [(不)在数组中](#filter-specifications-not-in-array)
*   [like](#filter-specifications-like)
*   [map](#filter-specifications-map)
*   [select](#filter-specifications-select)
*   [between](#filter-specifications-between)

> **注意**
> [过滤器值](#filter-values) 和 [值访问器](#value-accessors) 部分中还有更多有趣的内容。

### 全部

这是用于逻辑 `and` 运算的联合过滤器。

一些具有固定值的示例：

```php
use Spiral\DataGrid\Specification\Filter;

// price 应该等于 2 并且 quantity 应该大于 5
$all = new Filter\All(
    new Filter\Equals('price', 2),
    new Filter\Gt('quantity', 5)
);
```

传递的值将应用于所有子过滤器：

使用 `ValueInterface` 的示例：

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

$all = new Filter\All(
    new Filter\Equals('price', new Value\NumericValue()),
    new Filter\Gt('quantity', new Value\IntValue()),
    new Filter\Lt('option_id', 4)
);

// price 应该等于 5，quantity 应该大于 5，并且 option_id 小于 4
$all = $all->withValue(5);
```

### 任何

这是用于逻辑 `or` 运算的联合过滤器。

具有固定值的示例：

```php
use Spiral\DataGrid\Specification\Filter;

// price 应该等于 2 或 quantity 大于 5
$any = new Filter\Any(
    new Filter\Equals('price', 2),
    new Filter\Gt('quantity', 5)
);
```

传递的值将应用于所有子过滤器。

使用 `ValueInterface` 的示例：

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

$any = new Filter\Any(
    new Filter\Equals('price', new Value\NumericValue()),
    new Filter\Gt('quantity', new Value\IntValue()),
    new Filter\Lt('option_id', 4)
);

// price 应该等于 5，或者 quantity 应该大于 5，或者 option_id 小于 4
$any = $any->withValue(5);
```

### (不)等于

这些是用于逻辑 `=`、`!=` 运算的简单表达式过滤器。

带有固定值的示例：

```php
use Spiral\DataGrid\Specification\Filter;

$equals = new Filter\Equals('price', 2);       // price 应该等于 2
$notEquals = new Filter\NotEquals('price', 2); // price 不应该等于 2
```

使用 `ValueInterface` 的示例：

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// price 应该等于 2
$equals = new Filter\Equals('price', new Value\NumericValue());
$equals = $equals->withValue('2');

// price 不应该等于 2
$notEquals = new Filter\NotEquals('price', new Value\NumericValue());
$notEquals = $notEquals->withValue('2');
```

### 比较

这些是用于逻辑 `>`、`>=`、`<`、`<=` 运算的简单表达式过滤器。

带有固定值的示例：

```php
use Spiral\DataGrid\Specification\Filter;

$gt = new Filter\Gt('price', 2);   // price 应该大于 2
$gte = new Filter\Gte('price', 2); // price 应该大于 2 或等于 2
$lt = new Filter\Lt('price', 2);   // price 应该小于 2
$lte = new Filter\Lte('price', 2); // price 应该小于 2 或等于 2
```

使用 `ValueInterface` 的示例：

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// price 应该大于 2
$gt = new Filter\Gt('price', new Value\NumericValue());
$gt = $gt->withValue('2');

// price 应该大于 2 或等于 2
$gte = new Filter\Gte('price', new Value\NumericValue());
$gte = $gte->withValue('2');

// price 应该小于 2
$lt = new Filter\Lt('price', new Value\NumericValue());
$lt = $lt->withValue('2');

// price 应该小于 2 或等于 2
$lte = new Filter\Lte('price', new Value\NumericValue());
$lte = $lte->withValue('2');
```

### (不)在数组中

这些是用于逻辑 `in`、`not in` 运算的简单表达式过滤器。

带有固定值的示例：

```php
use Spiral\DataGrid\Specification\Filter;

$inArray = new Filter\InArray('price', [2, 5]);       // price 应该在数组 2 和 5 中
$notInArray = new Filter\NotInArray('price', [2, 5]); // price 不应该在数组 2 和 5 中
```

使用 `ValueInterface` 的示例：

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// price 应该在数组 2 和 5 中
$inArray = new Filter\InArray('price', new Value\NumericValue());
$inArray = $inArray->withValue(['2', '5']);

// price 不应该在数组 2 和 5 中
$notInArray = new Filter\NotInArray('price', new Value\NumericValue());
$notInArray = $notInArray->withValue(['2', '5']);
```

第三个参数允许自动使用 `ArrayValue` 包装 `ValueInterface`（默认启用）。如果你有一个非平凡值（或用访问器值包装），请传递 `false` 作为第三个参数来控制过滤器包装：

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;
use Spiral\DataGrid\Specification\Value\Accessor\Split;
use Spiral\DataGrid\SpecificationInterface;

$inArray = new Filter\InArray('field', new Split(new Value\ArrayValue(new Value\IntValue()), '|'), false);
$inArray->withValue('1|2|3')->getValue(); // [1, 2, 3]
```

### Like

这是用于 `like` 运算的简单表达式过滤器。

带有固定值的示例：

```php
use Spiral\DataGrid\Specification\Filter;

$likeFull = new Filter\Like('name', 'Tony', '%%%s%%'); // name 应该类似于 '%Tony%'
$likeEnding = new Filter\Like('name', 'Tony', '%s%%'); // name 应该类似于 'Tony%'
```

使用 `ValueInterface` 的示例：

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// name 应该类似于 '%Tony%'
$like = new Filter\Like('name', new Value\StringValue());
$like = $like->withValue('Tony');
```

### Map

Map 是一个复杂的过滤器，它表示带有其自身值的过滤器映射。

```php
use Spiral\DataGrid\Specification\Filter;

// price 应该大于 2，并且 quantity 应该小于 5
$map = new Filter\Map([
    'from' => new Filter\Gt('price', 2),
    'to'   => new Filter\Lt('quantity', 5)
]);
```

传递的值将应用于所有子过滤器，所有值都是必需的：

使用 `ValueInterface` 的示例：

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

$map = new Filter\Map([
    'from' => new Filter\Gt('price', new Value\NumericValue()),
    'to'   => new Filter\Lt('quantity', new Value\NumericValue())
]);

// price 应该大于 2，并且 quantity 应该小于 5
$map = $map->withValue(['from' => 2, 'to' => 5]);

// 输入无效，map 将设置为 null
$map = $map->withValue(['to' => 5]);
```

### Select

此规范表示一组可用的表达式。
从输入传递一个值将从此集合中选择单个或多个规范。

> **注意**
> 你只需要传递一个键或一个键的数组。请注意，不应该声明任何 `ValueInterface`。

带有单个值的示例：

```php
use Spiral\DataGrid\Specification\Filter;

// 注意，这里我们有整数键
$select = new Filter\Select([
    new Filter\Equals('name', 'value'),
    new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    new Filter\Equals('email', 'email@example.com'),
]);

// 选择第二个过滤器，将等于 'any' 规范。
$filter = $select->withValue(1);
```

带有多个值的示例：

```php
use Spiral\DataGrid\Specification\Filter;

$select = new Filter\Select([
    'one'  => new Filter\Equals('name', 'value'),
    'two'  => new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    'three' => new Filter\Equals('email', 'email@example.com'),
]);

// 过滤器将包含包装在 'all' 规范中的两个子过滤器
$filter = $select->withValue(['one', 'two']);
```

带有未知值的示例：

```php
use Spiral\DataGrid\Specification\Filter;

$select = new Filter\Select([
    'one'  => new Filter\Equals('name', 'value'),
    'two'  => new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    'three' => new Filter\Equals('email', 'email@example.com'),
]);

// 过滤器将等于 null
$filter = $select->withValue('four');
```

### Between

此过滤器表示 SQL `between` 操作，但可以表示为两个 `gt/gte` 和 `lt/lte` 过滤器。

你可以定义是否应包含边界值。如果未包含边界值，则此过滤器将被转换为 `gt`+`lt` 过滤器，否则，在通过 `getFilters()` 方法获取过滤器时，你可以指定使用原始的 `between` 运算符还是 `gte`+`lte` 过滤器。

> **注意**
> 并非所有数据库都支持 `between` 操作，这就是为什么默认情况下使用转换为 `gt/gte`+`lt/lte` 的原因。

Between 过滤器有两种修改：基于字段和基于值。

```php
use Spiral\DataGrid\Specification\Filter;

$fieldBetween  = new Filter\Between('field', [10, 20]);
$valueBetween  = new Filter\ValueBetween('2020 Apr, 10th', ['start_date', 'end_date']);
```

上面的示例类似于以下 SQL 查询：

```sql
#
基于字段
select *
from table_name
where field between 10 and 20;
#
或使用 gte/lte 转换
select *
from table_name
where field >= 10
  and field <= 20;

#
基于值
select *
from table_name
where '2020 Apr, 10th' between start_date and end_date;
#
或使用 gte/lte 转换
select *
from table_name
where start_date <= '2020 Apr, 10th'
  and end_date >= '2020 Apr, 10th';
```

使用 `ValueInterface` 的示例：

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// price 应该在 10 和 20 之间
$fieldBetween  = new Filter\Between('price', new Value\NumericValue());
$fieldBetween = $fieldBetween->withValue([10, '20']);

// '2020 Apr, 10th' 应该在 start_date 和 end_date 之间
$valueBetween  = new Filter\ValueBetween(new Value\DatetimeValue(), ['start_date', 'end_date']);
$valueBetween = $valueBetween->withValue('2020 Apr, 10th');
```

选择渲染类型：

```php
use Spiral\DataGrid\Specification\Filter;

$between  = new Filter\Between('price', [10, 20]);

$between->getFilters();     // 将转换为 gte+lte
$between->getFilters(true); // 将按原样呈现

$notIncludingBetween  = new Filter\Between('price', [10, 20], false, false);

// 无论如何都将转换为 gte+lte
$notIncludingBetween->getFilters();
$notIncludingBetween->getFilters(true);
```

> **注意**
> `ValueBetween` 过滤器也是如此。

## 混合规范

`Spiral\DataGrid\Specification\Filter\SortedFilter` 和 `Spiral\DataGrid\Specification\Sorter\FilteredSorter` 是特殊的序列规范，允许在单个名称下同时使用过滤器和排序器。

用法：

```php

$schema->addFilter(
    'filter',
    new Filter\Select(
        [
            'upcoming'      => new Sorter\SortedFilter(
                'upcoming',
                new Filter\Gt('date', new DateTimeImmutable('now')),
                new Sorter\AscSorter('date')
            ),
            'mostReviewed'  => new Sorter\SortedFilter(
                'mostReviewed',
                new Filter\Lte('date', new DateTimeImmutable('now')),
                new Sorter\DescSorter('count_reviews')
            )
        ]
    )
);
```

> **注意**
> 我们使用 `upcoming` 过滤器同时应用排序和过滤。

## 过滤器值

我们使用过滤器值来转换输入类型及其验证。请勿在未通过 `accepts()` 方法验证输入的情况下使用 `convert()` 方法。它们可以告诉你输入是否可接受，并将其转换为所需的类型。现在，以下值可用于网格：

