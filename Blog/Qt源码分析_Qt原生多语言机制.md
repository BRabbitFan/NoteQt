<!--
author: BRabbitFan
date: 2021-10-02
title: Qt 源码分析 --- Qt 原生多语言机制的实现原理
tags: Qt,C++,学习笔记,源码分析,多语言
category: Qt学习笔记
status: publish
summary: 本文通过 Qt 源码介绍 Qt 原生多语言机制的实现原理.
-->

<div id="返回顶部"></div>

> 转载请注明原文链接: http://brabbit.xyz/blog/NoteQt/Blog/Qt源码分析_Qt原生多语言机制.html

本文通过 Qt 源码介绍 Qt 原生多语言机制的实现原理.

---

# 文章目录

- [前言](#前言)  
  - [多语言动态切换的步骤 / 目标](#多语言动态切换的步骤/目标)  
  - [实现多语言动态切换需要解决的问题](#实现多语言动态切换需要解决的问题)  
- [分析 : Qt 原生多语言机制原理与实现方式](#Qt原生多语言机制原理与实现方式)  
  - [结论在前](#结论在前)
  - [基本思路](#基本思路)
  - [源码分析](#源码分析)
    - [语言文件 --- `.ts` 文件与 `.qm` 文件](#语言文件)
      - [`.ts` 文件与 `.qm` 文件的生成](#语言文件的生成)
    - [语言翻译器 `QTranslator`](#语言翻译器QTranslator)
      - [语言翻译器的内部数据结构](#语言翻译器的内部数据结构)
      - [语言翻译器的加载文件操作 --- `do_load`](#语言翻译器的加载文件操作)
      - [语言翻译器的获取译文操作 --- `do_translate`](#语言翻译器的获取翻译文本)
      - [语言翻译器之间的附属关系](#语言翻译器之间的附属关系)
      - [歧义消除与复数处理](#歧义消除与复数处理)
    - [翻译器的加载与管理 --- `QCoreApplication`](#翻译器的加载与管理)
      - [装载翻译器 --- `QCoreApplication::installTranslator`](#装载翻译器)
      - [卸载翻译器 --- `QCoreApplication::removeTranslator`](#卸载翻译器)
    - [获取最新的翻译文本 --- `QCoreApplication::translate` 与 `QObject::tr`](#获取最新的翻译文本)
      - [获取译文 --- `QCoreApplication::translate`](#获取译文)
      - [`QObject::tr` 与 `QCoreApplication::translate` 的关系](#QObject::tr与QCoreApplication::translate的关系)
    - [更新 UI 控件的时机 --- `QEvent::LanguageChange` 事件](#更新UI控件的时机)
      - [`QEvent::LanguageChange` 事件发给了谁?](#LanguageChange事件发给了谁)
      - [`top-level widgets` 包括了哪些控件?](#toplevelwidgets包括了哪些控件)
- [总结](#总结)

---

<div id="前言"></div>

# 前言 : 多语言适配的目标与需解决的问题

应用程序多语言适配从实现思路上分两种方式 : 

- **"冷切换"** : 用户选择语言, 待程序下一次启动初始化时读取对应的语言文件.
- **"热切换"** : 用户选择语言, 程序实时地切换对应的语言文件并更新至 UI.

这里第一种 "冷切换" 的方式实现起来比较简单, 不在本文的讨论范围之内. 而关于 "热切换" 的实现方式, 我把基本原理总结为下面这三个步骤, 也是我们适配多语言时需要达成的三个阶段性目标 :

[<返回顶部>](#返回顶部)

<div id="多语言动态切换的步骤/目标"></div>

#### 多语言动态切换的步骤 / 目标 :

1. 监测到切换语言的动作, 根据目标语言加载对应的语言文件.
2. 根据从语言文件载入的数据, 更新应用程序内存放 UI 文本的数据结构.
3. 将存放 UI 文本的数据结构内的新语言文本更新至 UI 控件.

这里强调下 `2.` `3.` 两步骤, 加载新的语言文件后不应该直接设置到 UI 控件上, 除非你的 UI 界面永远不会发生变化. 所以必然需要一个储存语言文本的数据结构统一管理, 下文会再说到这一点.

至于实现细节部分, 还有很多需要考虑的点与很多要解决的问题. 这些问题不光涉及到软件工程, 同样也需要考虑到语言学和平面设计等方面. 我这里简单罗列了几条对于开发人员来说需要注意的点, 也是开发过程中大概率会遇到的问题:

[<返回顶部>](#返回顶部)

<div id="实现多语言动态切换需要解决的问题"></div>

#### 实现多语言动态切换需要解决的问题 :

1. **语法兼容** : 对于静态文本来说无所谓, 但动态拼接的文本需要考虑不同语言之间的语法差异. 如英文中的序数词和中文中的量词要如何在另一种语言中对应.
2. **歧义消除** : 同一段文本在一种语言内或许可在多处复用, 但这并不代表在另一种语言中也可以在相同的地方复用. 如英文 "Start" 在中文的不同语境下可能需要分别翻译为 "启动" 或 "开始".
3. **字符串格式化** : 与第 `1.` 条类似, 部分需要格式化拼接的文本在不同语言下的或有格式不同. 如中文日期 "2021年 十月 2号" 与英文日期 "October 2, 2021" 对应的格式化字符串不同.

这几个问题并不是全部, 若要真正做到兼容多个地区/语言的使用习惯是门大学问, 此处不再展开讨论.

[<返回顶部>](#返回顶部)

---

<div id="Qt原生多语言机制原理与实现方式"></div>

# 分析 : Qt 原生多语言机制原理与实现方式

Qt 原生多语言机制涉及到如下几个类 : `QCoreApplication`, `QObject` 及其派生类, `QTranslator` 以及 `QApplication`.  

[<返回顶部>](#返回顶部)

<div id="结论在前"></div>

## 结论在前

先说结论, Qt 实现了这么一个核心功能 : **用户可以在任意时间任意位置获取到某段指定文本的最新翻译版本**. 这个功能是其整个多语言机制的核心功能, 也是 Qt 想达成的目标(唯一目标).

此外, 关于在前言中提到的三个步骤 / 目标, Qt 只达成了前两个. 最后一个目标 Qt 没有实现完全自动化, 不过可以做到即时地通知用户更新 UI 控件. 而至于我提到的三个问题, Qt 考虑到了第二条, 但消除歧义的方式很原始. 下面讲思路.

[<返回顶部>](#返回顶部)

<div id="基本思路"></div>

## 基本思路

整个系统可以分为这么几个部分来看:

- **语言文件** : 也就是 Qt 规定的 `.ts` 文件与 `.qm` 文件, **其实就是原始 XML 文件与二进制的 XML 文件**. Qt 规定使用 XML 文件来定义语言的翻译文本. 
- **语言翻译器** :  语言翻译器 `QTranslator` 类可以载入 `.qm` 文件的内容, 保存某一类语言的翻译文本. 其核心功能就是充当上述所谓的 **存放 UI 文本的数据结构** . 
- **翻译器的加载** : Qt 允许用户使用 `QCoreApplication` 来装载不同的语言翻译器, 可以同时使用多个翻译器且越晚装载的翻译器使用优先级越高. 
- **获取最新的翻译文本** : Qt 让 `QCoreApplication::translate` 与 `QObject::tr` 等方法始终按照使用优先级顺序读取语言翻译器内的翻译文本. 
- **更新 UI 控件** : Qt 将在转载一个新的语言翻译器后通过将事件 `QEvent::LanguageChange` 发送至全体属于 `top-level widgets` 范畴内的控件, 提醒其用户 **手动** 更新 UI 上的语言文本.  

其实说到这已经概括了整个系统的工作模式, 如果只想使用这个机制的话看到这其实已经足够了. 不过我建议继续看下面更细节的实现部分, 其中会包括到一些使用方式的建议.

[<返回顶部>](#返回顶部)

<div id="源码分析"></div>

## 源码分析

下面我按照 **基本思路** 中的几个部分分别介绍, 这里的重点在于 `QCoreApplication` 与 `QTranslator`.

[<返回顶部>](#返回顶部)

<div id="语言文件"></div>

### 语言文件 --- `.ts` 文件与 `.qm` 文件

这一部分其实没什么好介绍的, `.ts` 文件其实就是 XML 文件, 完全可以按照 XML 文件格式打开. 一个标准的 `.ts` 文件结构如下:

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE TS>
<TS version="2.1" language="zh_CN">                         <!-- 翻译的目标语言 -->
<context>                                                   <!-- 一个上下文, 通常对应一个类 -->
    <name>WindowWidgetClass</name>                          <!-- 上下文名, 通常与类名相对应 -->
    <message>                                               <!-- 一条文本的信息 -->
        <location filename="WindowWidget.ui" line="14"/>    <!-- 该文本的位置, 可以在 .ui .h .cpp 等文件中 -->
        <source>WindowWidget</source>                       <!-- 原文内容 -->
        <translatorcomment>这是窗口</translatorcomment>      <!-- 提供给翻译人员的注释 -->
        <translation type="unfinished">窗口</translation>   <!-- 译文内容 -->
    </message>
    <message>                                               <!-- 另一条文本的信息 -->
        <location filename="WindowWidget.cpp" line="52"/>
        <source>TextLabel</source>
        <translatorcomment>这是标签</translatorcomment>
        <translation type="unfinished">标签</translation>
    </message>
</context>
<context>                                                   <!-- 另一个上下文 -->
    <name>SubWidget</name>
    <message>
        <location filename="SubWidget.ui" line="14"/>
        <source>SubWidget</source>
        <translation type="unfinished"></translation>
    </message>
</context>
</TS>
```

而 `.qm` 文件则是转为二进制后的 `.ts` 文件. 也是程序最终会加载的文件. `.qm` 文件的内容格式没法直接看到, 但在下文中我们能够从侧面推测出它的基本结构.

> 个人猜测实际上 `.ts` 到 `.qm` 过程并不只是单纯地转二进制. 在这之前很可能还有对 `.ts` 文件内容的删减以及标签位置的哈希计算等, 后面会再谈到这一点.

[<返回顶部>](#返回顶部)

<div id="语言文件的生成"></div>

#### `.ts` 文件与 `.qm` 文件的生成

我们无需手动地去设置某个原文与它的上下文等这些参数, 当我们使用 Qt linguist 工具在工程目录下使用 `lupdate` 指令即可生成 `.ts` 文件, 当然也可以用 IDE 提供的快捷操作点击生成. 当生成 `.ts` 文件时, Qt linguist 将检查所有可被 `#include` 到的 `.cpp`, `.h`, `.ui` 文件 (注意, 不包括 `.ui` 文件生成的 `ui_xxx.h` 文件). 这些文件中若出现如下方法调用即被加入 `.ts` 文件中:

- `QCoreApplication::translate("上下文", "原文")`
- `QCoreApplication::trUtf8("上下文", "原文")`
- `QObject::tr("原文")`
- `QObject派生类::tr("原文")`

这些方法具体的实现与参数的意思在下文中会说到, 它们是用于获取翻译文本的. 当 Qt linguist 在 `lupdate` 操作里扫描到某处有获取译文的代码段时, 就会自动将其参数设置为对应的 `.ts` 文件中的标签中. 这里要注意的是, **使用这些方法时不可出现宏, 变量名, 方法调用等任何间接设置参数的方式**. 这是因为 Qt linguist 只是单纯的文本扫描, 它不会去理解你的代码. 因此若其发现参数列表中出现宏或者变量名等形式时, 将判定这段代码不是用于获取译文的.

[<返回顶部>](#返回顶部)

<div id="语言翻译器QTranslator"></div>

### 语言翻译器 `QTranslator`

`QTranslator` 核心工作就是从 `.qm` 文件内加载数据并保存, 以及在 `QCoreApplication` 需要的时候返回其所需的翻译文本. 我在这里想重点讨论其内部的数据结构, 这是整个系统里最让我耳目一新的地方.

从表面上来看, `QTranslator` 在头文件里的定义如下 : 

```cpp
class Q_CORE_EXPORT QTranslator : public QObject
{
    Q_OBJECT
public:
    explicit QTranslator(QObject *parent = nullptr);
    ~QTranslator();

    virtual QString translate(const char *context, const char *sourceText,
                              const char *disambiguation = nullptr, int n = -1) const;

    virtual bool isEmpty() const;

    bool load(const QString & filename,
              const QString & directory = QString(),
              const QString & search_delimiters = QString(),
              const QString & suffix = QString());
    bool load(const QLocale & locale,
              const QString & filename,
              const QString & prefix = QString(),
              const QString & directory = QString(),
              const QString & suffix = QString());
    bool load(const uchar *data, int len, const QString &directory = QString());

private:
    Q_DISABLE_COPY(QTranslator)
    Q_DECLARE_PRIVATE(QTranslator)
};
```

清晰明了, 提供了几个重载的 `load` 方法用于加载文件, 提供 `translate` 方法用于获取译文, 还有 `isEmpty` 方法用于判空. 这里简单介绍下第一种 `load` 方法, 先来看它的具体实现:

```cpp
bool QTranslator::load(const QString & filename, const QString & directory,
                       const QString & search_delimiters,
                       const QString & suffix)
{
    Q_D(QTranslator);
    d->clear();  // d 是私有类对象指针
    /* 确定前缀, 即目录部分 */
    QString prefix;
    if (QFileInfo(filename).isRelative()) {
        prefix = directory;
        if (prefix.length() && !prefix.endsWith(QLatin1Char('/')))
            prefix += QLatin1Char('/');
    }
    /* 确定后缀, 分隔符等 */
    const QString suffixOrDotQM = suffix.isNull() ? dotQmLiteral() : suffix;
    QStringRef fname(&filename);
    QString realname;
    const QString delims = search_delimiters.isNull() ? QStringLiteral("_.") : search_delimiters;
    /* 确定文件名, 注意这里 */
    for (;;) {
        QFileInfo fi;

        realname = prefix + fname + suffixOrDotQM;  // 检查 前缀+文件名+后缀
        fi.setFile(realname);
        if (fi.isReadable() && fi.isFile())
            break;

        realname = prefix + fname;  // 检查 前缀+文件名
        fi.setFile(realname);
        if (fi.isReadable() && fi.isFile())
            break;

        int rightmost = 0;
        for (int i = 0; i < (int)delims.length(); i++) {
            int k = fname.lastIndexOf(delims[i]);  // 定位到最后一个分隔符
            if (k > rightmost)
                rightmost = k;
        }

        // no truncations? fail
        if (rightmost == 0)  // 再也无法分割后判定加载失败
            return false;

        fname.truncate(rightmost);  // 根据上文的定位分割文件名
    }

    // realname is now the fully qualified name of a readable file.
    return d->do_load(realname, directory);  // 最终加载操作是私有类对象实现的
}
```

根据实现部分可以看出来, 具体的加载操作是调用了 `d` 指针的 `do_load` 方法. 而 `load` 方法则是用于确定一个可被载入的文件, 并且它不是简单地尝试载入参数所指定的文件. 实际上 `load` 方法将尝试检查下面这几个文件是否存在, 若有则加载:

- 目录 + 文件名 + 后缀
- 目录 + 文件名
- 目录 + 文件名 - 文件名当前最后一个分隔符及之后的内容 + 后缀
- 目录 + 文件名 - 文件名当前最后一个分隔符及之后的内容
- ......

最终只有当文件名再也无法分割还是找不到文件后, 才会最终判定加载失败.

我们发现 `QTranslator` 内部并没有实际加载文件的操作, 实际上也没有实际获取文本的操作, 其实 `QTranslator::translate` 方法内部也是调用了 `d` 指针的 `do_translate` 方法. `QTranslator` 甚至也没有保存数据的成员.
实际上这里的 `d` 指针是 `QTranslator` 的私有类 `QTranslatorPrivate` 对象的指针. 而我们所要找的这些方法与数据结构也是在私有类中.

> Qt 的源码中大量使用了私有类, 以保证最大程度上只暴露仅供用户使用的方法. 当翻阅源码时若你发现了不知道哪儿来的 `d` 指针或者 `d_func` 方法调用, 那就表明了接下来是要对对应的私有类对象进行操作, 你可以不用再去到处找它们的声明位置了.

[<返回顶部>](#返回顶部)

<div id="语言翻译器的内部数据结构"></div>

#### 语言翻译器的内部数据结构

在我的预想中私有类应该会维护一个 `QMap` 或者 `QHash` 之类的结构来保存数据, 但事实并非如此. `QTranslatorPrivate` 的声明部分位于 `qtranslator.cpp` 中, 如下:

```cpp
class QTranslatorPrivate : public QObjectPrivate
{
    Q_DECLARE_PUBLIC(QTranslator)
public:  /* 注意这个枚举 */
    enum { Contexts = 0x2f, Hashes = 0x42, Messages = 0x69, NumerusRules = 0x88, Dependencies = 0x96 };

    QTranslatorPrivate() :
#if defined(QT_USE_MMAP)
          used_mmap(0),
#endif
          unmapPointer(0), unmapLength(0), resource(0),
          messageArray(0), offsetArray(0), contextArray(0), numerusRulesArray(0),
          messageLength(0), offsetLength(0), contextLength(0), numerusRulesLength(0) {}

#if defined(QT_USE_MMAP)
    bool used_mmap : 1;
#endif
    char *unmapPointer;     // used memory (mmap, new or resource file)
    qsizetype unmapLength;

    // The resource object in case we loaded the translations from a resource
    QResource *resource;

    // used if the translator has dependencies
    QList<QTranslator*> subTranslators;

    // Pointers and offsets into unmapPointer[unmapLength] array, or user
    // provided data array
    const uchar *messageArray;      // 文本数据
    const uchar *offsetArray;       // 偏移量
    const uchar *contextArray;      // 上下文数据
    const uchar *numerusRulesArray; // 歧义文本数据
    uint messageLength;             // 文本数据总长
    uint offsetLength;              // 偏移量总长
    uint contextLength;             // 上下文数据总长
    uint numerusRulesLength;        // 歧义文本数据总长
    /* 上文中出现的具体 加载 / 获取译文操作 , 注意第二个 do_load */
    bool do_load(const QString &filename, const QString &directory);
    bool do_load(const uchar *data, qsizetype len, const QString &directory);
    QString do_translate(const char *context, const char *sourceText, const char *comment,
                         int n) const;
    void clear();
};
```

注意中文注释部分, 这里直接使用了字节数组来保存所有数据, 而每个数据的位置则是通过偏移量来确定的, 最后单独记录了这些数组的总长. 那么重点就在于这个偏移量是怎么计算的. 注意这里类中定义的枚举, 唯一能与偏移量相对的就只有 `Hashes`, 实际上我在 `translator.cpp` 中也发现了几个哈希运算相关的方法. 这里猜测这个偏移量应该是哈希结果. 

> 上文中猜测 `.ts` 到 `.qm` 的过程不只有二进制化一个步骤的理由之一就源自这里.

结合这个枚举与第二个 `do_load` 方法, 可以猜测这里的枚举则是文件内容的标签, 那么这里加载文件的最终操作是应该是对整个文件数据逐字节的读取, 根据标签判断接下来读取的内容. 不过扯了这么多都还是猜测, 下面我们直接看 `do_load` 和 `do_translate` 验证一下.

[<返回顶部>](#返回顶部)

<div id="语言翻译器的加载文件操作"></div>

#### 语言翻译器的加载文件操作 --- `do_load`

`QTranslatorPrivate::do_load` 有两个重载方法, 结合 `QTranslator` 的 `load` 方法不难看出, 这里的重载一个是打开文件并载入数据, 另一个是解析文件数据并初始化私有类对象成员. 实时也确实如此, 那么我们直接来看第二个重载方法 :

```cpp
bool QTranslatorPrivate::do_load(const uchar *data, qsizetype len, const QString &directory)
{
    bool ok = true;
    const uchar *end = data + len;

    data += MagicLength;

    QStringList dependencies;
    while (data < end - 5) {
        quint8 tag = read8(data++);       // 读第 1 个字节, 是为标签
        quint32 blockLen = read32(data);  // 读接下来的 4 个字节, 是为数据长度
        data += 4;
        if (!tag || !blockLen)
            break;
        if (quint32(end - data) < blockLen) {
            ok = false;
            break;
        }
        /* 根据标签确定数据类型, 将本段数据开头初始化为数据数组成员, 根据长度初始化数据总长度成员 */
        if (tag == QTranslatorPrivate::Contexts) {  // 上下文数据组
            contextArray = data;
            contextLength = blockLen;
        } else if (tag == QTranslatorPrivate::Hashes) {  // 偏移量数据组. 哈希值对应着偏移量, 猜测正确✔
            offsetArray = data;
            offsetLength = blockLen;
        } else if (tag == QTranslatorPrivate::Messages) {  // 译文数据组
            messageArray = data;
            messageLength = blockLen;
        } else if (tag == QTranslatorPrivate::NumerusRules) {  // 歧义译文数据组
            numerusRulesArray = data;
            numerusRulesLength = blockLen;
        } else if (tag == QTranslatorPrivate::Dependencies) {  // 附属翻译文本数据组
            QDataStream stream(QByteArray::fromRawData((const char*)data, blockLen));
            QString dep;
            while (!stream.atEnd()) {
                stream >> dep;
                dependencies.append(dep);
            }
        }

        data += blockLen;  // 移动指针到下一组数据的位置
    }
    /* 检查多套翻译数据的内容是否合法 */
    if (ok && !isValidNumerusRules(numerusRulesArray, numerusRulesLength))
        ok = false;
    if (ok) {  /* 初始化附属翻译器 (一个翻译器对应一个文本, 这里实际上是支持载入多个文本, 下文会细说) */
        const int dependenciesCount = dependencies.count();
        subTranslators.reserve(dependenciesCount);
        for (int i = 0 ; i < dependenciesCount; ++i) {
            QTranslator *translator = new QTranslator;
            subTranslators.append(translator);
            ok = translator->load(dependencies.at(i), directory);
            if (!ok)
                break;
        }

        // In case some dependencies fail to load, unload all the other ones too.
        if (!ok) {
            qDeleteAll(subTranslators);
            subTranslators.clear();
        }
    }

    if (!ok) {
        messageArray = 0;
        contextArray = 0;
        offsetArray = 0;
        numerusRulesArray = 0;
        messageLength = 0;
        contextLength = 0;
        offsetLength = 0;
        numerusRulesLength = 0;
    }

    return ok;
}
```

注意中文注释部分, 看来确实如我们所猜测一般. 首先偏移量确实是通过哈希算法得来, 而且哈希运算应该是在 `.ts` 文件转换为 `.qm` 文件的过程中进行的. 其次这里最终的加载操作的确是按字节读取, 根据先读入的字节数据确定后续字节数据的数据信息. 由此我们也可以猜测出 `.qm` 文件的数据基本格式如下:

|数据分组|数据长度|数据名称|数据内容|
|-|-|-|-|
|第一组|1 字节|tag|数据标签|
|第一组|4 字节|blockLen|数据长度|
|第一组|blockLen 字节|data|具体的数据信息|
|第二组|...|...|...|

> 这个读取数据的处理方式有点像处理 TCP 流数据的所谓 "粘包" 现象, 根据约定先读几个字节来确定后面数据的信息, 再读对应长度的字节数获得具体数据. 当然事实上 "粘包" 本身就是个伪命题, 不过那就是另一个话题了.

[<返回顶部>](#返回顶部)

<div id="语言翻译器的获取翻译文本"></div>

#### 语言翻译器的获取译文操作 --- `do_translate`

现在我们知道了具体的数据存入的方式, 那么读取的方法 `do_translate` 内部的实现我们也可以猜个八九不离十了, 直接上源码:

```cpp
QString QTranslatorPrivate::do_translate(const char *context, const char *sourceText,
                                         const char *comment, int n) const
{
    if (context == 0)
        context = "";
    if (sourceText == 0)
        sourceText = "";
    if (comment == 0)
        comment = "";

    uint numerus = 0;
    size_t numItems = 0;

    if (!offsetLength)
        goto searchDependencies;
    /* 这里检查该译文是否属于此翻译器, 也涉及到翻译器的附属关系, 下文会细说. */
    /*
        Check if the context belongs to this QTranslator. If many
        translators are installed, this step is necessary.
    */
    if (contextLength) {
        quint16 hTableSize = read16(contextArray);
        uint g = elfHash(context) % hTableSize;
        const uchar *c = contextArray + 2 + (g << 1);
        quint16 off = read16(c);
        c += 2;
        if (off == 0)
            return QString();
        c = contextArray + (2 + (hTableSize << 1) + (off << 1));

        const uint contextLen = uint(strlen(context));
        for (;;) {
            quint8 len = read8(c++);
            if (len == 0)
                return QString();
            if (match(c, len, context, contextLen))
                break;
            c += len;
        }
    }
    /* 检查该翻译器有没有保存文本 */
    numItems = offsetLength / (2 * sizeof(quint32));
    if (!numItems)
        goto searchDependencies;  // to Qt : goto should goto hell (delete
    /* 此处涉及到歧义消除与复数处理, 这里的 n 是文本中占位符 %n 的数值, 下文细说. */
    if (n >= 0)
        numerus = numerusHelper(n, numerusRulesArray, numerusRulesLength);
    /* 获得译文 */
    for (;;) {
        quint32 h = 0;  // 先计算得出哈希值
        elfHash_continue(sourceText, h);
        elfHash_continue(comment, h);
        elfHash_finish(h);
        /* 获取译文位置的偏移量 */
        const uchar *start = offsetArray;
        const uchar *end = start + ((numItems-1) << 3);
        while (start <= end) {  // 类似二分查找, 哈希值对应时得到对应的偏移量
            const uchar *middle = start + (((end - start) >> 4) << 3);
            uint hash = read32(middle);  // 哈希值也是4字节存储
            if (h == hash) {  // 找到则退出循环, 此时 (start <= end) === true
                start = middle;
                break;
            } else if (hash < h) {
                start = middle + 8;
            } else {
                end = middle - 8;
            }
        }
        /* 找到了就尝试获取译文文本 */
        if (start <= end) {
            // go back on equal key
            while (start != offsetArray && read32(start) == read32(start-8))
                start -= 8;

            while (start < offsetArray + offsetLength) {
                quint32 rh = read32(start);
                start += 4;
                if (rh != h)  // 为啥又校验一次哈希值?
                    break;
                quint32 ro = read32(start);  // 哈希值后面四个字节是实际的偏移量
                start += 4;
                QString tn = getMessage(messageArray + ro, messageArray + messageLength, context,
                                        sourceText, comment, numerus);  // 这里获取文本, 到这一步该有的参数都有了
                if (!tn.isNull())
                    return tn;
            }
        }
        if (!comment[0])
            break;
        comment = "";
    }
/* 想必这里就是 other translator of "many translators" witch been installed */
searchDependencies:
    for (QTranslator *translator : subTranslators) {
        QString tn = translator->translate(context, sourceText, comment, n);
        if (!tn.isNull())
            return tn;
    }
    return QString();
}
```

这里最终调用 `getMessage` 来获取文本, `getMessage` 里面再没有什么秘密了, 它直接利用这里得到的参数从 `messageArray` 中得到我们需要的数据.

在这几个小节里, 你可能会注意到私有类中还有成员 `subTranslators`, 并且 `do_load` 方法和 `do_translate` 方法里也有对其的相关的操作, 还有一些不知所云的变量如 `QTranslator::translate` 中的 `comment` 与 `n` 参数. 同时私有类中的 `numerusRulesArray` 具体用途也没有谈到. 我在源码中有给出一些注释简单带过, 下面就来细说这些东西的具体用途.

[<返回顶部>](#返回顶部)

<div id="语言翻译器之间的附属关系"></div>

#### 语言翻译器之间的附属关系

上文中说到 `QTranslator` 重载了多种 `load` 方法. 其中一个版本允许我们一次加载多个 `.qm` 文件数据, 实现如下:

```cpp
bool QTranslator::load(const uchar *data, int len, const QString &directory)
{
    Q_D(QTranslator);
    d->clear();

    if (!data || len < MagicLength || memcmp(data, magic, MagicLength))
        return false;
    /* directory 是所有要加载的 .qm 文件的目录 */
    return d->do_load(data, len, directory);
}
```

可见, 该方法并不是加载 `.qm` 文件, 而是直接加载数据. 利用这个方法, 我们可以一次性将多个 `.qm` 文件数据合并再加载进同一个 `QTranslator` 中. **当且仅当** 这种情况下, 才会使用到私有类中 `subTranslators` 成员及其相关的操作. 若使用另外两个 `load` 方法以一个 `QTranslator` 对象读取一个 `.qm` 文件的形式进行加载, 则不会出现 `QTranslator` 之间的附属关系.

> 至于这种特殊加载方式的用处, 我可以想到这么一种情况: 若一个应用程序过于庞大, 用同一个 `.ts` 文件难以管理同一种语言的译文. 这时便可以将同种语言的译文拆分进多个 `.ts` 文件中管理. 同理我们最后也会得到多个 `.qm` 文件. 此时若希望这些文件里的数据都能以最高优先级被读取, 则需要使用最后一种 `load` 重载方法一次性将多个 `.qm` 文件内的数据读取. 

这样一来, 在 `QCoreApplication` 中, 我们实际上只装载了一个翻译器, 因此受此翻译器所直接或间接管理的文本都拥有最高的被读取优先级. 而实际上该翻译器内部还是遵循着一个 `.qm` 文件对应着用一个翻译器进行管理, 只不过最终译文都通过装载进 `QCoreApplication` 的那个翻译器返回到外部.

**注意:** 使用这种方法的时候, 需要保证所加载的 `.qm` 文件都处于同一目录下.

[<返回顶部>](#返回顶部)

<div id="歧义消除与复数处理"></div>

#### 歧义消除与复数处理

歧义消除与复数处理是 Qt 多语言机制为翻译工作额外提供的功能. 关于这两个功能要注意的地方主要是使用方面, 因此这里不会说太多, 在下文有关翻译器使用的部分会详细说明.

私有类中的 `numerusRulesArray` 成员实际上存放着某些翻译文本的多个版本译文, 这些多个版本的译文主要用于解决前言中提到的 **歧义消除** 问题. 在 Qt 的设想下, 每一个应用程序应该默认提供英文的文本, 并以英文版为基础翻译出各个其他语言的版本. 此时便会遇到英文版的词语在其他语言的不同语境中出现多种不同的翻译版本的问题. 因此 Qt 允许在设置与获得译文时可额外指定一个字符串用于消除歧义(实际上就是标注一下它们是不同的字符串罢了), 而这个用于消除歧义的额外字符串正是 `QTranslator::translate` 中的 `comment` 参数. 

> 但正如我所言, 这个解决方案是解决了在 Qt 设想下才出现的问题. 实际上若我们把应用程序的每一种语言文本都视作翻译文本, 默认文本仅仅只是文本的标识. 那么从根源上就不会出现这种问题. 下文会再提到这点.

至于 Qt 官方所谓的"复数处理", 我们把它理解为一个基本的格式化字符串功能即可. 当我们设置原文的时候, 可以加入一个特殊的占位符 `%n`, 该占位符的内容可以在获取译文时动态地插入进去. 插入的方式便是通过设置 `QTranslator::translate` 的 `n` 参数. 而 `n` 参数是一个 `int` 型变量, 也就是说这个特殊的占位符我们只能设置数字. 这就是 Qt 所谓的"复数处理". 

> 想必你也发现这个功能有多鸡肋了, 毕竟直接设置普通的占位符再动态地插入文本就可以完全覆盖这个功能, 并且内容也不仅限于 `int` 型. 我一开始还以为这个 "复数处理" 是处理复数(complex), 没想到是处理复数(plural) (瞧, 又是一个待消除的歧义). 对于这个功能我能想到潜在的用处只有两点:
> - 使用这种方式可以避免字符串拼接时的拷贝构造等操作. 如果这类需要处理复数的文本将被大量使用, 这种方式或许会有效率上的优势(前提是你将大量地使用, 足够的大量).
> - 或许部分工程不允许大规模使用 `QString` 或是 `std::string` 等字符串数据格式(极低性能的嵌入式平台等), 也就无法使用其便捷的拼接操作, 此时只能使用这个方式勉强进行字符串拼接.

[<返回顶部>](#返回顶部)

<div id="翻译器的加载与管理"></div>

### 翻译器的加载与管理 --- `QCoreApplication`

Qt 原生多语言机制的底层原理其实通过看 `QTranslator` 的源码实现就可以了解的差不多了. 因此从本小节开始, 我将更着眼于使用方式上的介绍, 以及一些功能重叠的方法的选择推荐.

虽然上一节花了很大篇幅讨论 `QTranslator` 的实现细节, 但具体到使用多语言机制时, 我们用到 `QTranslator` 的地方通常只有装载翻译器这个操作. 当我们装载翻译器之后, 就可以直接通过 `QCoreAppliaction` 获取需要的译文了. 这里我从 `QCoreAppliaction` 的定义中截取出与多语言机制有关的片段:

```cpp
class Q_CORE_EXPORT QCoreApplication
#ifndef QT_NO_QOBJECT
    : public QObject
#endif
{
......
public:
......
#ifndef QT_NO_TRANSLATION
    static bool installTranslator(QTranslator * messageFile);
    static bool removeTranslator(QTranslator * messageFile);
#endif
    
    static QString translate(const char * context,
                             const char * key,
                             const char * disambiguation = nullptr,
                             int n = -1);
......
public: \
    static inline QString tr(const char *sourceText, const char *disambiguation = nullptr, int n = -1) \
        { return QCoreApplication::translate(#context, sourceText, disambiguation, n); } \
    QT_DECLARE_DEPRECATED_TR_FUNCTIONS(context) \
......
};
```

这里的 `installTranslator` 与 `removeTranslator` 显然是用于装载和卸载翻译器的. 而 `translate` 用于获取译文, `tr` 方法内部也是通过调用 `translate` 实现功能. 这里的 `QT_DECLARE_DEPRECATED_TR_FUNCTIONS` 宏用于定义一个 `trUtf8` 方法, 本质上也是调用 `translate`. 三种翻译方法参数列表完全相同. 

在这几个对翻译器的操作之外, 我并没有在 `QCoreApplication` 的声明中看到保存翻译器的成员, 想必这又是被放在私有类中了. `QCoreApplication` 对应的私有类名为 `QCoreApplicationPrivate`. 我截取出其声明中有关翻译机制的片段:

```cpp
typedef QList<QTranslator*> QTranslatorList;
......
class Q_CORE_EXPORT QCoreApplicationPrivate
#ifndef QT_NO_QOBJECT
    : public QObjectPrivate
#endif
{
......
public:
......
#ifndef QT_NO_TRANSLATION
    QTranslatorList translators;
    QReadWriteLock translateMutex;
    static bool isTranslatorInstalled(QTranslator *translator);
#endif
......
};
```

可见, 其内部使用了一个 `QList` 来保存翻译器. 至此, `QCoreApplication` 所有有关翻译机制的方法与成员都被找到了, 下面我们来看实现.

[<返回顶部>](#返回顶部)

<div id="装载翻译器"></div>

#### 装载翻译器 --- `QCoreApplication::installTranslator`

当我们要装载某个翻译器到 `QCoreApplication` 时, 请保证该翻译器已经正确读取了某个 `.qm` 文件. 详细细节来看 `QCoreApplication::installTranslator` 实现部分:

```cpp
bool QCoreApplication::installTranslator(QTranslator *translationFile)
{
    if (!translationFile)
        return false;

    if (!QCoreApplicationPrivate::checkInstance("installTranslator"))
        return false;
    QCoreApplicationPrivate *d = self->d_func();  // self 指针是 QCoreApplication 单例类的实例
    {
        QWriteLocker locker(&d->translateMutex);
        d->translators.prepend(translationFile);  // QList::prepend 方法将 item 添加到列表的头部
    }

#ifndef QT_NO_TRANSLATION_BUILDER
    if (translationFile->isEmpty())  // 判空, 当翻译器没有加载数据时装载错误, 但这里没有将其卸载
        return false;
#endif

#ifndef QT_NO_QOBJECT
    QEvent ev(QEvent::LanguageChange);  // 发送 QEvent::LanguageChange 事件
    QCoreApplication::sendEvent(self, &ev);
#endif

    return true;
}
```

当我们尝试装载一个 `QTranslator::isEmpty` 返回 `true` 的翻译器, 此时将不会发出 `QEvent::LanguageChange` 事件. 但是 `QCoreApplication` 也不会主动把这个无效的翻译器移除 `d->translators` 中. 因此正确的使用方式是先让翻译器加载完 `,qm` 文件, 再将其装载进 `QCoreApplication`. 

> 这里出现了一个 `self` 指针, 这是指向 `QCoreApplication` 单例的指针. 就像 `d` 指针往往用于指代某个类对应的私有类一样, 在 Qt 源码中往往使用 `self` 指针来指向某个单例类的实例.

[<返回顶部>](#返回顶部)

<div id="卸载翻译器"></div>

#### 卸载翻译器 --- `QCoreApplication::removeTranslator`

与 `installTranslator` 方法一样, `removeTranslator` 方法也需要指定目标翻译器的指针, 其实现如下:

```cpp
bool QCoreApplication::removeTranslator(QTranslator *translationFile)
{
    if (!translationFile)
        return false;
    if (!QCoreApplicationPrivate::checkInstance("removeTranslator"))
        return false;
    QCoreApplicationPrivate *d = self->d_func();
    QWriteLocker locker(&d->translateMutex);
    if (d->translators.removeAll(translationFile)) {  // 移除翻译器
#ifndef QT_NO_QOBJECT
        locker.unlock();
        if (!self->closingDown()) {
            QEvent ev(QEvent::LanguageChange);  // 发送 QEvent::LanguageChange 事件
            QCoreApplication::sendEvent(self, &ev);
        }
#endif
        return true;
    }
    return false;
}
```

这里直接将翻译器从列表中移除, 唯一值得注意的地方就是当此处移除翻译器成功时还会发出一次 `QEvent::LanguageChange` 事件.

> 不得不说这个方法相当鸡肋, 既然我装载的时候已经把翻译器交给 `QCoreApplication` 管理了, 卸载的时候却还需要我持有这个翻译器的指针? 如果我持有这个指针又何必从你这里获取译文呢? 😂 或许是因为单例可以被全局访问到吧.

[<返回顶部>](#返回顶部)

<div id="获取最新的翻译文本"></div>

### 获取最新的翻译文本 --- `QCoreApplication::translate` 与 `QObject::tr`

上一节里说到, `QCoreApplication` 中获取译文的方法有三种 :  `translate`, `tr` 与 `trUtf8`. 但这三种最终都是通过 `translate` 来获取译文. 与此之外, `QObject` 类也提供了方法 `tr` 用于获取译文. 下面来逐个解析:

[<返回顶部>](#返回顶部)

<div id="获取译文"></div>

#### 获取译文 --- `QCoreApplication::translate`

`QCoreApplication::tr`, 与 `QCoreApplication::trUtf8` 就不多说了, 我们直接来看 `QCoreApplication::translate`. 由于这个方法将会在业务代码中大量地使用, 因此这里介绍一下它的参数:

- `const char *context` : 上下文, 通常对应着类名.
- `const char *sourceText` : 源文本.
- `const char *disambiguation` : 歧义消除文本, 当同上下文内出现一段源文对应多个译文时用到. 缺省值 `nullptr`.
- `int n` : 复数值, 当源文本中出现了 `%n` 占位符, 则再此指定具体要用到的复数. 缺省值 `-1`. 

下面来看实现部分:

```cpp
QString QCoreApplication::translate(const char *context, const char *sourceText,
                                    const char *disambiguation, int n)
{
    QString result;

    if (!sourceText)
        return result;

    if (self) {
        QCoreApplicationPrivate *d = self->d_func();
        QReadLocker locker(&d->translateMutex);
        if (!d->translators.isEmpty()) {
            QList<QTranslator*>::ConstIterator it;
            QTranslator *translationFile;
            /* 从前到后, 因此最后一个装载的翻译器有最高优先级 */
            for (it = d->translators.constBegin(); it != d->translators.constEnd(); ++it) {
                translationFile = *it;
                /* 底层翻译直接调用 QTranslator::translate */
                result = translationFile->translate(context, sourceText, disambiguation, n);
                if (!result.isNull())
                    break;
            }
        }
    }

    if (result.isNull())
        result = QString::fromUtf8(sourceText);  // 若没有对应的译文则返回其 UTF-8 编码值

    replacePercentN(&result, n);  // 替换复数值到占位符 %n 处
    return result;
}
```

看过上面的文章后, 再看这段代码就没有什么不理解的地方了, 这里直接按照优先级顺序通过调用 `QTranslator::translate` 来检查每个翻译器里是否有对应的译文.

[<返回顶部>](#返回顶部)

<div id="QObject::tr与QCoreApplication::translate的关系"></div>

#### `QObject::tr` 与 `QCoreApplication::translate` 的关系

上面说到了除了 `QCoreApplication::translate` 可以获取译文外, `QObject::tr` 也可以用于获取译文. 那么它们二者有什么关系呢? 我们凭直觉猜测后者应该也是最终调用了前者, 事实也确实如此. 而我之所以要特地花一小节来说这件事, 是因为 `QObject::tr` 的源码部分是比较容易误导人的.

如果你曾尝试过使用 `QObject::tr` 或者 `QObject派生类::tr` 进行多语言适配, 你会发现这个方法是不需要设置上下文的. 而当你打开生成的 `.ts` 文件时又会发现 Qt 自动帮你将这段原文的上下文设置为了 `QObject` 或 `QObject派生类`. 对于这个问题我也会在这一小节里解答.

当我们直接打开 `QObject` 头文件中的声明部分可以看见这么一段代码:

```cpp
class Q_CORE_EXPORT QObject
{
......
#if defined(QT_NO_TRANSLATION) || defined(Q_CLANG_QDOC)
    static QString tr(const char *sourceText, const char * = nullptr, int = -1)
        { return QString::fromUtf8(sourceText); }  // 注意这里
#if QT_DEPRECATED_SINCE(5, 0)
    QT_DEPRECATED static QString trUtf8(const char *sourceText, const char * = nullptr, int = -1)
        { return QString::fromUtf8(sourceText); }  // 注意这里
#endif
#endif //QT_NO_TRANSLATION
......
```

如果你尝试过通过 IDE 来查找 `QObject::tr` 的声明位置, 最后也会跳转到这里. 注意中文注释处, 这里的实现部分直接返回了原文的 UTF-8 编码. 但我们在实际使用中却的确可以通过这个方法获得译文, 这就是容易让人误会的地方.

你可能有注意到源码中我截取出来的两个宏 `QT_NO_TRANSLATION` 与 `Q_CLANG_QDOC`, 如果你了解后者的意义, 那么也就明白了这里发生了什么. `Q_CLANG_QDOC` 宏实际上是 Qt 用于将其源代码生成文档的一个宏, 这个宏永远不会被 `#define`. 所有被 `#if defined(Q_CLANG_QDOC)` 包裹住的代码段, 都将被 Qt 用脚本自动化地生成文档.

> 不得不说这个宏骗了我好久, 以至于我一直以为 Qt 有什么神奇的机制可以自动化为我的控件设置译文. 😓

OK, 那么现在我们已经知道了 `QObject` 源码中的 `tr` 方法是假的, 哪真正的 `tr` 方法又在哪呢? 其实就藏在我们经常用到但大多数人都不知所云的 `Q_OBJECT` 宏里. 这个宏被定义在 `qobjectdefs.h` 中, 我这里截取出相关片段:

```cpp
// qobjectdefs.h
......

#ifndef QT_NO_TRANSLATION
// full set of tr functions
#  define QT_TR_FUNCTIONS \  // 注意这个宏, 实现了 tr 方法
    static inline QString tr(const char *s, const char *c = nullptr, int n = -1) \
        { return staticMetaObject.tr(s, c, n); } \  // 内部调用了 staticMetaObject.tr
    QT_DEPRECATED static inline QString trUtf8(const char *s, const char *c = nullptr, int n = -1) \
        { return staticMetaObject.tr(s, c, n); }
#else
// inherit the ones from QObject
# define QT_TR_FUNCTIONS
#endif

......

#define Q_OBJECT \
public: \
    QT_WARNING_PUSH \
    Q_OBJECT_NO_OVERRIDE_WARNING \
    static const QMetaObject staticMetaObject; \  // 注意这个 staticMetaObject
    virtual const QMetaObject *metaObject() const; \
    virtual void *qt_metacast(const char *); \
    virtual int qt_metacall(QMetaObject::Call, int, void **); \
    QT_TR_FUNCTIONS \  // 注意这里, Q_OBJECT 宏包含了 QT_TR_FUNCTIONS宏
private: \
    Q_OBJECT_NO_ATTRIBUTES_WARNING \
    Q_DECL_HIDDEN_STATIC_METACALL static void qt_static_metacall(QObject *, QMetaObject::Call, int, void **); \
    QT_WARNING_POP \
    struct QPrivateSignal {}; \
    QT_ANNOTATE_CLASS(qt_qobject, "")

......
```

可见, 具体的 `tr` 方法是通过 `QT_TR_FUNCTIONS` 宏实现的, 而这个宏又被 `Q_OBJECT` 宏包含. 这里才是实现 `tr` 方法的地方. 但是事情还没结束, 我们可以看到实现部分是通过调用了 `staticMetaObject.tr` 这个方法的来的. 这个变量是 `Q_OBJECT` 宏所设置的一个 `static const QMetaObject` 成员. 想不到我们凭直觉猜测的 `QObject::tr` 和 `QCoreApplication::translate` 两个方法之间的关系, 具体实现中居然还牵扯到了 Qt 的元对象系统.

Qt 的元对象系统是一个很庞大复杂的机制, 在这里我们需要知道元对象系统可以提供类似 "反射" 的功能. 在下文中会再提到这一点.

不管怎么样, 我们还是得追查到底, 下面直接来看 `QMetaObject` 的源码, 我直接贴出相关的部分:

```cpp
// qobjectdefs.h  // 声明部分还是在 qobjectdefs.h 中
struct Q_CORE_EXPORT QMetaObject
{
......
#if !defined(QT_NO_TRANSLATION) || defined(Q_CLANG_QDOC)
    QString tr(const char *s, const char *c, int n = -1) const;
#endif // QT_NO_TRANSLATION
......
};

// qmetaobject.cpp  // 实现部分在 qmetaobject.cpp 中
......
#ifndef QT_NO_TRANSLATION
/*!
    \internal
*/
QString QMetaObject::tr(const char *s, const char *c, int n) const
{   /* 注意此处的第一个参数 */
    return QCoreApplication::translate(objectClassName(this), s, c, n);
}
#endif // QT_NO_TRANSLATION
......
```

终于水落石出了, `QObject::tr` 实际上是调用了 `QMetaObject::tr` , 而 `QMetaObject::tr` 最后还是调用了 `QCoreApplication::translate`. 我们的猜测最终得到了证实.

至于本小节开头的那个问题, `QObject::tr` 方法为什么不需要指定上下文? 而 Qt 为什么又自动帮我们在 `.ts` 文件中将上下文设置为 `QObject`? 相信你看到这已经有了答案. 这个答案同时也解释了 `QObject::tr` 到 `QCoreApplication::translate` 的过程中为什么要牵扯到元对象系统. 这里实际上是利用了元对象系统的 "反射" 机制获得到调用方法的对象的类名. 也就是代码中的 `objectClassName` 方法. 所有当我们在代码中用 `QObjcet::tr` 的形式获取译文时, `.ts` 文件中将自动指定上下文为 `QObject`; 而我们在 `QObject` 的派生类中调用 `tr` 方法, `.ts` 文件中将自动指定上下文为 `QObject派生类`.

> 个人认为 Qt 库最精彩的部分就是它的元对象系统. 这里先挖个坑, 以后尝试分析 Qt 的元对象系统源码.

[<返回顶部>](#返回顶部)

<div id="更新UI控件的时机"></div>

### 更新 UI 控件的时机 --- `QEvent::LanguageChange` 事件

在讲解 `QCoreApplication` 时, 我们知道了当 `QTranslator` 翻译器在 `QCoreApplication` 中被正确地装载 / 卸载时, `QCoreApplication` 将利用 `sendEvent` 方法发出 `QEvent::LanguageChange` 事件. 这个操作涉及到了 Qt 的事件系统.

Qt 的事件系统简单来说就是为了跨平台而将各个操作系统自身的事件系统进行封装而得来的一个中间层. 任何一个 `QObject` 及其子类对象都可以通过重载 `event` 等方法来处理由 `QCoreApplication` 单例发来的事件. 不过要记得将你所有不想亲自处理的事件交由基类的同名方法进行处理.

[<返回顶部>](#返回顶部)

<div id="LanguageChange事件发给了谁"></div>

#### `QEvent::LanguageChange` 事件发给了谁?

上文中我有提到, `QEvent::LanguageChange` 事件最终会发给所有属于 `top-level widgets` 范畴内的控件. 而这一小节的目的在于要解开一个看源码时你可能会面临的一个疑惑.

如果你仔细看了上文中的源码, 你会发现当 `QCoreApplication` 发出 `QEvent::LanguageChange` 事件的地方的代码是这样的:

```cpp
......
QEvent ev(QEvent::LanguageChange);
QCoreApplication::sendEvent(self, &ev);
......
```

可见 `QCoreApplication` 将 `QEvent::LanguageChange` 事件发送给了 `self` 指针, 也就是它自身的单实例. 于是我们接着来看它处理事件的地方, 也就是 `QCoreApplication::event` 方法:

```cpp
bool QCoreApplication::event(QEvent *e)
{
    if (e->type() == QEvent::Quit) {
        quit();
        return true;
    }
    return QObject::event(e);
}
```

可见它并没有处理 `QEvent::LanguageChange` 事件, 而是将其交给了 `QObject::event` 进行处理. 这里我没有贴出 `QObject::event` 的实现部分, 但我可以明确地告诉你这个方法里也没有对 `QEvent::LanguageChange` 事件进行任何处理. 那么这个事件究竟在哪里被处理了呢? 

本文中多次提到 `QCoreApplication` 这个单例, 这个单例我们可以将其看作就是我们的 Qt 程序. 但相信你在 Qt 工程的 `main` 方法中还发现过 `QApplication` 或者 `QGuiApplication` 这两个类. 这三个类之间具有继承关系, 它们都可以代表着我们的 Qt 程序. 这里为了方便后文讲解简单介绍一下三者关系:

- `QCoreApplication` : 继承自 `QObjcet`, 是 Qt 程序的核心. 用于无 GUI 界面的 Qt 程序.
- `QGuiApplication` : 继承自 `QCoreApplication`. 用于**仅**使用 QML 实现 GUI 界面的 Qt 程序.
- `QApplication` : 继承自 `QGuiApplication`.用于使用**了**任意 `QWidget` 相关类实现 GUI 界面的 Qt 程序.

当我们的应用程序是无 GUI 界面 `QCoreApplication` 时, 不需要考虑更新 UI 界面上文本的需求, 自然也就不用关心 `QEvent::LanguageChange` 事件了. 但若是有 GUI 界面的 `QGuiApplication` 程序或 `QApplication` 程序, 则是需要在接收到这个事件后更新 UI 界面上文本. 因此这里 `QCoreApplication` 将 `QEvent::LanguageChange` 发给 `self` , 当 `self` 是后两种 Qt App 单例的话, 就应该会去处理这个事件了. 下面我们来逐个验证:

首先来看 `QGuiApplication::event` 的实现:

```cpp
bool QGuiApplication::event(QEvent *e)
{
    if(e->type() == QEvent::LanguageChange) {  // 有处理 QEvent::LanguageChange 事件
        setLayoutDirection(qt_detectRTLLanguage()?Qt::RightToLeft:Qt::LeftToRight);
    } else if (e->type() == QEvent::Quit) {
        // Close open windows. This is done in order to deliver de-expose
        // events while the event loop is still running.
        for (QWindow *topLevelWindow : QGuiApplication::topLevelWindows()) {
            // Already closed windows will not have a platform window, skip those
            if (!topLevelWindow->handle())
                continue;
            if (!topLevelWindow->close()) {
                e->ignore();
                return true;
            }
        }
    }

    return QCoreApplication::event(e);
}
```

可见 `QGuiApplication::event` 事件是有被处理的.

> 由于 `QGuiApplication` 是用于纯 QML 实现 GUI 界面的 Qt 程序中, 而我对 QML 的了解程度基本为 0, 因此这里也没有办法再为你继续讲解这个 `setLayoutDirection` 方法到底做了什么. 若以后我学了 QML 再来补充吧.

然后我们再来看 `QApplication::event` 的实现:

```cpp
bool QApplication::event(QEvent *e)
{
    Q_D(QApplication);
    if (e->type() == QEvent::Quit) {
    ......
#ifndef Q_OS_WIN
    } else if (e->type() == QEvent::LocaleChange) {
    ......
#endif
    } else if (e->type() == QEvent::Timer) {
    ......
#if QT_CONFIG(whatsthis)
    } else if (e->type() == QEvent::EnterWhatsThisMode) {
    ......
#endif
    }

    if(e->type() == QEvent::LanguageChange) {  // 有处理 QEvent::LanguageChange 事件
        /* 将事件转发给了所有 topLevelWidgets() 列表中的控件 */
        const QWidgetList list = topLevelWidgets();
        for (auto *w : list) {
            if (!(w->windowType() == Qt::Desktop))
                postEvent(w, new QEvent(QEvent::LanguageChange));  // 发送事件
        }
    }

    return QGuiApplication::event(e);
}
```

由于 `QApplication::event` 处理了很多事件, 我把无关的部分省略了. 我们可以看到这里 `QApplication` 将这个事件转发给了所有从方法 `topLevelWidgets` 返回的控件列表内的控件. OK, 至此我们知道了 `QCoreApplication` 会将 `QEvent::LanguageChange` 发送给 Qt App 单例, 当这个单例是 `QGuiApplication` 或 `QApplication` 时便会处理这个事件. 而 `QApplication` 的处理方式便是将事件转发给所有数据 `top-level widgets` 范畴内的控件. 那么接下来的问题就在于, `top-level widgets` 的定义到底是什么? 它的范畴有多大?

[<返回顶部>](#返回顶部)

<div id="toplevelwidgets包括了哪些控件"></div>

#### `top-level widgets` 包括了哪些控件?

我们接着来看这个 `QApplication::topLevelWidgets` 方法究竟返回了什么样的列表:

```cpp
// qwindowdefs.h
typedef QList<QWidget *> QWidgetList;

// qapplication.cpp
QWidgetList QApplication::topLevelWidgets()
{
    QWidgetList list;
    if (QWidgetPrivate::allWidgets != nullptr) {  // 返回的是 QWidgetPrivate::allWidgets 的子集
        const auto isTopLevelWidget = [] (const QWidget *w) {
            /* 判断条件 : isWindow() == true 且 windowType() != Qt::Desktop */
            return w->isWindow() && w->windowType() != Qt::Desktop;
        };
        std::copy_if(QWidgetPrivate::allWidgets->cbegin(), QWidgetPrivate::allWidgets->cend(),
                     std::back_inserter(list), isTopLevelWidget);  // 符合条件就加入返回列表
    }
    return list;
}
```

可见成为 `top-level widgets` 一份子的控件需要满足以下三个条件:

- 该控件在 `QWidgetPrivate::allWidgets` 之内;
- 该控件的 `isWindow` 方法返回值为 `true`;
- 该控件的 `windowType` 方法返回值不为 `Qt::Desktop`.

下面我们来逐一检查, 首先是 `QWidgetPrivate::allWidgets`. 我们来看一下声明部分:

```cpp
// qwindowdefs.h
template<class K, class V> class QHash;
typedef QHash<WId, QWidget *> QWidgetMapper;

template<class V> class QSet;
typedef QSet<QWidget *> QWidgetSet;

// qwidget_p.h
class Q_WIDGETS_EXPORT QWidgetPrivate : public QObjectPrivate
{
......
public:
    // All widgets are added into the allWidgets set. Once
    // they receive a window id they are also added to the mapper.
    // This should just ensure that all widgets are deleted by QApplication
    static QWidgetMapper *mapper;
    static QWidgetSet *allWidgets;
......
};
```

根据官方注释, 这个 `QSet` 保存了所有控件的指针, 并且其存在目的是为了便于 `QApplication` 可以方便地删除所有控件. 我检查了一下实现部分, 发现确实如此. 当有 `QWidget` 被构造时便将其指针加入这个 `QSet`, 而当这个 `QWidget` 被析构时, 也会将对应的指针移除这个 `QSet`. 到这里条件一明确了, `QWidgetPrivate::allWidgets` 实际上就是当前存在的所有 `QWidget`.

接下来我们来检查剩下的条件, 也就是 `QWidget::isWindow` 与 `QWidget::windowType` 的实现:

```cpp
class Q_WIDGETS_EXPORT QWidget : public QObject, public QPaintDevice
{
public:
    inline Qt::WindowType windowType() const;
    bool isWindow() const;
......
QWidgetData *data;
};
inline bool QWidget::isWindow() const
{ return (windowType() & Qt::Window); }

inline Qt::WindowType QWidget::windowType() const
{ return static_cast<Qt::WindowType>(int(data->window_flags & Qt::WindowType_Mask)); }
```

这里 `isWindow` 通过 `windowType` 的返回值与枚举 `Qt::WindowType::Window` 进行按位与, 而 `windowType` 又是通过 `data->window_flags` 与与枚举 `Qt::WindowType::WindowType_Mask` 进行按位与. 那么我们再来看 `QWidgetData *data` 与 `data->window_flags` 以及 `Qt::WindowType` 是什么:

```cpp
// qwidget.h
class QWidgetData
{
public:
    WId winid;
    uint widget_attributes;
    Qt::WindowFlags window_flags;  // 注意这里
    ......
};

// qnamespace.h
enum WindowType {
    Widget = 0x00000000,
    Window = 0x00000001,                    // & Qt::Window == true
    Dialog = 0x00000002 | Window,           // & Qt::Window == true
    Sheet = 0x00000004 | Window,            // & Qt::Window == true
    Drawer = Sheet | Dialog,                // & Qt::Window == true
    Popup = 0x00000008 | Window,            // & Qt::Window == true
    Tool = Popup | Dialog,                  // & Qt::Window == true
    ToolTip = Popup | Sheet,                // & Qt::Window == true
    SplashScreen = ToolTip | Dialog,        // & Qt::Window == true
    Desktop = 0x00000010 | Window,          // & Qt::Window == true, != Qt::Desktop == false
    SubWindow = 0x00000012,
    ForeignWindow = 0x00000020 | Window,    // & Qt::Window == true
    CoverWindow = 0x00000040 | Window,      // & Qt::Window == true

    WindowType_Mask = 0x000000ff,           // 注意这里
    ......
};

Q_DECLARE_FLAGS(WindowFlags, WindowType)  // Qt::WindowFlags 其实就是 Qt::WindowType 的 QFlags 形式
```

看来 `data->window_flags` 是 `Qt::WindowFlags` , 而 `Qt::WindowFlags` 又是通过 `Q_DECLARE_FLAGS` 宏根据 `Qt::WindowType` 枚举生成的一个 `QFlags`, `QFlags` 实际上与 `enum class` 类似, 都是通过 `class` 来实现枚举, 实际上二者提供的方法有所差异, 但在这里我们可以把 `Qt::WindowFlags` 看作是一个特殊的枚举类即可.

经过检查, 我们可以得出控件的 `WindowType` 等于这几个枚举时, 条件二成立:

- `Qt::WindowType::Window`
- `Qt::WindowType::Dialog`
- `Qt::WindowType::Sheet`
- `Qt::WindowType::Drawer`
- `Qt::WindowType::Popup`
- `Qt::WindowType::Tool`
- `Qt::WindowType::ToolTip`
- `Qt::WindowType::SplashScreen`
- `Qt::WindowType::Desktop` : 在条件三中不成立
- `Qt::WindowType::ForeignWindow`
- `Qt::WindowType::CoverWindow`

而条件三则是在条件二的基础上排除了 `Qt::Desktop` 这个选项. 

最终, 我们发现满足 `top-level widgets` 条件的控件是非常多的, 只有当一个控件的 `WindowType` 属于以下两类时, 才不被视为 `top-level widgets`:

- `Qt::WindowType::SubWindow` : 当控件是子窗口时拥有此枚举.
- `Qt::WindowType::Desktop` : 只有 `QDesktopWidget` 拥有此枚举.

换言之, 只要我们想要接收 `QEvent::LanguageChange` 事件的控件的 `WindowType` 不是以上两种之一时, 就可以正常地接收到 `QEvent::LanguageChange` 事件了.

[<返回顶部>](#返回顶部)

---

<div id="总结"></div>

# 总结

在此之前我一直对 Qt 的多语言机制了解不够深入, 一直不知道 Qt 究竟什么时候才会跟新我的 UI 控件, 不知道 `QObject::tr` 方法究竟是怎么得到译文的. 不过现在, 我们分析完了 Qt 原生多语言机制的源码部分. 相信读到这的你已经对这个系统的实际运作了然于胸了.

最后, 当你使用 Qt 的多语言机制时, 请牢记我在最开始给出的结论. Qt 实现的核心功能是 : **用户可以在任意时间任意位置获取到某段指定文本的最新翻译版本**, 其余的功能都是围绕着这个核心功能所展开. 相信你充分理解这句话的意思后可以游刃有余地运用 Qt 的原生多语言机制了.

我接下来会给出一个完全依赖于 Qt 原生多语言机制实现动态语言切换的范例. 不过篇幅有限, 写到这已经 1000 多行了, 之后再发上来吧.

[<返回顶部>](#返回顶部)