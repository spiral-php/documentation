# 高级 — 静态内存

框架 (组件 `spiral/boot`) 提供了一个方便的接口，用于存储进程之间共享的一些计算数据。

> **注意**
> 当前共享内存的实现通过 OpCache 将数据存储在物理文件中。未来的实现会将数据存储移至 RoadRunner 或使用 SHM 的共享 PHP 扩展，请不要将您的代码库与物理文件耦合。

## MemoryInterface

使用接口 `Spiral\Boot\MemoryInterface` 存储计算结果：

```php
/**
 * 长期内存缓存。使用此存储来记住您的计算结果，不要在此处存储用户或非静态数据 (!)。
 */
interface MemoryInterface
{
    /**
     * 从长期内存缓存中读取数据。必须返回与保存值完全相同的值，或者为 null。目前的惯例允许存储可序列化（var_export-able）的数据。
     *
     * @param string $section 不区分大小写。
     * @return string|array|null
     */
    public function loadData(string $section);

    /**
     * 将数据放入长期内存缓存。不允许内部引用或闭包。目前的惯例允许存储可序列化（var_export-able）的数据。
     *
     * @param string       $section 不区分大小写。
     * @param string|array $data
     */
    public function saveData(string $section, $data);
}
```

## 用例

内存的一般思想是通过缓存某些功能的执行结果来加速应用程序。内存组件用于存储配置缓存、ORM 和 ODM 模式、控制台命令列表和分词器缓存。它也可以用于缓存编译后的路由等。

> **注意**
> 应用程序内存永远不能用于存储用户数据。

## 实际例子

让我们看一个用于分析可用类以计算一些行为（操作）的服务示例：

```php
abstract class Operation
{
    /**
     * 执行某些操作。
     */
    abstract public function perform(mixed $request): void;
}

class OperationService
{
    /**
     * 与其类关联的操作列表。
     * @var class-string[] 
     */
    protected array $operations = [];

    public function __construct(
        MemoryInterface $memory, 
        ClassesInterface $classes
    ) {
        $this->operations = $memory->loadData('operations');

        if (\is_null($this->operations)) {
            $this->operations = $this->locateOperations($classes); // 慢操作
            $memory->saveData('operations', $this->operations);
        }      
    }

    public function run(string $operation, mixed $request): void
    {
        // 根据 $operations 属性执行操作
    }

    /**
     * @return class-string[]
     */
    protected function locateOperations(ClassesInterface $classes): array
    {
        // 通过扫描每个可用的类来生成可用操作的列表
    }
}
```

> **注意**
> 您目前只能在内存中存储数组或标量值。

您可以使用 APC、XCache、RoadRunner 上的 DHT、Redis 甚至 Memcache 来实现您自己的 `Spiral\Boot\MemoryInterface` 版本。

在您将 `Spiral\Boot\MemoryInterface` 嵌入到您的组件或服务之前：

* 不要存储任何与用户请求、操作或信息相关的数据。内存仅用于逻辑缓存
* 假设内存随时可能消失
* `saveData` 是线程安全的，但随着并发性的提高而变慢
* `saveData` 比 `loadData` 更昂贵，请确保在应用程序运行时不要在内存中存储任何内容
* [引导程序](../framework/bootloaders.md) 和 [命令](../console/commands.md) 是使用内存的最佳位置
