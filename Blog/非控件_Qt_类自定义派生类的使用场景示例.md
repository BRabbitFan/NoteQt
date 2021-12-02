<!--
author: BRabbitFan
date: 2021-10-09
title: 非控件 Qt 类自定义派生类的使用场景示例
tags: Qt,C++,学习笔记
category: Qt学习笔记
status: publish
summary: 关于自定义 QLayout 与 自定义 QGraphicsEffect 的示例.
-->

> 转载请注明原文链接: http://brabbit.xyz/blog/NoteQt/Blog/非控件_Qt_类自定义派生类的使用场景示例.html

---

# 文章目录

- [前言](#前言)
- [`QLayout` 子类 --- 自定义层叠布局器](#QLayout子类)
- [`QGraphicsEffect` 子类 --- 自定义毛玻璃图像效果器](#QGraphicsEffect子类)

---

<div id="前言"></div>

# 前言

继承 Qt 类往往是为了自定义 UI 控件, 如 `QWidget` 的自定义派生类等. 但在一些特殊情况下可能需要对非 UI 控件类实现一些自定义的派生类. 

---

<div id="QLayout子类"></div>

# `QLayout` 子类 --- 自定义层叠布局器

Qt 原生提供的布局器有这么几种:

- `QVBoxLayout` : 垂直布局器
- `QHBoxLayout` : 水平布局器
- `QGridLayout` : 网格布局器
- `QFormLayout` : 表单布局器

这四种布局器可以直接在 Qt Designer 中使用. 而当需要使用到一些特殊的布局时, 这四种布局器可能无法满足需求.

### 实现 `QLayout` 的派生类

Qt 中所有的布局器都继承自 `QLayout` 基类. 若我们想自定义布局器, 则需要基础该基类同时实现以下几个方法:

- `virtual void QLayout::addItem(QLayoutItem *) = 0;` : 向布局器加入新的 item.
- `virtual QLayoutItem *QLayout::itemAt(int index) const = 0;` : 获取布局器中的 item.
- `virtual QLayoutItem *QLayout::takeAt(int index) = 0;` : 取出布局器中的 item.
- `virtual QSize QLayoutItem::sizeHint() const = 0;` : 获取控件的 Hint Size.
- `virtual void QLayoutItem::setGeometry(const QRect&) = 0;` : 设置控件坐标的方法.

以上几个纯虚函数是必须要实现的. 其中前三者通常用于用户控制布局器, 后二者通常被 Qt 所调用. 其中 `setGemoetry` 是最重要的一个方法, 这个方法决定了我们的自定义布局器要如何布局其所管理的 **布局项目(QLayoutItem)** . 

在这几个方法之外, Qt 官方也建议我们实现下面这两个方法:

- `QSize minimumSize() const override;` : 获取控件的 Minimum Size
- `QSize maximumSize() const override;` : 获取控件的 Maximum Size

这两个方法并不是必须的, 但如同 `sizeHint` 一样也会经常被 Qt 所调用. 实际上在 Qt 源码中, 所有派生的 Layout 都实现了这两个功能.

以上部分都是官方的规定/建议, 在实际操作中通常还需要在布局器中使用一个容器来保存受布局器管理的 **布局项目** . 具体用哪种容器则根据实际情况决定. 

#### 布局项目 --- `QLayoutItem`

在布局器中, 并不会直接管理控件, 内部的容器也不会保存 `QWidget`. 我们在布局器中操作的是 `QLayoutItem`, 它抽象出一个受布局器所管理的对象需要有的各种方法. 但 `QLayoutItem` 也是一个基类. 一个对象被加入布局器后会首先转化成它的某个派生类, 可能是 `QSpacerItem` 或 `QWidgetItem` 或 `QLayout`, 最后用一个基类 `QLayoutItem` 的指针加入到布局器中.

上文中说到实现自定义布局器需要继承 `QLayout` 并实现几个纯虚方法, 其中 `QLayoutItem::sizeHint` 与 `QLayoutItem::setGeometry` 并不是 `QLayout` 的方法, 而是 `QLayoutItem` 的. 这是因为 `QLayout` 也继承自 `QLayoutItem`

### 自定义堆叠布局器 --- `MyStackedLayout`

这里举一个例子来制作一个简单的自定义布局器. 我们的需求如下: 某些控件需要在父级控件内同时地重叠显示出来. 此时我们使用上文提到的四种布局器都是不好实现的. 于是我们自定义一个 `QLayout` 的派生类, 做一个自定义的"堆叠布局器" `MyStackedLayout` 来达成目标

我们在内部使用 `QList` 作为保存所有布局项目的容器, 同时在 `setGeometry` 方法中设置所有的布局项目坐标与尺寸始终保持与父级控件一致, 以此达成最基本的层叠显示效果. 具体代码见下:

```cpp
// MyStackedLayout.hpp
class MyStackedLayout : public QLayout {
private: Q_OBJECT;
public:  explicit MyStackedLayout(QWidget* parent) : QLayout(parent) {};
public:  ~MyStackedLayout() = default;

private: QList<QLayoutItem*> itemList_;  // 保存布局项目的容器

/* 操作 itemList_ 的方法, addItem 加入项目, itemAt 获取项目, takeAt 取出项目 */

public: void addItem(QLayoutItem* item) override {
  if (itemList_.contains(item)) { return; }
  itemList_.append(item);
}

public: QLayoutItem* itemAt(int index) const override {
  if (index > itemList_.count() - 1) { return nullptr; }
  return itemList_.at(index);
}

public: QLayoutItem* takeAt(int index) override{
  if (index > itemList_.count() - 1) { return nullptr; }
  return itemList_.takeAt(index);
}

/* 设置项目的坐标, 尺寸. 始终与父级控件保持一致 */

public: void setGeometry(const QRect&) override{
  for (auto item : itemList_) { item->setGeometry(rect); }
  __super::setGeometry(rect);
}

/* 获取 Hint Size, Minimum Size, Maximum Size */
public: inline QSize sizeHint() const override { return parentWidget()->size(); }
public: QSize minimumSize() const override { return sizeHint(); }
public: QSize maximumSize() const override { return sizeHint(); }
};
```

具体实现非常简单, 其中最重要的就是 `setGeometry` 方法, 这里在这里写了布局的规则, 实际运用中 Qt 将主动调用这个方法来帮我们管理布局.

### Qt 原生堆叠布局器 --- `QStackedLayout`

上述例子的功能其实 Qt 也实现了. 当我们使用 `QStackedWidget` 的时候, 其内部使用的就是一个特殊的布局器 `QStackedLayout` . 这个布局器虽然在 Qt Designer 中找不到, 但我们还是可以在代码中使用.

具体使用方式不赘述了, 唯一要注意的是我们这里的需求是显示所有的控件, 并且重叠. 因此我们需要通过 `QStackedLayout::setStackingMode` 方法设置 `QStackedLayout::StackAll` 显示布局内的所有控件.

---

<div id="QGraphicsEffect子类"></div>

# `QGraphicsEffect` 子类 --- 自定义毛玻璃图像效果器

Qt 原生提供的图像效果器有这么几种:

- `QGraphicsBlurEffect` : 模糊效果器
- `QGraphicsColorizeEffect` : 色调效果器
- `QGraphicsDropShadowEffect` : 阴影效果器
- `QGraphicsOpacityEffect` : 透明度效果器

当我们的需求不局限于这四种效果的情况下, 就需要自定义图像效果器了.

### 实现 `QGraphicsEffect` 的派生类

Qt 中所有的图像效果器均继承自 `QGraphicsEffect` 基类, 若我们想自定义效果器则需要基础该基类同时实现下面的方法:

- `virtual void QGraphicsEffect::draw(QPainter *painter) = 0;` : 绘制图像效果

在这个方法中我们可以通过参数获得 `QPainter` , 同时可以通过 `QGraphicsEffect::sourcePixmap` 方法获得原视画面的 `QPixmap` . 有了这两个对象, 我们就可以在图像上绘制我们想要的效果了.

当遇到不需要添加效果的情况, 可以调用 `QGraphicsEffect::drawSource(QPainter *painter)` 方法将 `QPainter` 对象传递给父级.

### 自定义毛玻璃图像效果器 --- `MyGrassEffect`

这里举一个例子来制作一个简单的自定义图像效果器. 我们的需求如下: 需要对某个控件的部分区域设置毛玻璃效果. 使用上述四中效果器是没法实现的, 它们甚至无法在部分区域进行设置效果. 因此必须实现自定义的图像效果器.

关于毛玻璃效果的实现这里不多赘述, 详见: [图像毛玻璃效果与图像模糊算法](http://brabbit.xyz/blog/%E6%9D%82%E9%A1%B9/%E5%9B%BE%E5%83%8F%E6%AF%9B%E7%8E%BB%E7%92%83%E6%95%88%E6%9E%9C%E4%B8%8E%E5%9B%BE%E5%83%8F%E6%A8%A1%E7%B3%8A%E7%AE%97%E6%B3%95.html)

我们加入成员变量来管理生效的区域, 再通过实现 `draw` 方法在其中绘制毛玻璃效果, 具体代码见下:

```cpp
// MyStackedLayout.hpp
#include "OpenCvUtil.hpp"  // 详见 "图像毛玻璃效果与高斯模糊算法"

class MyGrassEffect : public QGraphicsEffect {
private:   Q_OBJECT;
public:    MyGrassEffect(QObject* parent = nullptr) : QGraphicsEffect(parent) { }
public:    ~MyGrassEffect() = default;

private:   QRect effectRect_ = QRect(-1, -1, -1, -1);  // 生效区域
public:    const QRect& getEffectRect() const { return effectRect_; }
public:    void setEffectRect(QRect effectRect) { effectRect_ = effectRect; }

/* 绘制毛玻璃效果 算法实现详见 "图像毛玻璃效果与高斯模糊算法" */
protected: void draw(QPainter* painter) override {
  if (effectRect_.isEmpty()) {
    return __super::drawSource(painter);  // 不需要加入效果的情况传递给父级
  }

  auto sourceRect  = sourcePixmap().rect();
  auto sourceImage = sourcePixmap().toImage();
  auto sourceCvmat = OpenCvUtil::qimageToCvMat(sourceImage);
  auto effectCvmat = OpenCvUtil::addGrassStyle(sourceCvmat, effectRect_);
  auto effectImage = OpenCvUtil::cvMatToQImage(effectCvmat);

  painter->drawPixmap(sourceRect, QPixmap::fromImage(effectImage));
}
};
```

实际运用中可以像使用其他图像效果器一样地使用这个图像效果器, Qt 将主动调用其中的 `MyGrassEffect::draw` 方法来绘制图像效果.

---