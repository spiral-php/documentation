# 入门 — 安装

Spiral 提供了一个方便的应用程序安装程序，允许开发人员通过命令行界面轻松地安装和配置框架以及任何所需的包。这有助于简化使用 Spiral 的过程，并简化开发环境的设置。

这对于安装所有必需的配置文件、环境变量和期望的包和组件的引导程序非常有用。安装程序允许开发人员轻松安装各种包和组件，包括：

- [Cycle 桥接](../basics/orm.md)
- [验证器](../validation/factory.md)
- [事件调度器](../advanced/events.md)
- [RoadRunner 桥接](../start/server.md)
- [Temporal 桥接](../temporal/configuration.md)
- [Cron 作业调度程序](../advanced/scheduler.md)
- [Sentry 桥接](../basics/errors.md)
- [Stempler](../views/stempler.md) 或 [Twig](../views/twig.md) 模板引擎
- [序列化器](../advanced/serializer.md)
- [邮件组件](../advanced/sendit.md)
- [RoadRunner 指标](../advanced/prometheus-metrics.md)

通过安装程序准备好这些包和组件并正确配置，可以节省开发人员设置开发环境和启动项目的时间和精力。

## 服务器要求

确保您的服务器配置了以下 PHP 版本和扩展：

* PHP 8.1+，64 位
* *mb-string* 扩展（spiral 是以 UTF-8 为中心的框架）
* *socket* 扩展
* *curl* 扩展
* *zip* 扩展

## 安装

使用安装程序的安装过程非常简单易用。您可以使用以下命令创建一个新项目：

```terminal
composer create-project spiral/app my-app
```

您将看到以下输出：

```output
 [32mCreating a "spiral/app" project at "./my-app" [39m
 [32mInstalling spiral/app [39m
 [32m1.1.1 [39m
  - Installing  [32mspiral/app [39m ( [33m1.1.1 [39m): Extracting archive
 [32mCreated project in /var/www/my-app [39m
> Installer\Installer::install

   [30;46mWhich application preset do you want to install? [39;49m
  [ [33m1 [39m] Web
  [ [33m2 [39m] Cli
  [ [33m3 [39m] gRPC
  Make your selection  [33m(1) [39m:
```

> **注意**
> 当安装过程中出现问题时，您始终可以通过运行 `composer install` 命令来重新启动它。

安装应用程序后，将在项目的根目录中生成 `README.md` 文件，其中包含有关如何启动应用程序服务器以及如何运行应用程序的说明。

> **警告**
> RoadRunner 应用程序服务器将自动下载。需要 `php-curl` 和 `php-zip`。

## 运行服务器

要启动应用程序服务器，请执行：

:::: tabs

::: tab Linux

使用以下命令在 **Linux** 上运行 RoadRunner 服务器

```terminal
./rr serve
```

> **警告**
> 确保 `rr` 二进制文件是可执行的。

:::

::: tab Windows
使用以下命令在 **Windows** 上运行 RoadRunner 服务器

```terminal
.\rr.exe serve
```

:::

::::

默认情况下，该应用程序将在 `http://localhost:8080` 上可用。

> **了解更多**
> 在 [入门 — 应用程序服务器](../start/server.md) 部分阅读更多关于应用程序服务器的信息。

<hr>

## 接下来做什么？

现在，通过阅读一些文章深入了解基础知识：

* [应用程序服务器](../start/server.md)
* [目录结构](../start/structure.md)
* [第一个 HTTP 控制器](../start/http-basics.md)
