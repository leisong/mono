“Mono 是一个软件平台，旨在让开发人员轻松创建跨平台应用程序。它是基于 C# 的 ECMA 标准和公共语言运行时的 Microsoft .NET Framework 的开源实现。”

编译安装
=======

最好使用 Mac OS X，iOS 和 Android 都可以编译。也可使用 Linux，可以编译 Android。Windows 下没有尝试，不推荐

下面是针对 Mac OS X 的编译说明，详细可参照
[Mac OS X](https://www.mono-project.com/docs/compiling-mono/mac/),
[Linux](https://www.mono-project.com/docs/compiling-mono/linux/) 

安装依赖项
---------

    brew install autoconf automake libtool pkg-config cmake python3

从 Git 签出构建 Mono
-------------------

    PREFIX=/usr/local
    PATH=$PREFIX/bin:$PATH
    git clone https://github.com/mono/mono.git
    cd mono
    ./autogen.sh --prefix=$PREFIX --disable-nls
    make
    make install

使用 Mono
=========

安装完 Mono 后，可以使用 `mono program.exe`（运行时引擎）、`mcs program.cs`（C# 编译器）等工具。

移动平台下构建 Mono
==================
相比在电脑上使用 Mono，我们更需要的是在移动平台的 App 中嵌入运行 Mono 的能力。在 sdks 目录下能找到我们所需要的

依赖项
-----
* automake 1.16.1

  如果曾经用以前版本的 automake 构建过，可能需要清理

  ```bash
  git clean -xffd
  ```

  或者

  ```bash
  git reset --hard && git clean -xffd && git submodule foreach --recursive git reset --hard && git submodule foreach --recursive git clean -xffd && git submodule update --init --recursive
  ```

- [ninja 构建系统](https://ninja-build.org)

  - 获得 Ninja

    - Linux: `apt-get install ninja-build`
    - Mac: `brew install ninja`

设置
----

复制 Make.config.sample 到 Make.config，安卓打开 ENABLE_ANDROID = 1，iOS 打开 ENABLE_IOS = 1

构建
---

    # Android
    make -C sdks/builds provision-android && make -C sdks/android accept-android-license
    make -C sdks/builds provision-mxe
    make -C sdks/builds archive-android NINJA= IGNORE_PROVISION_ANDROID=1 IGNORE_PROVISION_MXE=1

    # iOS
    make -C sdks/builds archive-ios NINJA=

默认构建是 release 版本，想要 debug 版本可以在 make 命令后加上 CONFIGURATION=debug。构建成功后在 sdks/out 中能找到各种平台对应的库和头文件。

移动平台使用 Mono
================

Android 示例
-------

安卓提供了一个例子，在 sdks/android 下，经研究发现它是一个安卓的单元测试应用，里面使用了 C# 的 unitlite 测试框架。虽然缺乏文档，但是这个例子将 mono 运行时功能展示的相当充分，是一个很好的参考

### 编译

保证前面的安卓构建成功，启动模拟器或者连接安卓真机，然后执行

    cd sdks/android
    make check-*
    
其中 * 号代表要测试的库，如 `make check-corlib`，会调用 nunit-lite 对示例中的安卓应用进行测试

### 流程

sds/android/Makefile 中指明了测试的流程：

    .PHONY: check-%
    check-%: .stamp-install
    	PATH="$$PATH:$(dir $(ADB))" mono --debug $(TOP)/sdks/out/android-bcl/monodroid_tools/nunit-lite-console.exe \
    		-labels \
    		-exclude:NotOnMac,NotWorking,CAS,InetAccess,MobileNotWorking,AndroidNotWorking,LargeFileSupport,AndroidSdksNotWorking \
    		-android:$(PACKAGE)/$(RUNNER) \
    		-result:TestResult-$*.xml \
    		-format:nunit2 \
    		$(if $(TESTNAME),-test:$(TESTNAME)) \
    		$(TOP)/sdks/out/android-bcl/monodroid/tests/monodroid_$*_test.dll
    		
先是依赖于安卓应用构建并安装到设备，接着本机的 mono 调用了本机 nunit-lite-console.exe，其中 android 选项会调用 Xamarin.AndroidTestAssemblyRunner 启动设备中的单元测试应用，对设备中的 monodroid_$*_test.dll 进行测试（注：虽然路径给的是本机的 dll，实际 Xamarin.AndroidTestAssemblyRunner 中只取了文件名），最后通过本机和设备的 tcp 连接和端口 6100 把测试结果发送回本机，输出 log 以及存储测试结果到 xml
    
### 代码分析

这里只对安卓应用的代码做简要分析。应用包含 java 文件 AndroidRunner.java 和 C 代码 runtime-bootstrap.c

* AndroidRunner.java

    * onCreate
    获取测试 dll 文件和参数
    
    * onStart
    将 apk 中的 mono 库以及测试库文件拷贝到设备文件目录下
    调用 C 的 native 代码 runTests
    
* runtime-bootstrap.c
    
    初始化 mono 运行时环境，使用 mono 库 nunitlite.dll 对传进来的测试 dll 文件进行单元测试。

    * 代码前面用了很多篇幅来声明 mono 的枚举、结构体以及函数，应该是为了去除编译器对 mono 头文件的依赖，使得没有安装 mono 也能编译通过。接下来是一些 help 函数，用来打印 log、操作字符串、操作文件以及环境变量等

    * Java_org_mono_android_AndroidRunner_runTests
    对应 AndroidRunner.java 中的 runTexts，真正的测试代码所在。
    
        * 首先加载了 libmonosgen-2.0.so 库，然后使用 dlsym 取出库中的 mono 函数指针并对前面声明的函数赋值。而后设置了 mono 的运行环境，个人觉得比较重要的有 `mono_set_assemblies_path`设置 mono 库文件路径，以及`mono_dllmap_insert`加载 mono 库文件等。
        * `root_domain = mono_jit_init_version ("TEST RUNNER", "mobile");` 初始化 mono 运行时，返回一个 MonoDomain，代码将在其中执行。[参考这里](https://www.mono-project.com/docs/advanced/embedding/#initializing-the-mono-runtime)
        * 调用 `mono_assembly_open` 打开 nunitlite.dll；`mono_class_from_name` 获得 Xamarin.AndroidTestAssemblyRunner.Driver 类；`mono_class_get_method_from_name` 获得 RunTests 函数；`mono_runtime_invoke` 运行函数
        
    * 后面的代码定义了很多帮助函数，以及与安卓交互的函数，略去不提
        
    
iOS 示例
---

iOS 提供了一个例子，在 sdks/ios 下，它也是一个使用了 unitlite 的单元测试应用

### 编译

保证前面的 iOS 构建成功

    cd sdks/ios
    # 模拟器
    make build-ios-sim-all
    make run-ios-sim-all
    # 真机(解释器模式)
    make build-ios-dev-interp-only-all IOS_SIGNING_IDENTITY="iPhone Developer: XXX" IOS_TEAM_IDENTIFIER="XXXX"
    make run-ios-dev-all
    
真机非解释器模式我尝试跑了一下，没有成功就没再深究。使用解释器模式没有问题，还有一种解释器混合 aot 模式没有尝试。

### 分析

示例除了 iOS 的 app 代码，还包含了三个辅助的 C# 程序，分别是 app 构建器(appbuilder)、单元测试代码(test-runner)、app 启动器(harness)，它们都是使用 mono 来运行的。

* app

    初始化 mono 运行时环境，从参数或配置文件中获取测试用例 dll 文件来加载并运行

    * main.m

        创建 UI 布局，显示测试用例的运行结果

    * runtime.m

        重点是 mono_ios_runtime_init 函数，在运行目标是真机时调用了 appbuilder 生成的 modules.m 中的函数，注册 aot 模块并设置 aot 运行模式(full aot 模式或解释器模式)。还有值得注意的是设置了几个 hook 函数，在预加载程序集(assembly)、预加载 aot 数据、异常处理等时候做一些工作。

* appbuilder

    根据 make 文件传递的参数，生成 mono 模块代码文件(modules.m)和 build.ninja 文件，最终生成 iOS 应用。

    * modules.m

        由 appbuilder 自动生成，包括两个函数：

        1. mono_ios_register_modules 注册 aot 模块，在解释器模式下只有 mscorlib.dll 被注册

        2. mono_ios_setup_execution_mode 设置 aot 运行模式，此处设置为 MONO_AOT_MODE_INTERP  

    * build.ninja

        由 appbuilder 自动生成的 ninja 构建文件，主要做的事有：

        1. 使用 aarch64-darwin-mono-sgen 从核心库 mscorlib.dll 中生成符号文件 mscorlib.dll.s 和 aot 数据文件 mscorlib.aotdata；之后由符号文件 mscorlib.dll.s 生成 mscorlib.dll.o

        2. 拷贝 mono 基础类文件到 app，即 System.dll 等；拷贝单元测试库等文件

        3. 链接 mono 静态库、module.o(由 module.m 生成)、以及预先由 make 构建生成的 app 库，最终生成 app 中的可执行文件

* test-runner

    使用 xunit 或 nunit 的单元测试程序

* harness

    接受测试应用包名、测试平台等参数，在模拟器或真机安装部署应用，运行应用，建立 tcp 连接接收测试结果

补充
====

构建
----

我 fork 了一个 mono 项目，做了一些修改，来更好的说明
- 安卓构建时可删除一些用不到的目标，如跨平台混合构建等，具体可见[这里的修改](https://github.com/leisong/mono/commit/0e6b5a337f12c12b823122248dc82fe7c8998455)

坑
--

* make 过程中安装的 cmake 版本 3.18.1，编译安卓示例代码报错，我安装了 3.10.2 可以。使用 cmake 3.10.2 首先安装，然后在环境变量中添加 cmake 3.10.2 的 bin 目录到 PATH 的前面，然后修改 [build.gradle](https://github.com/leisong/mono/commit/b06617cec0a8137976cfdca45383b94716d30064)

* 安卓编译工具链要求比较严格，有时差一点都不行。比如打出来的 so 是带版本的，soname 是 xxx.so.0，安卓不认。还没找到具体原因，但是重新清理打包解决了

* 为了查 so 带版本的问题在 Mac 上装了 binutils，结果工具链变成了使用 binutils 带的，编译报错。原因查了很久终于在[这里](https://github.com/bitcoin/bitcoin/issues/20825)找到

* Mac 构建时可能会遇到 brew install mingw-zlib 失败，使用 `brew edit` 修改了对应的 formula 解决

* iOS 版本有若干 bug，如[这里缺少了依赖库](https://github.com/mono/mono/commit/de3fc68d763d90909d369550182d687b70d39e8d)，[这里参数应该使用真机的设置](https://github.com/mono/mono/commit/c82facef441a72417c4c85b0d576083d03306595)，[某版本的模拟器终止未运行的 app 会返回非 0 错误码](https://github.com/mono/mono/commit/73db21a90ae613b4a48aa6671dd03cd05e847ce9)。另外，iOS 程序要在使用多域 MonoDomain 时，要注意多线程的问题，不同线程操作域会 crash，待研究

命名缩写
-------
BCL : Base Class Library

TPN : Third Party Notice

参考文档
--------
https://www.mono-project.com/docs/compiling-mono/mac/

https://github.com/mono/mono/blob/main/sdks/README.md

https://www.mono-project.com/docs/advanced/embedding/

http://docs.go-mono.com/

https://www.mono-project.com/docs/advanced/aot/#full-aot

https://www.mono-project.com/docs/advanced/runtime/docs/aot/

https://www.mono-project.com/news/2017/11/13/mono-interpreter/
