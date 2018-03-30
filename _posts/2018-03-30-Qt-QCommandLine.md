---
layout:     post
title:      "Qt 解析命令行"
subtitle:   "Qt 解析命令行参数，类似 argparse 工具"
date:       2018-03-30
author:     Mcoder
header-img: img/JCQ_0383-Pano.jpg
catalog: true
tags:
  - 日常学习
  - Qt
---

# Qt解析命令行
我们使用 Python 写个简单的脚本很方便，直接 `import argparse` 就能很容易的实现命令行参数解析的功能，还可以通过 `--help` 来输出帮助功能，而 Qt5 页提供了这方面的支持。

Qt 从 Qt5.2之后提供了 `QCommandLineParser` 和 `QCommandLineOption` 两个类来负责这个功能。

`QCoreApplication` 提供了方法用一个简单的 String 队列来获取参数；`QCommandLineParser` 提供了定义一些选项、复制命令行参数、储存实际有必要的参数的能力。如果一个参数不是一个选项，那么它会被认为是`positional argument`. `QCommandLineParser` 可以处理短名称`-a`、长名称`--all`、多个名称、选项值等。

命令行的选型是由 '-' 或 "--" 开头的字符串。短选项是'-a'，只有一个字母，通常表示标准输入。有的时候也会写成紧凑样式，比如`-a -b -c `表示三个选项，`-abc`也表示三个短选项，只有当你设置 `QCommandLineParser` 的模式为 `ParseAsLongOptions` 时，`-abc` 才表示维长选项。

长选项多余两个字母长，并且不可以写成紧凑样式，例如 `verbose` 写为`-verbose 或 --verbose`而且写错了并不会自动纠正。

在选项后面赋值有两种方式，一种是空格符间隔将值跟在选项后面，另一种是选项紧跟'='号和值。例如:`-v=value --verbose=value` 或者是 `-v value --verbose value`。

而且经过我的测试，Qt 的这个工具获得的值只能是 QString 类型，也就是说，如果你需要的是 `int double float` 之类的参数，你需要通过其他的函数自己进行转换。

## 样例代码

```C++
int main(int argc, char *argv[])
  {
      // 设置程序信息
      QCoreApplication app(argc, argv);
      QCoreApplication::setApplicationName("my-copy-program");
      QCoreApplication::setApplicationVersion("1.0");

      // 解析工具
      QCommandLineParser parser;
      parser.setApplicationDescription("Test helper");
      parser.addHelpOption();
      parser.addVersionOption();
      parser.addPositionalArgument(
          "source",
          QCoreApplication::translate("main", "Source file to copy."));
      parser.addPositionalArgument(
          "destination",
          QCoreApplication::translate("main", "Destination directory."));

      // A boolean option with a single name (-p)
      QCommandLineOption showProgressOption(
          "p",
          QCoreApplication::translate("main", "Show progress during copy"));
      parser.addOption(showProgressOption);

      // A boolean option with multiple names (-f, --force)
      QCommandLineOption forceOption(
          QStringList() << "f" << "force",
          QCoreApplication::translate("main", "Overwrite existing files."));
      parser.addOption(forceOption);

      // An option with a value
      QCommandLineOption targetDirectoryOption(
          QStringList() << "t" << "target-directory",
          QCoreApplication::translate("main", "Copy all source files into <directory>."),
          QCoreApplication::translate("main", "directory"));
      parser.addOption(targetDirectoryOption);

      // Process the actual command line arguments given by the user
      parser.process(app);

      const QStringList args = parser.positionalArguments();
      // source is args.at(0), destination is args.at(1)

      bool showProgress = parser.isSet(showProgressOption);
      bool force = parser.isSet(forceOption);
      QString targetDir = parser.value(targetDirectoryOption);
      // ...
  }
```

如果你使用了C++ 11 标准的编译器，那么你还可以用下面的写法来做。
```C++
      parser.addOptions({
          // A boolean option with a single name (-p)
          {"p",
              QCoreApplication::translate("main", "Show progress during copy")},
          // A boolean option with multiple names (-f, --force)
          {{"f", "force"},
              QCoreApplication::translate("main", "Overwrite existing files.")},
          // An option with a value
          {{"t", "target-directory"},
              QCoreApplication::translate("main", "Copy all source files into <directory>."),
              QCoreApplication::translate("main", "directory")},
      });
```


## 设置为bool类型的
功能是判断是否有出现过某个选项，操作如下：
```C++
QCoreApplication a(argc, argv);
// A boolean option with a single name (-p)
QCommandLineOption showProgressOption(
            "p",
            QCoreApplication::translate("main", "Show progress during copy"));
parser.addOption(showProgressOption);

// ......
parser.process(a);

bool showProgress = parser.isSet(showProgressOption);   // 查看选项是否出现在命令行参数中。
```

## 设置可获得值的参数
```C++
QCoreApplication a(argc, argv);
// An option with a value
QCommandLineOption targetDirectoryOption(
            QStringList() << "t" << "target-directory",
        QCoreApplication::translate("main", "Copy all source files into <directory>."),
        QCoreApplication::translate("main", "directory"));
parser.addOption(targetDirectoryOption);

// .......
parser.process(a);
QString targetDir = parser.value(targetDirectoryOption);

```

# Reference
- [QCommandLineParser Class](http://doc.qt.io/qt-5/qcommandlineparser.html)
- [QCommandLineOption Class](http://doc.qt.io/qt-5/qcommandlineoption.html)
