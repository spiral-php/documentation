# Keeper — 入门指南

Keeper 是一个基于 Stempler 构建的门户引擎和 UI 框架。该框架包含了各种元素，例如动态表单、数据表格、标签页和其他布局部分。您可以使用 Keeper 来创建复杂的管理员面板、用户面板和其他应用程序。

## 安装：

1. 安装依赖：

```terminal
composer require spiral/keeper
```

2. 注册一个 Keeper 的 bootloader (参见 [bootloaders](../keeper/bootloaders.md) 部分)

## 命名空间

Keeper 的基本理念是所有子模块都在给定的 `namespace` 中被隔离。例如，可以存在 `admin` 和 `profile` 这两个命名空间。

所有 Keeper 功能都与 `spiral/security` 集成，并遵循 RBAC 规则。

![Keeper Demo](https://user-images.githubusercontent.com/796136/81418518-79353800-
9155-11ea-8266-e19fb2cce45a.png)

配置可以通过代码或注解进行。

## 组件

Keeper 带有以下组件列表：

- [Forms (表单)](../keeper/components.md)
- [Autocomplete (自动补全)](../keeper/components.md)
- [DatePicker (日期选择器)](../keeper/components.md)
- [DataGrids (数据网格)](../keeper/components.md)

有关组件的更多信息，请参阅[文档](https://github.com/spiral/app-keeper)。
