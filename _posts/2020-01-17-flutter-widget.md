---
layout: post
title: Widget 知识点
excerpt_separator: "<!--more-->"
categories:
  - Flutter
tags:
  - Widget
last_modified_at: 2020-01-17T22:17:01-05:0
---


![图1 Flutter 架构图](https://flutter.dev/assets/resources/diagram-layercake-73512ded89f7df8301f622c66178633f04f91187822daf1ddff0d54b2d2676dc.png)

### 问题

- 什么是 Widget ?
- 如何理解 Widget 、Element 和 RenderObject 这三个概念？它们之间是一一对应的吗？能否在 Android/iOS/Web 中找到对应的概念呢？
  
在 iOS 中 UIView 相当于 Element, CALayer 相当于 RenderObject. UIKit 没有 Widget 这个概念。


### 概念

**Flutter 的核心设计思想是"一切 Widget"**。

Widget 是 Flutter 功能的抽象描述，是视图的配置信息，同样也是数据的映射，是 Flutter 开发框架中最基本的概念。如 View, View Controller, Activity, Application, Layout等，在 Flutter 中都是 Widget。

### Widget 渲染过程

Flutter 将视图树的概念进行了扩展，把视图数据的组织和渲染抽象为三部分，即 Widget，Element 和 RenderObject。这三部分之间的关系，如下所示：

![img](https://static001.geekbang.org/resource/image/b4/c9/b4ae98fe5b4c9a7a784c916fd140bbc9.png)

### Widget

Widget 是 Flutter 世界里对视图的一种结构化描述，可看作前端中的“控件”或“组件”。Widget 是控件实现的基本逻辑单位，里面存储的是有关视图渲染的配置信息，包括布局、渲染属性、事件响应信息等。

在页面渲染上，Flutter 将“Simple is best”这一理念做到了极致。Flutter 将 Widget 设计成不可变的，所以当视图渲染的配置信息发生变化时，Flutter 会选择重建 Widget 树的方式进行数据更新，以数据驱动 UI 构建的方式简单高效。但这样做的缺点是，因为涉及到大量对象的销毁和重建，所以会对垃圾回收造成压力。不过，Widget 本身并不涉及实际渲染位图，所以它只是一份轻量级的数据结构，重建的成本很低。

另外，由于 Widget 的不可变性，可以以较低成本进行渲染节点复用，因此在一个真实的渲染树中可能存在不同的 Widget 对应同一个渲染节点的情况，这无疑又降低了重建 UI 的成本。

### Element

Element 是 Widget 的一个实例化对象，它承载了视图构建的上下文数据，是连接结构化的配置信息到完成最终渲染的桥梁。

Flutter 渲染过程，可以分为这么三步：
- 首先，通过 Widget 树生成对应的 Element 树；
- 然后，创建相应的 RenderObject 并关联到 Element.renderObject 属性上；
- 最后，构建成 RenderObject 树，以完成最终的渲染。

可以看到，Element 同时持有 Widget 和 RenderObject。而无论是 Widget 还是 Element，其实都不负责最后的渲染，只负责发号施令，真正去干活儿的只有 RenderObject。那么，为什么不直接 Widget 命令 RenderObject 去干活呢？因为这样会极大增加渲染带来的性能损耗。因为 Widget 具有不可变性，但 Element 却是可变的。实际上，Element 树这一层将 Widget 树的变化（类似 React 虚拟 DOM diff）做了抽象，可以只将真正需要修改的部分同步到真实的 RenderObject 树中，最大程度降低对真实渲染视图的修改，提高渲染效率，而不是销毁整个渲染视图树重建。这，就是 Element 树存在的意义。

### RenderObject

RenderObject 是主要负责实现视图渲染的对象。渲染对象树在 Flutter 的展示过程分为四个阶段，即布局、绘制、合成和渲染。其中，布局和绘制在 RenderObject 中完成，Flutter 采用深度优先机制遍历渲染对象树，确定树中各个对象的位置和尺寸，并把它们绘制到不同的图层上。绘制完毕后，合成和渲染的工作则交给 Skia 搞定。

**Flutter 通过引入 Widget、Element 与 RenderObject 这三个概念，把原本从视图数据到视图渲染的复杂构建过程拆分得更简单、直接，在易于集中治理的同时，保证了较高的渲染效率。**

### RenderObjectWidget

在 Flutter 中，布局和绘制工作实际上是在 Widget 的另一个子类 RenderObjectWidget 内完成的。

RenderObjectWidget 的源码如下:

```dart
abstract class RenderObjectWidget extends Widget {
  @override
  RenderObjectElement createElement();
  @protected
  RenderObject createRenderObject(BuildContext context);
  @protected
  void updateRenderObject(BuildContext context, covariant RenderObject renderObject) { }
  ...
}
```
RenderObjectWidget 是一个抽象类。我们通过源码可以看到，这个类中同时拥有创建 Element、RenderObject，以及更新 RenderObject 的方法。实际上，RenderObjectWidget 本身并不负责这些对象的创建与更新。

对于 Element 的创建，Flutter 会在遍历 Widget 树时，调用 createElement 去同步 Widget 自身配置，从而生成对应节点的 Element 对象。而对于 RenderObject 的创建与更新，其实是在 RenderObjectElement 类中完成的。

```dart
abstract class RenderObjectElement extends Element {
  RenderObject _renderObject;

  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    _renderObject = widget.createRenderObject(this);
    attachRenderObject(newSlot);
    _dirty = false;
  }
   
  @override
  void update(covariant RenderObjectWidget newWidget) {
    super.update(newWidget);
    widget.updateRenderObject(this, renderObject);
    _dirty = false;
  }
  ...
}
```

在 Element 创建完毕后，Flutter 会调用 Element 的 mount 方法。在这个方法里，会完成与之关联的 RenderObject 对象的创建，以及与渲染树的插入工作，插入到渲染树后的 Element 就可以显示到屏幕中了。

如果 Widget 的配置数据发生了改变，那么持有该 Widget 的 Element 节点也会被标记为 dirty。在下一个周期的绘制时，Flutter 就会触发 Element 树的更新，并使用最新的 Widget 数据更新自身以及关联的 RenderObject 对象，接下来便会进入 Layout 和 Paint 的流程。而真正的绘制和布局过程，则完全交由 RenderObject 完成：

```dart
abstract class RenderObject extends AbstractNode with DiagnosticableTreeMixin implements HitTestTarget {
  ...
  void layout(Constraints constraints, { bool parentUsesSize = false }) {...}
  
  void paint(PaintingContext context, Offset offset) { }
}
```
布局和绘制完成后，接下来的事情就交给 Skia 了。在 VSync 信号同步时直接从渲染树合成 Bitmap，然后提交给 GPU。

### 总结

Flutter 中视图数据的组织和渲染抽象的三个核心概念: Widget 、Element 和 RenderObject。

Widget 是 Flutter 世界里对视图的一种结构化描述，里面存储的是有关视图渲染的配置信息；Element 则是 Widget 的一个实例化对象，将 Widget 树的变化做了抽象，能够做到只将真正需要修改的部分同步到真实的 Render Object 树中，最大程度地优化了从结构化的配置信息到完成最终渲染的过程；而 RenderObject，则负责实现视图的最终呈现，通过布局、绘制完成界面的展示。