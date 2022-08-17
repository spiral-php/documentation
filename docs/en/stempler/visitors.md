# Stempler - AST Modifications

Stempler template engine fully exposes template AST (DOM) and provides API for the modifications similar to
https://github.com/nikic/PHP-Parser.

You can create magical (in both ways) workflows and helpers by implementing your Node Visitors.

> **Note**
> You can read more about how traversing
> works [here](https://github.com/nikic/PHP-Parser/blob/master/doc/2_Usage_of_basic_components.markdown#node-traversation).

## Create Visitor

To create an AST visitor, you must implement interface provided by the Stempler engine
- `Spiral\Stempler\VisitorInterface`.

We will try to create visitor which automatically adds `alt` attribute to all `img` tags found in your templates:

```php
use Spiral\Stempler\Node\HTML;
use Spiral\Stempler\VisitorContext;
use Spiral\Stempler\VisitorInterface;

class AltImageVisitor implements VisitorInterface
{
    public function enterNode($node, VisitorContext $ctx)
    {
    }

    public function leaveNode($node, VisitorContext $ctx)
    {
        if ($node instanceof HTML\Tag && $node->name === 'img') {
            $alt = null;
            foreach ($node->attrs as $attr) {
                if ($attr->name === 'alt') {
                    $alt = $attr;
                    break;
                }
            }

            if ($alt === null) {
                $node->attrs[] = new HTML\Attr('alt', '"this is image alt"');
            }
        }
    }
}
```

> **Note**
> You can inject other tags or even PHP into your templates.

## Register Visitor

Call `StemplerBootloader`->`addVistitor` to register visitors in the template engine. We can do it using application
bootloader:

```php
namespace App\Bootloader;

use App\Visitor\AltImageVisitor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Stempler\Bootloader\StemplerBootloader;

class AltImageBootloader extends Bootloader
{
    public function boot(StemplerBootloader $stempler)
    {
        $stempler->addVisitor(AltImageVisitor::class);
    }
}
```

> **Note**
> You have to clean view cache to view newly applied changes.

Now all the `img` tags will always include `alt` attribute.

## Standalone Usage

You can use the Stempler to process any HTML content.

```php
use Spiral\Stempler;

$parser = new Stempler\Parser();
$parser->addSyntax(
    new Stempler\Lexer\Grammar\HTMLGrammar(), 
    new Stempler\Parser\Syntax\HTMLSyntax()
);

$template = $parser->parse(new Stempler\Lexer\StringStream("<BODY>content</BODY>"));

$traverser = new Stempler\Traverser();
$traverser->addVisitor(new CustomVisitor());

$template->nodes = $traverser->traverse($template->nodes);

$compiler = new Stempler\Compiler();
$compiler->addRenderer(new Stempler\Compiler\Renderer\CoreRenderer());
$compiler->addRenderer(new Stempler\Compiler\Renderer\HTMLRenderer());

dump($compiler->compile($template)->getContent());
```
