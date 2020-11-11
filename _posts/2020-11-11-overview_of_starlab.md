---
layout:     post
title:      "Starlab 源码剖析系列（一）"
subtitle:   "总览 Starlab 设计"
date:       2020-11-11
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - Starlab
    - meshlab
    - 图形学
---

哈喽，各位好，这里我要挖一个大坑了，具体是什么呢，听我细细道来。

也许大家没看出来，其实我不仅仅是一个努力学习后端的程序员，还是个稍微会点儿图形编程的图形学爱好者。我们在学习图形学算法时，常常碰到的坑是什么呢，是怎么写一个三维场景。
**试想一下：我想要学习写一个图形算法，直接使用 OpenMesh 读取模型，然后处理后另存为一个新的模型。这样子算法虽然跑成功了，但是很难看到即时效果，也比较难以交互性调整参数。又或者像程序员的浪漫一样，自己造轮子，写一个UI框架，然后新的算法都在框架中添加，每新增一个算法便重新添加一个 `QAction`，为其编写 UI `Widget`。**

有没有什么方法可以让自己写图形算法时不需要考虑这么多 UI 上的问题呢？还真有，**meshlab、starlab 等开源图形编辑软件设计上采取了大量插件形式的设计，你可以将算法以插件的形式编写，然后复用软件的 GUI 部分，可以使你大量的精力集中在算法设计而不是 UI 编写上。**

## Starlab

那么 Starlab 相比于 meshlab 有什么优势吗？ 其实 starlab 自 2015年后便没有继续更新，像是被废弃了的框架。不过它在设计上与 meshlab 及其相似，主要的思路上也及其的相似。而且不再继续更新的好处是，你可以随意魔改 Starlab 的代码，不需要考虑 Starlab 主版本更新了，自己的魔改就没法继续用了。因为，**Starlab 大概率不会继续更新了，开发人员跑路了**。

本篇文章我们先简单介绍下 Starlab 的设计思路。

## 设计思路

Starlab 采用插件化设计，使用了 qt plugin 的功能，具体用法参见之前的文章 [qt plugin](https://www.mcoder.cc/2020/10/30/qt_plugin/)。
它将程序主要分为如下几个部分：

* GUI 启动程序 starlab 工程
* lib
  * surfacemesh starlab 中所使用的的数据结构
  * starlib 关于 UI 插件管理等内容
* 各类插件
  * GUI 面板插件
  * filter 插件
  * mode 插件
  * io 插件
  * Render 插件
  * Decoratre 插件

可以说，核心代码根本不包含图形算法、渲染算法，大部分的算法都通过插件实现。

## 插件

### Filter 插件

Filter 插件适合写一些比较简单的图形处理算法，他通过 `RichParamter` 自动生成 UI，无需亲自编写 Widget，但其灵活度受限。在重写 `initParameters` 函数中配置参数，由 starlab 自动生成 UI 面板，在 `applyFilter` 函数中执行需要的算法即可。

```cpp
class STARLIB_EXPORT FilterPlugin : public StarlabPlugin{
public:
    // 应用 Filter
    virtual void applyFilter(RichParameterSet*) = 0;
    // 初始化 UI
    virtual void initParameters(RichParameterSet*){}
    // 检查当前场景中的模型是否符合 Filter 执行条件
    virtual bool isApplicable(Starlab::Model* model) = 0;

    /// @{ access to resources
    protected:
        using StarlabPlugin::document;
        using StarlabPlugin::drawArea;
        Starlab::Model* model(){ return document()->selectedModel(); }
    /// @}
};
```

### Mode 插件

Mode 插件具有更为丰富的功能，可以监控鼠标、键盘等交互，但控制面板需要自己手写 `Widget` 实现，适合参数较为复杂、或者需要鼠标交互逻辑的图形算法。Mode 模式通常在 `create` 函数中自己初始化一个 `Widget`，后续的图形算法调用可通过在 `Widget` 上绘制 `PushButton` 来手动调用。

```cpp
class STARLIB_EXPORT ModePlugin : public StarlabPlugin {
    Q_OBJECT
public:
virtual bool isApplicable() = 0;

public:
    // 初始化 Mode 插件
    virtual void create(){}

    // 停止使用 Mode 插件
    virtual void destroy(){}

    virtual bool documentChanged(){ return false; }
protected:
    QWidget* parent;
    ModePlugin() :parent(NULL){}
private:
    friend class Starlab::MainWindow;
    friend class gui_mode;
    void __internal_create(){
        Q_ASSERT(parent==NULL);
        parent = new QWidget(mainWindow());
        create();
    }
    void __internal_destroy(){
        Q_ASSERT(parent!=NULL);
        delete parent;
        parent = NULL;
        destroy();
    }

public:
    virtual void suspend(){}
    virtual void resume(){}

    virtual QString filter() { return QString(); }
    virtual bool loadByDrop(QString) { return false; }

public:
    virtual void decorate(){ emit decorateNeeded(); }
signals:
    void decorateNeeded();

public:
    virtual void drawWithNames(){}
    virtual bool endSelection(const QPoint& point){ Q_UNUSED(point); return false; }
    virtual bool postSelection(const QPoint& point){ Q_UNUSED(point); return false; }

public:
    virtual bool mousePressEvent        (QMouseEvent* ) { return false; }
    virtual bool mouseMoveEvent         (QMouseEvent* ) { return false; }
    virtual bool mouseReleaseEvent      (QMouseEvent* ) { return false; }
    virtual bool keyReleaseEvent        (QKeyEvent*   ) { return false; }
    virtual bool keyPressEvent          (QKeyEvent*   ) { return false; }
    virtual bool tabletEvent            (QTabletEvent*) { return false; }
    virtual bool wheelEvent             (QWheelEvent* ) { return false; }
    virtual bool mouseDoubleClickEvent  (QMouseEvent* ) { return false; }

public:
    using StarlabPlugin::drawArea;
    using StarlabPlugin::document;
    using StarlabPlugin::mainWindow;
    using StarlabPlugin::application;
    using StarlabPlugin::pluginManager;
    Starlab::Model* selection(){ return document()->selectedModel(); }
};
```

### IO 插件

IO 插件则主要在 `name` 函数中指明支持的模型后缀名，在 `open` 函数中填写加载模型逻辑，在 `save` 函数中填写保存模型逻辑。

```cpp
class STARLIB_EXPORT InputOutputPlugin : public StarlabPlugin{
public:
    virtual QString name() = 0;

    virtual bool isApplicable(Starlab::Model*) = 0;
    virtual Starlab::Model* open(QString path) = 0;
    virtual void save(Starlab::Model*, QString path) = 0;

protected:
    QString pathToName(QString path){
        QFileInfo fi(path);
        return fi.completeBaseName();
    }
private:
    friend class Starlab::Application;
    void checkReadable(QString path){
        QFileInfo fi(path);
        QString absPath = fi.absoluteFilePath();
        if(!fi.exists()) throw StarlabException("File at: '%s' does not exist.",qPrintable(absPath));
        if(!fi.isReadable()) throw StarlabException("File at: '%s' does not exist.",qPrintable(absPath));
    }
};
```

### Render 插件

Render 插件更像是个简单工厂模式，插件主要通过 `instance` 函数获得一个渲染器，将渲染器与模型绑定，使场景中的每个模型可以同时使用不同的渲染器。模型的渲染，在渲染器的 `render` 中共执行。

```cpp
class RenderPlugin : public StarlabPlugin{
public:
    virtual bool isApplicable(StarlabModel* model) = 0;

    /// 获得一个 new 出来的渲染器实例
    virtual Renderer* instance() = 0;

    virtual bool isDefault() { return false; }

    using StarlabPlugin::drawArea;
};

class STARLIB_EXPORT Renderer : public QObject{
    Q_OBJECT
public slots:
    virtual void init(){}

    virtual void render() = 0;
    virtual void initParameters(){}
    virtual void finalize(){}
public:
    StarlabModel* model(){ return _model; }
    RenderPlugin* plugin(){ Q_ASSERT(_plugin); return _plugin; }
    RichParameterSet* parameters(){ return &_parameters; }
private:
    friend class Starlab::Model;
    StarlabModel* _model;
    RenderPlugin* _plugin;
    RichParameterSet _parameters;
};

```

## 结论

Starlab 这个项目有很多不足，很多地方的设计有些臃肿导致运行速度偏慢，但因其于 meshlab 设计思路一脉相承，对于自己理解一个图形框架，以及自己可以在优化 Starlab 的过程中得到长进。
关于 Starlab 的编译及插件编写将在后续的文章中继续描述。

## Reference

1. [Starlab](https://github.com/OpenGP/starlab)
2. [Lydialab](https://github.com/mcoder2014/LydiaLab)
3. [LydiaLabExpPlugins](https://github.com/mcoder2014/LydiaLabExpPlugins)
4. [qt plugin](https://www.mcoder.cc/2020/10/30/qt_plugin/)
