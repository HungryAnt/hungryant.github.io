---
layout: post
title:  "WPF技术分享"
date:   2014-04-20 00:00:00
categories: 
---

## WPF概述

- Windows Presentation Foundation是一个用于Windows的图形显示系统
- 针对.NET而设计，受现代显示技术以及硬件加速技术影响
- Windows95以来对Windows用户界面影响最深远的技术

### Windows图形演化

#### 传统显示技术

超过20年以来，Windows开发人员一直在使用本质上上相同的显示技术

标准Windows应用程序依靠系统提供的如下两个部分创建用户界面

	- User32
	
		为 窗口、按钮、文本框 等提供基本Windows外观

	- GDI/GDI+

		为渲染简单形状、文本以及图像提供绘图支持 （增加复杂度，通常性能较差）

Winforms/MFC/VB等提供不同API，但底层都是使用相同部分来工作。新的UI框架，只是提供了User32与GDI/GDI+交互的包装器而已。本质相同

#### DirectX

针对User32与GDI/GDI+的限制，微软提供了一套新的解决方案：DirectX。

- 针对旧技术的限制，新的解决方案
- 图形引擎，复杂的纹理映射、特殊效果、3D图形
- API用于游戏开发
- 硬件加速，GPU完成渲染工作

由于其复杂性，几乎不用于开发传统Windows应用程序。

**WPF扭转这一局面，其不再是GDI/GDI+的包装器，而是替代技术，底层使用DirectX**

硬件加速，GPU完成渲染工作，CPU去完成其他工作

### 体系结构

![体系结构](/image/posts/2014-04-20-cs-wpf-share-01.png)

## XAML

Extensible Application Markup Language  发音 "zammel“

用于实例化.NET对象的标记语言。主要用于构造WPF用户界面。


> 开发人员很早就意识到，要处理图形丰富的复杂应用程序，最有效的方式就是讲图形部分从底层的代码中分离出来。
艺术人员可以独立地设计图形，开发人员可以独立地编写代码。 这两部分工作可以单独地进行设计与修改，而不会有任何版本问题。

### XAML基本概念

- 在XAML文档中的所有元素都映射为一个.NET类的实例
- 与所有XML文档一样，可以在一个元素中嵌套另一个元素
- 可以通过特性(attribute)设置每个类的属性(property)

Demo演示：Coding Demo09_Xaml

- Xaml名称空间
- 代码隐藏类
- 命名元素
- 简单属性与类型转换器
- 复杂属性 如Background
- 标记扩展
- 附加属性
- 事件


## 布局与控件

### 布局

- 抵制基于坐标的布局注重创建灵活的布局，以使布局能够适应内容的变化
- WPF窗口只能包含一个元素，需要在窗口上放置一个容器，然后在容器中添加其他元素

![im](/image/posts/2014-04-20-cs-wpf-share-02.png)

![layout](/image/posts/2014-04-20-cs-wpf-share-03.jpg)

### 布局重要原则

- 不应显示设定元素(例如控件)的尺寸

	元素应该可以改变尺寸以适合它们的内容。

	可以设置最大和最小尺寸来限制可以接受的控件尺寸范围
	
- 不应使用屏幕坐标指定元素的位置

	元素应当由它们的容器，根据它们的尺寸、顺序以及（可选的）其他特定于具体布局容器的信息进行安排。如果需要在元素之间添加空白空间，可以使用Margin属性

- 布局容器和它们的子元素“共享”可以使用的空间

	如果空间允许，布局容器会（根据元素的内容）尽可能为所含的元素设置更合适的尺寸，它们还会向一个或多个子元素分配多余的空间
	
- 可以嵌套布局容器

### 布局过程

- 测量(meature)阶段

	容器遍历所有子元素，并询问子元素它们所期望的尺寸

- 排列(arrange)阶段

	容器在合适的位置放置子元素

### 布局容器

- 核心布局面板

- StackPanel
- WrapPanel
- DockPanel
- Grid
- UniformGrid
- Canvas

![The hierarchy of the Panel class](/image/posts/2014-04-20-cs-wpf-share-04.jpg)


### 控件

WPF包含几类控件

- 内容控件（Label、Button、ToolTip等）

	这些控件能够包含嵌套的元素，有无限的显示能力

- 带有标题的内容控件（TabItem、GroupBox、Expander等）

	这些控件是允许添加一个主要内容部分以及一个单独标题部分的内容控件

- 文本控件（Textbox、Passwordbox、RichTextBox）

	允许用户输入文本

- 列表控件（ListBox、CommboBox、DataGri、TreeView）

	这些控件在列表中显示项目集合

- 基于范围的控件（Slider、ProgressBar）

	这些控件通常指关心一个属性：Value，可以使用预先规定范围内的任何数字设置该属性

- 日期控件（WPF4支持  Calendar、DatePicker）

Demo05 内容控件

![demo05截图](/image/posts/2014-04-20-cs-wpf-share-05.jpg)

更多WPF控件

[Extended WPF Toolkit](http://wpftoolkit.codeplex.com/)

## 元素绑定

- 将元素绑定到一起

	可以使用从元素到元素的绑定使元素的交互方式自动化，从而当用户修改控件时，另一个元素会被自动更新。

- Demo07_Binding

	将单行文本TextBlock元素的FontSize属性绑定到Slider控件的Value属性

- 常见的绑定形式

	- 绑定到其他元素的某个依赖属性(DependencyProperty)
	- 绑定到资源
	- 绑定到任何类型的静态属性/字段


## 资源

- WPF资源系统是一种保管一系列对象的简单方法（例如常用的画刷、样式或模板），从而可以更容易地重用这些对象
- 通常在XAML标记中定义资源
- 资源可以在特定元素、特定窗口或者在整个应用程序中定义


Demo08_Resources

- 静态资源 与 动态资源
- 应用程序资源
- 资源字典
- 程序集之间共享资源


## 样式与模板

- 样式Style
	- 组织和重用格式化选项，对相同的元素无需使用重复的标记填充XAML（如 外边距Marging、内边距Padding、颜色、字体 等）
	- 可以创建一系列封装所有细节的样式，在需要之处通过一个属性应用样式

	WPF样式系统和HTML标记中的层叠样式表(cascading style sheet, CSS)标准扮演类似的角色


- 模板Template
	- 控件模板ControlTemplate

		ControlTemplate用于描述控件本身. 使用TemplateBinding来绑定控件自身的属性, 比如`{TemplateBinding Background}`

	- 数据模板DataTemplate
		DataTemplate用于描述控件的Content. 使用Binding来绑定数据对象的属性, 比如`{Binding PersonName}`

- Demo

	Coding  Demo09 转为使用Style
	- Demo
		- 窗口样式
		- TabControl样式
	- Demo_2048 数据模板

## 动画

- WPF内置了强大的动画功能
- 简单易用，直接操作UI元素
- Demo
	- C#代码或者Xaml构建动画

> Winform中使用动画，使用定时器，每间隔几时毫秒则迫使窗口无效，使得Paint方法得到调用，使用GDI+提供的各种绘图方法绘制图形

## 数据绑定DataBinding

- 使得软件由事件驱动转变为数据驱动

- Demo04_FileDownloader
	- 文本编辑框数据绑定
	- ItemsControl数据绑定
	- Command绑定


## MVVM

MVVM（Model-View-ViewModel）

WPF、Silverlight、WP7软件中常用的架构模式

- The Model

	- Model察觉不到除自身之外的任何其他模块
	- Model可以被任何程序使用，不被限定关联于任何特殊的框架或技术

- The View

	- View尽量做到全部由XAML定义
	- View只需要知道需要绑定的对象的名称。View不需要知晓属性(Properties)变更是会发生什么事，也不需要知道命令(Commands)的执行过程
	- View的状态完全依赖于数据绑定

- The ViewModel

	- ViewModel对View完全不知晓
	- ViewModel与Model直接交互，目的是准备好数据以便View进行数据绑定

![mvvm 图示](/image/posts/2014-04-20-cs-wpf-share-06.png)

![mvvm 图示](/image/posts/2014-04-20-cs-wpf-share-07.png)