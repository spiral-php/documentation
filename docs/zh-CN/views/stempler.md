# 视图 — Stempler 模板引擎

Stempler 引擎提供了一个强大且灵活的模板引擎，它能够在词法分析器、解析器和 AST 编译级别进行自定义。 默认情况下，该驱动程序与 Spiral 框架应用程序的 Web 构建版本一起启用，并支持类似 Blade 的指令和回显、HTML 组件、栈等。

## 基本用法

在本节中，我们将引导您完成使用 Stempler 创建和渲染基本视图的步骤。

### 创建视图

第一步是创建视图文件。视图文件应保存在 `app/views` 目录（或在 `ViewsBootloader` 中配置的任何其他目录）中。 Stempler 模板的文件扩展名必须是 `.dark.php`。

让我们创建一个名为 `welcome.dark.php` 的视图文件，其内容如下：

```php app/views/welcome.dark.php
Hello, {{ $name }}!
```

并将其存储在 `app/views` 目录中。

### 渲染视图

现在我们可以从控制器渲染视图。

:::: tabs

::: tab Prototyped
在我们的示例中，我们将使用 `PrototypeTrait` 来简化从容器获取 `ViewsInterface` 实例的操作。

> **了解更多**
> 在 [The Basics — Prototyping](../basics/prototype.md) 部分阅读更多关于原型 trait 的内容。

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): string
    {
        return $this->views->render('welcome', [
            'name' => 'John',
        ]);
    }
}
```

:::

::: tab ViewInterface
您也可以直接从容器访问 `Spiral\Views\ViewsInterface` 实例。

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Views\ViewsInterface;

class HomeController
{
    public function __construct(
        private readonly ViewsInterface $views
    ) {
    }

    public function index(): string
    {
        return $this->views->render('welcome', [
            'name' => 'John',
        ]);
    }
}
```

:::

::::

您应该在屏幕上看到 `Hello, John!`。

Stempler 模板也支持 PHP 底层语法：

```php app/views/welcome.dark.php
Hello, <?= $name ?>!
```

> **危险**
> 重要的是要注意，语法 `{{ $name }}` 提供自动转义，这有助于防止 [XSS 攻击](https://owasp.org/www-community/attacks/xss/) 等安全问题。另一方面，传统的 PHP 语法 `<?= $name ?>` 不提供自动转义。如果您选择使用传统的 PHP 语法，建议手动转义变量以确保应用程序的安全性。

#### 上下文感知转义

转义策略将根据您回显值的位置而改变。您可以在 `script` 标签内回显/嵌入您的值：

```php app/views/welcome.dark.php
<script>
    const value = {{ $name }};
</script>
```

它将根据值的类型进行不同的渲染：

:::: tabs

::: tab String

如果是字符串值 `['name' => 'John']`，该值将自动被引用：

```html
<script>
    const value = "John";
</script>
```

:::

::: tab Number

如果是数字值 `['name' => 123]`：

```html
<script>
    const value = 123;
</script>
```

:::

::: tab Null

如果是 null 值 `['name' => null]`：

```html
<script>
    const value = null;
</script>
```

:::

::: tab List

如果是一个数组 `['name' => ['John']]` 值：

```html
<script>
    const value = ["John"];
</script>
```

:::

::: tab Array

如果是一个关联数组 `['name' => ['first' => 'John', 'last' => 'Doe']]` 值：

```html
<script>
    const value = {"first": "John", "last": "Doe"};
</script>
```

:::

::::

#### 禁用转义

要输出一个值而不进行任何自动转义，您可以使用替代语法。

```php
{!! $value !!}
```

当您想输出 HTML 内容或其他不应转义的内容时，这会很有用。

这里有一个例子：

```php app/src/Endpoint/Web/HomeController.php
public function index(): string
{
    return $this->views->render('welcome', [
        'html' => '<div>Hello world</div>'
    ]);
}
```

视图文件：

```php app/views/welcome.dark.php
{!! $html !!}
```

以及输出：

:::: tabs

::: tab Unescaped
禁用转义后，HTML 内容将按原样输出，没有任何自动转义。

```html
<div>Hello world</div>
```

:::

::: tab Escaped
另一方面，如果使用带有启用自动转义的语法 `{{ $html }}`，HTML 标签将被转义。

```html
&lt;div&gt;Hello world&lt;/div&gt;
```

:::

::::

<hr>

## 指令

除了经典的 echo 结构外，Stempler 还支持许多类似 Blade 的指令，以控制模板的业务逻辑。

与 Blade 或 Twig 不同，Stempler 指令仅负责管理业务逻辑。

> **注意**
> 参见 [组件和属性](#components-and-props) 和 [继承](#inheritance-and-stacks) 以检查如何扩展您的模板并实现虚拟组件。

### 循环指令

Stempler 提供了几个循环指令，以帮助您管理模板中重复元素的渲染。这些指令使将动态内容合并到您的模板中变得容易。

> **注意**
> 指令声明类似于原生 PHP 语法。

#### Foreach

使用指令 `@foreach` 和 `@endforeach` 渲染循环：

```php
<ul>
    @foreach($items as $item)
    <li>{{ $item }}</li>
    @endforeach
</ul>
```

#### For

使用指令 `@for` 和 `@endfor` 渲染循环：

```php
<ul>
    @for($i = 0; $i < 10; $i++)
    <li>{{ $i }}</li>
    @endfor
</ul>
```

#### While

使用指令 `@while` 和 `@endwhile` 渲染 `while` 循环：

```php
<ul>
    @while($i < 10)
    <li>{{ $i }}</li>
    @php $i++; @endphp
    @endwhile
</ul>
```

#### Break 和 Continue

使用 `@break` 和 `@continue` 指令中断您的循环：

```php
<ul>
    @while(true)
    <li>{{ $i }}</li>
    @if($i++ > 10)
        @break
    @endif
    @endwhile
</ul>
```

> **注意**
> `@break(2)` 等同于 `break 2`。在下面阅读更多关于 `if` 指令的内容。

### 条件指令

Stempler 提供了几个用于在模板中创建条件语句的指令。这些指令被转录成原生 PHP 代码，并提供了一种更具可读性和效率的方式来处理模板中的条件。

这些例子给出了以下变量：

```php
return $this->views->render('welcome', [
    'value' => 123
]);
```

#### If 和 Else

要创建简单的条件语句，请使用 `@if` 和 `@endif` 指令。

```php
@if($value === 123)
    Hello World
@endif
```

要添加 `else` 条件，请使用 `@else` 指令。

```php
@if($value !== 123)
    Value is not 123
@else
    
@endif
```

对于更复杂的条件，请使用 `@elseif` 指令。

```php
@if($value === 124)
    Value is not 124
@elseif($value === 123)
    Value is 123
@else
    Another value
@endif
```

#### Unless

`@unless` 指令允许您创建否定条件，并且可以像 `@if` 指令一样与 `@else` 和 `@elseif` 一起使用。

```php
@unless($value === 124)
    Value is not 124
@endunless
```

> **注意**
> 您可以将 `@else` 和 `@elseif` 与 `@unless` 指令一起使用。

#### Empty 和 Isset

分别使用 `@empty` 和 `@isset` 条件来检查变量是否为空或已设置。

:::: tabs

::: tab Empty

```php
@empty($value)
    Value is empty
@endempty
```

:::

::: tab Isset

```php
@isset($value)
    Value is set
@endisset
```

:::

::::

#### Switch case

对于更复杂的条件，您可以使用 `@switch`、`@case` 和 `@break` 语句。

```php
@switch($value)
    @case(123) value is 123 @break
    @case(124) value is 124 @break
    @case(125) value is 125 @break
@endswitch
```

### Json 指令

`@json` 指令允许您在页面内渲染 JSON 数据。要使用它，只需将一个变量传递给该指令，如下所示：

```php
@json($value)
```

> **注意**
> `@json` 指令等同于 `json_encode($value)`。

并设置一个变量：

```php
return $this->views->render('welcome', [
    'value' => ...
]);
```

输出将是：

:::: tabs

::: tab String

如果是字符串值 `['value' => 'Hello world']`：

```html
"Hello world"
```

:::

::: tab Number

如果是数字值 `['value' => 123]`：

```html
123
```

:::

::: tab Null

如果是 null 值 `['value' => null]`：

```html
null
```

:::

::: tab List

如果是一个数组 `['value' => ['John']]` 值：

```html
["John"]
```

:::

::: tab Array
如果是一个关联数组 `['value' => ['first' => 'John', 'last' => 'Doe']]` 值：

```html
{"first":"John","last":"Doe"}
```

:::
::::

#### 嵌入 JSON 数据

将 JSON 数据嵌入到 JavaScript 语句中可能很有用：

这是一个视图模板的示例，其值为 `['value' => ['key' => 'value']]`：

```php
<script type="text/javascript">
    var value = @json($value);
    console.log(value.key);
</script>
```

然后，生成的视图将如下所示：

```php
<script type="text/javascript">
    var value = {"key":"value"};
    console.log(value.key);
</script>
```

### 框架特定指令

Spiral 提供了许多可以在模板中使用的框架特定指令，包括：

#### Container

要将容器依赖项调用到模板中，请使用 `@inject($variable, "class")` 指令：

```php
@inject($app, App\App::class)
{{ get_class($app) }}
```

#### Route

要创建一个路由，请使用指令 `@route`：

```php
<a href="@route('home:index')">click me</a>
```

您可以使用 `controller:action` 模式来处理由 `default route` 或路由名称处理的目标：

```php
$router->addRoute(
    'html',
    new Route('/<action>.html', new Controller(HomeController::class))
);
```

使用第二个参数传递参数：

```php
<a href="@route('html', ['action' => 'index'])">click me</a>
```

这些参数将自动 slugify 到路由 URL 中。 在路由模式中未找到的那些参数将作为查询参数传递：

```php
<a href="@route('html', ['action' => 'index', 'id' => 10])">click me</a>
```

结果 `/index.html?id=10`。

> **了解更多**
> 在 [HTTP — 路由](../http/routing.md) 部分阅读更多关于路由和命名路由的内容。

### Raw PHP

要将 PHP 逻辑嵌入到您的模板中，请使用经典的 `<?php` 和 `?>` 标签，或替代的 `@php` 和 `@endphp`：

```php
@php
    echo "hello world";
@endphp
```

### 转义控制 '@' 字母

只需双写 'at' 字母，例如

```php
@@ // -> 将呈现为 '@'
```

### 自定义指令

Stempler 提供了一种通过自定义指令扩展其功能的方法。 自定义指令是一个扩展 `Spiral\Stempler\Directive\AbstractDirective` 类的类，并实现一个 `render` 方法，该方法接受 `Spiral\Stempler\Node\Dynamic\Directive` 参数。

要创建自定义指令，请按照以下步骤操作：

#### 创建一个指令类

创建一个扩展 `Spiral\Stempler\Directive\AbstractDirective` 并实现 render 方法的类，该方法具有所需的功能。

```php app/src/Integration/Stempler/DatetimeDirective.php
namespace App\Integration\Stempler;

use Spiral\Stempler\Directive\AbstractDirective;
use Spiral\Stempler\Node\Dynamic\Directive;

final class DatetimeDirective extends AbstractDirective
{
    public function renderDateTime(Directive $directive): string
    {
        return '<?php echo date("Y-m-d H:i:s"); ?>';
    }
}
```

> **注意**
> 也可以实现 `Spiral\Stempler\Directive\DirectiveRendererInterface`，以便更低级别地访问渲染过程。

#### 注册指令

使用引导程序类中的 `StemplerBootloader::addDirective()` 方法注册自定义指令。

```php app/src/Application/Bootloader/CustomDirectiveBootloader.php
namespace App\Application\Bootloader;

use App\Integration\Stempler\DatetimeDirective;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

final class CustomDirectiveBootloader extends Bootloader
{
    public function boot(StemplerBootloader $stempler): void
    {
        $stempler->addDirective(DatetimeDirective::class);
    }
}
```

#### 使用指令

自定义指令可以在模板中使用，通过使用适当的语法调用它。

这是模板代码：

```php
<div>
    @dateTime
</div>
```

这是由该指令生成的最终 PHP 代码：

```php
<div>
    <?php echo date("Y-m-d H:i:s"); ?>
</div>
```

通过使用自定义指令，您可以向模板引擎添加自定义功能并在不同的模板中重复使用。

#### 传递值

您可以使用 `Directive` 对象的 `body` 和 `values` 属性将值传递给自定义指令。这些属性可用于访问传递给指令的值。 这允许您将动态值传递给指令，使其更灵活和可重用。

这是一个使用 `body` 属性的示例：

```php app/src/Integration/Stempler/DatetimeDirective.php
public function renderDateTime(Directive $directive): string
{
    return \sprintf('<?php echo date(%s ?? "Y-m-d H:i:s"); ?>', $directive->body);
}
```

例子：

:::: tabs

::: tab String

```php
<div>
    @dateTime('l')
</div>
```

此指令将生成以下 PHP 代码：

```php
<div>
    <?php echo date('l'); ?>
</div>
```

:::

::: tab Variable

```php
@php
$format = 'l';
@endphp
<div>
    @dateTime($format)
</div>
```

此指令将生成以下 PHP 代码：

```php
<?php
$format = 'l';
?>

<div>
   <?php echo date($format); ?>
</div>
```

:::

::: tab Constant

```php
@dateTime(DATE_RFC2822)
```

此指令将生成以下 PHP 代码：

```php
<div>
   <?php echo date(DATE_RFC2822); ?>
</div>
```

:::

::::

要访问传递给指令的由逗号分隔的特定值：

```php app/src/Integration/Stempler/DatetimeDirective.php
public function renderDateTime(Directive $directive): string
{
    return \sprintf(
        '<?php echo date(%s, %s); ?>',
        $directive->values[0], // first value passed to the directive
        $directive->values[1] // second value passed to the directive
    );
}
```

例子：

```php
@php
$format = 'Y-m-d H:i:s';
$timestamp = 199999999;
@endphp

<div>
    @dateTime($format, $timestamp)
</div>
```

此指令将生成以下 PHP 代码：

```php
<?php
$format = 'Y-m-d H:i:s';
$timestamp = 199999999;
?>

<div>
    <?php echo date($format, $timestamp); ?>
</div>
```

> **警告**
> 这些值不会自动转义，因此您必须在使用它们之前手动转义它们。

#### 访问指令上下文

要获取有关指令从何处调用的信息，请使用 `$directive->getContext()->getPath()` 方法：

```php
public function renderDateTime(Directive $directive): string
{
    return \sprintf(
        <<<PHP
<?php echo date(%s, %s); ?>
<!-- invoked from "%s" template -->
PHP,
        $directive->values[0],
        $directive->values[1],
        $directive->getContext()->getPath()
    );
}
```

处理此指令时，它将生成以下 PHP 代码：

```php
<div>
    <?php echo date($format, $timestamp); ?>
    <!-- invoked from "welcome" template -->
</div>
```

## 继承和栈

随着您的视图变得越来越复杂，正确分离页面和布局特定的内容至关重要。 Stempler 提供了几个控制语句来实现这一点。

### 扩展布局

首先，让我们为我们的页面创建一个标准的 HTML 模板：

```html app/views/home.dark.php
<!DOCTYPE html>
<html>
    <head>
        <title>This is homepage.</title>
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </head>
    <body>
        Page content
    </body>
</html>
```

很可能，您的应用程序将包含多个页面模板。为了避免代码重复，Stempler 提供了继承父布局的能力。

> **注意**
> Stempler 将模板和父布局编译为优化的 PHP 代码。您可以排除任意数量的布局，而不会影响性能。

创建一个布局：

```html app/views/layout/base.dark.php
<!DOCTYPE html>
    <html>
    <head>
        <title>This is homepage.</title>
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </head>
    <body>
        Page content
    </body>
</html>
```

现在，我们可以通过 `extends:path` 标签使用 `home.dark.php` 扩展此布局：

```html app/views/home.dark.php
<extends:layout.base/>
```

> **注意**
> 使用分隔符 `.` 将目录名称包含到您的模板中。

或者，使用语法：

```html app/views/home.dark.php
<extends path="layout/base"/>
```

> **注意**
> 您可以在此类声明中使用视图命名空间，例如：`<extends path="default:layout/base"/>`。

#### 替换块

扩展父布局并没有多大意义，除非我们可以重新定义其某些内容。 要定义一个可替换的块，请使用标签 `<block:name>`。 相应地更改 `layout/base.dark.php`：

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title><block:title/></title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body>
<block:content>
    Default content body.
</block:content>
</body>
</html>
```

> **注意**
> 您可以在 `<block:name></block:name>` 标签对内包含默认块内容。

要重新定义块值，请在 `home.dark.php` 模板中使用 `block:name` 或类似的标签：

```html app/views/home.dark.php
<extends:layout.base/>

<block:title>Homepage</block:title>

<block:content>
    This is homepage content.
</block:content>
```

#### 短语法

在您的块定义短字符串或用作标签参数的情况下，使用替代语法 `${name|default}`。 将布局更改为：

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="${body-class|default}">
<block:content>
    Default content.
</block:content>
</body>
</html>
```

短语法值可以通过 `<block:name>value</block:name>` 标签提供给父布局。

```html app/views/home.dark.php
<extends:layout.base/>

<block:title>Homepage</block:title>

<block:body-class>homepage</block:body-class>

<block:content>
    This is homepage content.
</block:content>
```

您可以使用 `extends` 标签属性传递一些块值以避免大型子模板，相应地更改 `app/views/home.dark.php`：

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage"/>

<block:content>
    This is homepage content.
</block:content>
```

在两种情况下，生成的 HTML 都将如下所示：

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage">
This is homepage content.
</body>
</html>
```

#### 调用父内容

要保留父块内容，请在重新定义的块的任何位置使用 `<block:parent/>`：

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage"/>

<block:content>
    This is homepage content.
    <block:parent/>
</block:content>
```

生成的 HTML：

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage">
This is homepage content.
Default content.
</body>
</html>
```

使用 `${parent}`，以在短块定义中实现相同的目标：

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<block:content>
    This is homepage content.
    <block:parent/>
</block:content>
```

输出：

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage default">
This is homepage content.
Default content.
</body>
</html>
```

#### 嵌套布局

可以将布局基于其他布局创建，创建 `app/views/layout/page.dark.php`：

```html app/views/layout/page.dark.php
<extends:layout.base body-class="page ${parent}"/>

<block:content>
    <div class="page-wrapper">
        <block:page/>
    </div>
</block:content>
```

> **注意**
> 扩展标签始终需要完整的路径规范，请确保包含 `layout` 目录。

您可以在 `app/views/home.dark.php` 中扩展此布局而不是 `base`：

```html app/views/home.dark.php
<extends:layout.page title="Homepage" body-class="homepage ${parent}"/>

<block:page>
    Page content.
</block:page>
```

生成的 HTML：

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
</head>
<body class="homepage page default">
<div class="page-wrapper">
    Page content.
</div>
</body>
</html>
```

> **注意**
> 您可以根据需要嵌套任意数量的模板，它只会影响编译速度。

### 栈

Stempler 包含聚合在模板中定义的多个块的能力。

#### 经典方法

您通常需要向您的布局添加自定义 JS 或 CSS 资源。为了实现这一点，请使用 `block` 指令，将必要的资源包装在一个块中，然后在您的子模板中将其追加到它。

修改 `app/views/layout/base.dark.php` 为：

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <block:styles>
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </block:styles>
</head>
<body class="${body-class|default}">
<block:content>
    Default content.
</block:content>
</body>
</html>
```

要在您的页面模板中添加自定义样式资源：

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<block:styles>
    <block:parent/>
    <link rel="stylesheet" href="/styles/homepage.css"/>
</block:styles>

<block:page>
    Page content.
</block:page>
```

生成的 HTML：

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
    <link rel="stylesheet" href="/styles/homepage.css"/>
</head>
<body class="homepage default">
Default content.
</body>
</html>
```

#### 创建栈

为了演示如何使用栈实现以下内容，我们应该从 `app/views/home.dark.php` 中的一个简单示例开始。使用 `<stack:collect name="name"/>` 创建一个栈占位符：

```html app/views/home.dark.php
collect name="my-stack">
    default content
</stack:collect>
```

要将一个值追加到栈中：

```html
<stack:collect name="my-stack">
    default content
</stack:collect>

<stack:push name="my-stack">
    my value
</stack:push>
```

结果的 HTML：

```html
  default content
my value
```

要将一个值预先添加到栈中：

```html
<stack:collect name="my-stack">
    default content
</stack:collect>

<stack:prepend name="my-stack">
    my value
</stack:prepend>
```

输出：

```html
  my value
default content
```

您可以在 `push` 和 `prepend` 标签之前或之后定位栈定义：

```html
<stack:prepend name="my-stack">
    my value
</stack:prepend>

<stack:collect name="my-stack">
    default content
</stack:collect>
```

#### 深层栈

如果它位于同一标签树级别上，则栈标签将仅聚合 `push` 和 `prepend` 值。

例如，这将起作用：

```html
<stack:collect name="my-stack">
    default content
</stack:collect>

// stack my-stack is active here
<div>
    // and here
    <stack:prepend name="my-stack">
        my value
    </stack:prepend>
</div>
```

而此示例将不起作用：

```html
<div>
    // stack my-stack is active here
    <stack:collect name="my-stack">
        default content
    </stack:collect>
    // and here
</div>

// stack my-stack is not active at this level

<stack:prepend name="my-stack">
    my value
</stack:prepend>
```

> **注意**
> 此限制是由栈收集器的 AST 本质引起的。

要在不将占位符级别提高的情况下绕过此限制，请使用 `stack:collect` 属性 `level`：

```html
<div>
    <stack:collect name="my-stack" level="1">
        default content
    </stack:collect>
</div>

<stack:prepend name="my-stack">
    my value
</stack:prepend>
```

属性 `level` 将栈配置为多个激活级别更高。 例如，此示例**将不起作用**：

```html
<div>
    // stack my-stack is active here
    <div>
        // stack my-stack is active here
        <stack:collect name="my-stack" level="1">
            default content
        </stack:collect>
    </div>
</div>

// stack my-stack is no active at this level

<stack:prepend name="my-stack">
    my value
</stack:prepend>
```

但是这个会起作用：

```html
<div>
    // stack my-stack is active here
    <div>
        // stack my-stack is active here
        <stack:collect name="my-stack" level="2">
            default content
        </stack:collect>
    </div>
</div>

// stack my-stack is active here

<stack:prepend name="my-stack">
    my value
</stack:prepend>
```

#### 布局中的栈

您可以将值推送到在父布局中定义的栈中。 相应地修改 `app/views/layout/base.dark.php`：

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <stack:collect name="styles" level="2">
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </stack:collect>
    <block:resources/>
</head>
<body class="${body-class|default}">
<block:content>
    Default content.
</block:content>
</body>
</html>
```

现在您可以从 `app/views/home.dark.php` 推送该值：

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<block:resources>
    <stack:push name="styles">
        <link rel="stylesheet" href="/styles/homepage.css"/>
    </stack:push>
</block:resources>

<block:page>
    Page content.
</block:page>
```

> **注意**
> 您必须确保 `stack:push` 位于其中一个扩展块中。 参见下面如何绕过它。

### 上下文和隐藏内容

正如您在前面的示例中看到的那样，同时使用栈和块并不方便。这是因为栈收集发生在父布局扩展之后。将栈保留在任何 `block` 之外将使其排除在模板之外。

所有在子模板中定义的位于 `block` 标签之外的 stempler 块都将出现在系统块 `context` 中。 我们可以像这样修改父布局 `app/views/layout/base.dark.php`：

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <stack:collect name="styles" level="2">
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </stack:collect>
</head>
<body class="${body-class|default}">
<block:content>
    Default content.
</block:content>
</body>
<block:context/>
</html>
```

现在我们可以在 `app/views/home.dark.php` 中定义栈，如下所示：

```html app/views/home.dark.php
<extends:layout.base title="Homepage" body-class="homepage ${parent}"/>

<stack:push name="styles">
    <link rel="stylesheet" href="/styles/homepage.css"/>
</stack:push>

some random string

<block:page>
    Page content.
</block:page>
```

要了解上下文的工作原理，请查看生成的 HTML：

```html
<!DOCTYPE html>
<html>
<head>
    <title>Homepage</title>
    <link rel="stylesheet" href="/styles/welcome.css"/>
    <link rel="stylesheet" href="/styles/homepage.css"/>
</head>
<body class="homepage default">
Default content.
</body>
some random string
</html>
```

请注意，`some random string` 被添加而不是 `block:context`，此内容由 `app/views/home.dark.php` 声明。您最有可能将模板的块定义之间的区域用于注释和其他控制指令。
要从最终使用中隐藏此类内容，请在 `app/views/layout/base.dark.php` 中使用 `<hidden></hidden>` 标签：

```html app/views/layout/base.dark.php
<!DOCTYPE html>
<html>
<head>
    <title>${title|Default title}</title>
    <stack:collect name="styles" level="2">
        <link rel="stylesheet" href="/styles/welcome.css"/>
    </stack:collect>
</head>
<body class="${body-class|default}">
<block:content>
    Default content.
</block:content>
</body>
<hidden>
    <block:context/>
</hidden>
</html>
```

现在，堆叠将像以前一样工作。 但是，`some random string` 不会出现在页面上。

> **注意**
> 将栈与继承和 [组件](#components-and-props) 结合使用，以创建特定于领域的渲染 DSL。

## 组件和属性

Stempler 提供了创建开发人员驱动的模板组件作为虚拟标签的能力。

### 简单组件

在许多情况下，您的模板不仅会重用父布局，还会重用模板部分，例如：

```html app/views/home.dark.php
<extends:layout.base title="Homepage"/>

<block:content>
    This is the homepage.

    <div class="article">
        <div class="title">Article title</div>
        <div class="preview">article preview</div>
    </div>

    <div class="article">
        <div class="title">Article title 2</div>
        <div class="preview">article preview 2</div>
    </div>

    <div class="article">
        <div class="title">Article title 3</div>
        <div class="preview">article preview 3</div>
    </div>
</block:content>
```

我们可以将 article div 移到单独的模板 `app/views/partial/article.dark.php` 中：

```html app