# 第零一章 • 安装

在开始学习 C 语言之前，我们需要安装一些必要的东西，搭建好编程的环境。安装的过程并不复杂，实际上我们只需要两样最主要的东西：一个代码编辑器和一个编译器。

## 文本编辑器

代码编辑器其实就是一个更适合写代码的文本编辑器。

在 *Linux* 上，我建议你用 *gedit*。不过如果你手头上有现成的代码编辑器可以用的话，也是可以的。请不要使用  **IDE**。杀鸡焉用宰牛刀，本书构建的小程序不会用到 IDE 所提供的便利，反而让你不知道到底发生了什么。

在 *Mac OS X* 上，一个可以使用的编辑器是 [*TextWrangler*](http://www.barebones.com/products/textwrangler/)，如果你有其他喜欢的，也是可以的，但请不要使用 **Xcode**，这种小项目使用 IDE 反而会让你搞不清楚细节。

在 *Microsoft Windows* 上，我建议你使用 *Notapad++*，如果你有其他喜欢的，也是可以的啦。但是请不要使用 **Visual Studio**，因为它对 C 语言的支持并不好，如果使用它你会遇到很多问题。

## 编译器

编译器的作用是将我们写好的 C 语言的代码翻译成机器能够运行的程序。不同的操作系统安装编译器的过程也是有差别的。

另外编译和运行 C 程序需要知道一些基本的命令行操作，本书不会教你怎么使用命令行。如果你从来没听说过命令行，你可以到网上搜一些教程看看。

在 *Linux* 上，你可以下载一下包来安装编译器。如果你的系统是 Ubuntu 或 Debian，你可以通过这行命令来安装：`sudo apt-get install build-essential`。

在 *Mac OS X* 上，你需要在应用商店里下载并最新版的 Xcode。然后在命令行中运行 `xcode-select --install` 来安装 *Command Line Tools*。

在 *Microsoft Windows* 上，你可以下载并安装 [MinGW](http://www.mingw.org/)，具体的安装及配置方法可以到网上搜几个教程看一看。

## 测试安装好的 C 编译器

为了验证一下 C 编译器是否安装成功了，请在命令行中键入下面的语句并运行：

`cc --version`

如果你得到了一些关于编译器版本的信息，那就说明安装成功了！在我的 Mac 上，返回信息如下所示：

    $ cc --version
    Apple LLVM version 7.0.0 (clang-700.1.76)
    Target: x86_64-apple-darwin15.0.0
    Thread model: posix
