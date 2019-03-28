# rust-cross

> 关于交叉编译Rust程序需要了解的一切！

如果您想将Rust工具链设置为交叉编译器，那么您来对地方了！ 我已经记录了所有必要的步骤，以及在此过程中可能遇到的陷阱和常见问题。

> 亲爱的读者，如果你发现一个错字，一个失效的链接，或一个措辞不好/混乱的句子/段落，请打开一个指出问题的`issue`，我会更新文本。 当然，欢迎提出修复拼写错误或失效链接的`PR`！

## 太长不看：Ubuntu 示例

在新的Ubuntu Trusty上，以下命令将stable Rust工具链设置为 ARMv7 设备的交叉编译器。此示例旨在展示交叉编译的设置非常简单。

注意：这些指令不适用于 Raspberry Pi (1) 的交叉编译，那是一个 ARM v6 设备。

    # 安装 Rust. 强烈推荐 rustup.rs (https://www.rustup.rs/)
    # 或者你也可以使用 multirust (https://github.com/brson/multirust)
    $ curl https://sh.rustup.rs -sSf | sh

    # 步骤 0: 我们的目标是一个 ARMv7 设备
    # 目标三元组为`armv7-unknown-linux-gnueabihf`

    # 步骤 1: 安装 C 交叉编译工具链
    $ sudo apt-get install -qq gcc-arm-linux-gnueabihf

    # 步骤 2: 安装交叉编译目标
    $ rustup target add armv7-unknown-linux-gnueabihf

    # 步骤 3: 为 cargo 配置交叉编译选项
    $ mkdir -p ~/.cargo
    $ cat >>~/.cargo/config <<EOF
    > [target.armv7-unknown-linux-gnueabihf]
    > linker = "arm-linux-gnueabihf-gcc"
    > EOF

    # 测试：交叉编译一个 Cargo 项目
    $ cargo new --bin hello
    $ cd hello
    $ cargo build --target=armv7-unknown-linux-gnueabihf
    Compiling hello v0.1.0 (file:///home/ubuntu/hello)
    $ file target/armv7-unknown-linux-gnueabihf/debug/hello
    hello: ELF 32-bit LSB  shared object, ARM, EABI5 version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=67b58f42db4842dafb8a15f8d47de87ca12cc7de, not stripped

    # 测试程序
    $ scp target/armv7-unknown-linux-gnueabihf/debug/hello me@arm:~
    $ ssh me@arm:~ ./hello
    Hello, world!

1\. 2. 3. 完成！

有关更多示例，请查看 [Travis CI 构建](https://travis-ci.org/japaric/rust-cross).

本指南的其余部分将解释并概括前一个示例中执行的每个步骤。

## 目录

本指南分为两部分：主要文本，高级主题。主要文本涵盖了最简单的情况：交叉编译 Rust 程序，这些程序依赖于 `std` crate 来编译到官方构建支持的目标。高级主题部分包括 `no_std` 程序，目标特化文件，如何交叉编译“标准crate”，以及常见问题的解决方法。

高级主题部分建立在正文中解释的信息之上。 因此，即使您的用例与主文本所涵盖的用例不同，您仍应在跳转到高级主题部分之前阅读主要文本。

<!-- TOC -->

- [rust-cross](#rust-cross)
    - [太长不看：Ubuntu 示例](#太长不看ubuntu-示例)
    - [目录](#目录)
    - [术语](#术语)
    - [依赖](#依赖)
        - [目标三元组](#目标三元组)
        - [C 交叉工具链](#c-交叉工具链)
        - [交叉编译 Rust crate](#交叉编译-rust-crate)
    - [使用 `rustc` 交叉编译](#使用-rustc-交叉编译)
    - [使用 `cargo` 交叉编译](#使用-cargo-交叉编译)
    - [高级主题](#高级主题)
        - [交叉编译标准crate](#交叉编译标准crate)
        - [安装交叉编译的标准包](#安装交叉编译的标准包)
        - [目标规范文件](#目标规范文件)
        - [交叉编译`no_std`代码](#交叉编译no_std代码)
        - [常见问题](#常见问题)
            - [找不到crate](#找不到crate)
                - [症状](#症状)
                - [原因](#原因)
                - [解法](#解法)
            - [crate 与此版本的 rustc 不兼容](#crate-与此版本的-rustc-不兼容)
                - [症状](#症状-1)
                - [原因](#原因-1)
                - [解法](#解法-1)
            - [未定义引用](#未定义引用)
                - [症状](#症状-2)
                - [原因](#原因-2)
                - [解法](#解法-2)
            - [无法加载库](#无法加载库)
                - [症状](#症状-3)
                - [原因](#原因-3)
                - [解法](#解法-3)
            - [`$symbol` 未找到](#symbol-未找到)
                - [症状](#症状-4)
                - [原因](#原因-4)
                - [解法](#解法-4)
            - [非法指令](#非法指令)
                - [症状](#症状-5)
                - [原因](#原因-5)
    - [FAQ](#faq)
        - [我想为Linux，Mac和Windows构建二进制文件。如何从Linux交叉编译到Mac？](#我想为linuxmac和windows构建二进制文件如何从linux交叉编译到mac)
        - [如何编译完全静态链接的Rust二进制文件？](#如何编译完全静态链接的rust二进制文件)

<!-- /TOC -->

## 术语

让我们首先通过定义一些术语来确保我们说的是同一种东西！

在最基本的形式中，交叉编译涉及两个不同的系统/计算机/设备。

**host**：编译程序的**主机**系统
**target**：执行编译程序的**目标**系统。

例如，如果您在笔记本电脑上交叉编译 Rust 程序，则在 Raspberry Pi 2 (RPi2) 上执行它。你的笔记本电脑是**主机**，RPi2 是**目标**。

但是，（交叉）编译器不会生成仅适用于单个系统（例如RPi2）的二进制文件。生成的二进制文件也可以在几个其他系统（例如ODROID）上执行，这些系统共享某些特性，例如它们的架构（例如ARM）和它们的操作系统（例如Linux）。要引用具有共享特征的这组系统，我们使用一个名为 **triple** 的字符串。**三元组**的格式通常如下：{arch} - {vendor} - {sys} - {abi}。

例如，`arm-unknown-linux-gnueabihf`指的是具有以下特征的系统：

+ architecture (架构): `arm`.
+ vendor (供应商): `unknown`.
    在这种情况下没有指定供应商，这不重要。
+ system (系统): `linux`.
+ ABI: `gnueabihf`.
    `gnueabihf`表示系统使用`glibc`作为其C标准库 (libc) 实现，并具有硬件加速浮点算法（即FPU）。

像 RPi2，ODROID 这样的系统，以及运行 GNU/Linux 系统的几乎所有 ARMv7 开发板都属于这个三元组。

有些三元组省略了供应商或 abi 组件，因此它们是真正的“三元组”。 这种三元组的一个例子是`x86_64-apple-darwin`，其中：

+ 架构: `x86_64`.
+ 供应商: `apple`.
+ 系统: `darwin`.

**注意**：从现在开始，我将重载术语 **目标** 以表示单个目标系统，并且还要引用具有由某个三元组指定的共享特征的一组系统。

## 依赖

要编译Rust程序，我们需要4件东西：

+ 找出目标系统的三元组。
+ 一个`gcc`交叉编译器，因为`rustc`使用`gcc`将东西[链接](https://en.wikipedia.org/wiki/Linker_(computing))在一起。
+ C 依赖项，通常是为目标系统交叉编译的`libc`。
+ Rust 依赖项，通常是为目标系统交叉编译的`std` crate.

### 目标三元组

要找出目标的三元组，首先需要弄清楚这四个目标信息：

+ 架构：在类UNIX系统上，您可以使用命令 `uname -m` 找到它。
+ 供应商：
  + Linux：通常是 `unknown`
  + Windows：`pc`
  + OSX / iOS：`apple`
+ 系统：在类UNIX系统上，您可以使用命令`uname -s`找到它。
+ ABI：
  + Linux上，ABI 指的是 libc 实现，你可以用`ldd --version`找到它。
  + Mac 和 *BSD 系统不提供多个 ABI，因此省略该字段。
  + Windows上，据我所知，只有两个 ABI ：gnu 和 msvc。

接下来，您需要将此信息与`rustc`支持的目标进行比较，并检查是否存在匹配。如果您有`nightly-2016-02-14`,`1.8.0-beta.1`或更新的`rustc`，您可以使用`rustc --print target-list`命令获取支持目标的完整列表。

以下是`1.8.0-beta.1`支持的目标列表：

    $ rustc --print target-list | pr -tw100 --columns 3
    aarch64-apple-ios                i686-pc-windows-gnu              x86_64-apple-darwin
    aarch64-linux-android            i686-pc-windows-msvc             x86_64-apple-ios
    aarch64-unknown-linux-gnu        i686-unknown-dragonfly           x86_64-pc-windows-gnu
    arm-linux-androideabi            i686-unknown-freebsd             x86_64-pc-windows-msvc
    arm-unknown-linux-gnueabi        i686-unknown-linux-gnu           x86_64-rumprun-netbsd
    arm-unknown-linux-gnueabihf      i686-unknown-linux-musl          x86_64-sun-solaris
    armv7-apple-ios                  le32-unknown-nacl                x86_64-unknown-bitrig
    armv7-unknown-linux-gnueabihf    mips-unknown-linux-gnu           x86_64-unknown-dragonfly
    armv7s-apple-ios                 mips-unknown-linux-musl          x86_64-unknown-freebsd
    asmjs-unknown-emscripten         mipsel-unknown-linux-gnu         x86_64-unknown-linux-gnu
    i386-apple-ios                   mipsel-unknown-linux-musl        x86_64-unknown-linux-musl
    i586-unknown-linux-gnu           powerpc-unknown-linux-gnu        x86_64-unknown-netbsd
    i686-apple-darwin                powerpc64-unknown-linux-gnu      x86_64-unknown-openbsd
    i686-linux-android               powerpc64le-unknown-linux-gnu

**注意**：如果你想知道`arm-unknown-linux-gnueabihf`和`armv7-unknown-linux-gnueabihf`之间有什么区别，`arm`三元组覆盖 ARMv6 和 ARMv7 处理器，而`armv7`只支持 ARMv7 处理器。 因此，`armv7`三元组可以实现仅在 ARMv7 处理器上生效的优化。另一方面，如果你使用`arm`三元组，你必须通过将额外的标志（例如`-C target-feature=+neon`）传递给`rustc`来选择加入这些优化。

太长不看：为了生成更高效的二进制文件，如果目标有 ARMv7 处理器，请使用`armv7`。

如果您没有找到与目标系统匹配的三元组，那么您将需要[创建目标规范文件](#target-specification-files)。

从这里开始，我将使用变量`$rustc_target`来指代您在本节中找到的三元组。例如，如果你发现你的目标是`arm-unknown-linux-gnueabihf`，那么只要你看到类似于`--target=$rustc_target`的内容，就在心里展开`$rustc_target`，于是你知道这是`--target=arm-unknown-linux-gnueaibhf`

同样，我将使用变量`$host`来指代主机三元组。您可以在`rustc -Vv`的输出中的主机字段下找到此三元组。 例如，我的主机是`x86_64-unknown-linux-gnu`。

### C 交叉工具链

从这里开始，事情变得有点令人迷惑。

`gcc`交叉编译器只针对一个三元组。并且这个三元组用于为所有工具链命令添加前缀：`ar`，`gcc`等。这有助于区分本机工具链和交叉工具链，例如： `gcc` 和 `arm-none-eabi-gcc`。

令人困惑的是，三元组可能是非常随意的。所以你的 C 交叉编译器很可能会以与`$rustc_target`不同的三元组作为前缀。例如，在 Ubuntu 中，ARM 设备的交叉编译器打包为`arm-linux-gnueabihf-gcc`，同样的交叉编译器在 [Exherbo](http://exherbo.org/) 中以`armv7-unknown-linux-gnueabihf-gcc`为前缀，而`rustc`使用`arm-unknown-linux-gnueabihf`作为该目标的三元组。这些三元组都不匹配，但它们指的是同一组系统。

确认您的目标系统具有正确的交叉工具链的最佳方法是交叉编译C程序，最好是不重要的程序，并测试在目标系统上执行它。

至于何处获得C交叉工具链，这取决于您的系统。一些Linux发行版提供打包的交叉编译器。在一些情况下，您需要自己编译交叉编译器。像`crosstool-ng`这样的工具可以为你提供帮助。对于Linux 到 OSX，请查看[osxcross](https://github.com/tpoechtrager/osxcross)项目。

下面是一些打包的交叉编译器的示例：

+ 对于`arm-unknown-linux-gnueabi`, Ubuntu 和 Debian 提供了`gcc-*-arm-linux-gnueabi`包，其中 `*` 是 gcc 的版本。例如：`gcc-4.9-arm-linux-gnueabi`
+ 对于`arm-unknown-linux-gnueabihf`, 同上，但要用`gnueabihf` 替换 `gnueabi`。
+ 对于 OpenWRT 设备, 即 `mips-unknown-linux-uclibc` (15.05 或更早版本) 和 `mips-unknown-linux-musl` (15.05之后), 请使用[OpenWRT SDK](https://wiki.openwrt.org/doc/howto/obtain.firmware.sdk)
+ 对于 Raspberry Pi, 请使用[Raspberry tools](https://github.com/raspberrypi/tools/tree/master/arm-bcm2708)

注意：C交叉工具链将附带针对您的目标的交叉编译的libc。 确保：

+ 工具链libc与目标libc匹配。例如，如果您的目标使用 musl libc，那么您的工具链也必须使用 musl libc。

+ 工具链libc与目标libc兼容。这通常意味着工具链libc必须比目标libc旧。理想情况下，工具链libc和目标libc应具有完全相同的版本。

从这里开始，我将使用变量`$gcc_prefix`来引用您在本节中安装的交叉编译工具（即交叉工具链）的前缀。

### 交叉编译 Rust crate

大多数 Rust 程序链接到`std` crate，所以至少你需要一个交叉编译的`std` crate来交叉编译你的程序。获得它的最简单途径是[官方构建](http://static.rust-lang.org/dist/)。

如果您使用的是 multirust，从2016-03-08开始，您可以使用以下命令安装这些包：`multirust add-target nightly $rustc_target`。如果您使用的是rustup.rs，请使用以下命令：`rustup target add $rustc_target`。如果您不使用，请按照以下说明手动安装 crate。

你需要的压缩包是 `$date/rust-std-nightly-$rustc_target.tar.gz`，其中`$date`通常和`rustc -V`中显示的提交日期项匹配，但有时可能会相差一天或几天。

举个例子，对于`arm-unknown-linux-gnueabihf`目标和`rustc 1.8.0-nightly (3c9442fc5 2016-02-04)`，这是正确的压缩包：

    http://static.rust-lang.org/dist/2016-02-04/rust-std-beta-arm-unknown-linux-gnueabihf.tar.gz

要安装这个压缩包，请使用里面的 `install.sh`

    tar xzf rust-std-nightly-arm-unknown-linux-gnueabihf.tar.gz
    cd rust-std-nightly-arm-unknown-linux-gnueabihf
    ./install.sh --prefix=$(rustc --print sysroot)

**警告**：上面的命令将输出如下所示的消息："creating uninstall script
at /some/path/lib/rustlib/uninstall.sh"。不要运行该脚本，因为它将卸载交叉编译的标准包和本机标准包，让你无法使用Rust安装，你将无法进行原生编译。

如果由于某种原因需要卸载刚刚安装的crate，只需删除以下目录：`$(rustc --print sysroot)/lib/rustlib/$rustc_target`.

**注意**：如果您使用的是 nightly 通道，则每次更新Rust时，您都必须安装一组新的交叉编译标准包。为此，只需下载新的压缩包并像以前一样使用`install.sh`脚本。 据我所知，脚本还将负责删除旧的 crate。

## 使用 `rustc` 交叉编译

这是最简单的部分！

使用`rustc`进行交叉编译只需要在其调用中传递一些额外的标志：

+ `--target=$rustc_target`，告诉`rustc`，我们正在为`$rustc_target`交叉编译。
+ `-C linker=$gcc_prefix-gcc`，指示`rustc`使用交叉链接器而不是本机链接器（`cc`）.

接下来，测试交叉编译的示例：

+ 在主机上创建一个hello world程序

        $ cat hello.rs
        fn main() {
            println!("Hello, world!");
        }

+ 在主机上交叉编译程序

        $ rustc \
            --target=arm-unknown-linux-gnueabihf \
            -C linker=arm-linux-gnueabihf-gcc \
            hello.rs

+ 在目标上运行该程序

        $ scp hello me@arm:~
        $ ssh me@arm ./hello
        Hello, world!

## 使用 `cargo` 交叉编译

要使用 `cargo` 交叉编译，我们必须首先使用其[配置系统](http://doc.crates.io/config.html)为目标设置适当的链接器和归档器。设置后，我们只需要将 `--target`标志传递给cargo命令。
cargo配置存储在TOML文件中，我们感兴趣的关键是`target.$rustc_target.linker`。存储在此键中的值与我们在上一节中传递给`rustc`的值相同。由您决定是将此配置设为全局还是特定于项目。

我们来看一个例子：

+ 创建一个新的二进制cargo项目

    ```bash
    cargo new --bin foo
    cd foo
    ```

+ 向项目添加依赖项

    ```bash
    $ echo 'clap = "2.0.4"' >> Cargo.toml
    $ cat Cargo.toml
    [package]
    authors = ["me", "myself", "I"]
    name = "foo"
    version = "0.1.0"

    [dependencies]
    clap = "2.0.4"
    ```

+ 仅为此项目配置目标链接器和归档程序

    ```bash
    $ mkdir .cargo
    $ cat >.cargo/config <<EOF
    > [target.arm-unknown-linux-gnueabihf]
    > linker = "arm-linux-gnueabihf-gcc"
    > EOF
    ```

+ 编写应用程序

    ```bash
    $ cat >src/main.rs <<EOF
    > extern crate clap;
    >
    > use clap::App;
    >
    > fn main() {
    >     let _ = App::new("foo").version("0.1.0").get_matches();
    > }
    > EOF
    ```

+ 为目标构建项目

    ```bash
    cargo build --target=arm-unknown-linux-gnueabihf
    ```

+ 将二进制文件部署到目标

    ```bash
    scp target/arm-unknown-linux-gnueabihf/debug/foo me@arm:~
    ```

+ 在目标上运行二进制文件

    ```bash
    $ ssh me@arm ./foo -h
    foo 0.1.0

    USAGE:
            foo [FLAGS]

    FLAGS:
        -h, --help       Prints help information
        -V, --version    Prints version information
    ```

## 高级主题

### 交叉编译标准crate

现在，您只能交叉编译标准包到Rust构建系统（RBS）支持的目标。 您可以在[mk/cfg](https://github.com/rust-lang/rust/tree/3c9442fc503fe397b8d3495d5a7f9e599ad63cf6/mk/cfg)目录中找到所有受支持目标的列表（注意链接目录不是最新版本）。 从`rustc 1.8.0-nightly (3c9442fc5 2016-02-04)`开始，我看到以下支持的目标：

    $ ls mk/cfg
    aarch64-apple-ios.mk              i686-pc-windows-msvc.mk           x86_64-pc-windows-gnu.mk
    aarch64-linux-android.mk          i686-unknown-freebsd.mk           x86_64-pc-windows-msvc.mk
    aarch64-unknown-linux-gnu.mk      i686-unknown-linux-gnu.mk         x86_64-rumprun-netbsd.mk
    arm-linux-androideabi.mk          le32-unknown-nacl.mk              x86_64-sun-solaris.mk
    arm-unknown-linux-gnueabihf.mk    mipsel-unknown-linux-gnu.mk       x86_64-unknown-bitrig.mk
    arm-unknown-linux-gnueabi.mk      mipsel-unknown-linux-musl.mk      x86_64-unknown-dragonfly.mk
    armv7-apple-ios.mk                mips-unknown-linux-gnu.mk         x86_64-unknown-freebsd.mk
    armv7s-apple-ios.mk               mips-unknown-linux-musl.mk        x86_64-unknown-linux-gnu.mk
    armv7-unknown-linux-gnueabihf.mk  powerpc64le-unknown-linux-gnu.mk  x86_64-unknown-linux-musl.mk
    i386-apple-ios.mk                 powerpc64-unknown-linux-gnu.mk    x86_64-unknown-netbsd.mk
    i686-apple-darwin.mk              powerpc-unknown-linux-gnu.mk      x86_64-unknown-openbsd.mk
    i686-linux-android.mk             x86_64-apple-darwin.mk
    i686-pc-windows-gnu.mk            x86_64-apple-ios.mk

注意：如果RBS不支持您的目标，则需要为其添加对目标的支持。 我不会详细介绍添加对新目标的支持，但您可以使用[此PR](https://github.com/rust-lang/rust/pull/31078)作为参考。

注意如果您正在进行裸机编程，构建自己的内核，或者通常使用`#![no_std]`代码，那么您可能不希望（并且可能不能，因为没有操作系统）构建所有标准包，只想构建`core` crate和其他独立crate。如果是这种情况，请阅读[交叉编译`no_std`代码](#交叉编译no_std代码)部分而不是此部分。

交叉编译标准包的步骤并不复杂，但构建它们的过程确实需要很长时间，因为RBS将引导新的编译器，然后使用该编译器来交叉编译包。希望[即将推出](https://github.com/rust-lang/rust/pull/31123)的基于cargo的构建系统可以让您使用已安装的`rustc`和`cargo`来交叉编译标准包，从而提高速度。

回到说明，首先你要弄清楚你的`rustc`的提交哈希值。这列在`rustc -Vv`的输出下。 例如，这个`rustc`：

```bash
$ rustc -Vv
rustc 1.8.0-nightly (3c9442fc5 2016-02-04)
binary: rustc
commit-hash: 3c9442fc503fe397b8d3495d5a7f9e599ad63cf6
commit-date: 2016-02-04
host: x86_64-unknown-linux-gnu
release: 1.8.0-nightly
```

提交哈希值: `3c9442fc503fe397b8d3495d5a7f9e599ad63cf6`.

接下来，您需要获取Rust源代码并checkout该提交。不要省略checkout，否则你将得到编译器无法使用的包。

```bash
$ git clone https://github.com/rust-lang/rust
$ cd rust
$ git checkout $rustc_commit_hash
# Triple check the git checkout matches `rustc` commit hash
$ git rev-parse HEAD
$rustc_commit_hash
```

接下来，我们为源代码构建准备一个构建目录。

```bash
# Anywhere
$ mkdir build
$ cd build
$ /path/to/rust/configure --target=$rustc_target
```

`configure`接受许多其他配置标志，请查看`configure --help`以获取更多信息。请注意，默认情况下，没有任何标志，`configure`将准备完全优化的构建。

接下来我们开始构建：

```bash
make -j$(nproc)
```

如果在构建期间遇到此错误：

```bash
make[1]: $rbs_prefix-gcc: Command not found
```

别恐慌！

发生这种情况是因为RBS期望gcc具有针对每个目标的特定前缀，但是此前缀可能与您安装的交叉编译器的前缀不匹配。例如，在我的系统中，安装的交叉编译器是`armv7-unknown-linux-gnueabihf-gcc`，但是当为`arm-unknown-linux-gnueabihf`目标构建时，RBS期望交叉编译器被命名为`arm-none-gnueabihf-gcc`。

这可以通过一些`shim`二进制文件轻松修复：

```bash
# In the build directory
$ mkdir .shims
$ cd .shims
$ ln -s $(which $gcc_prefix-ar) $rbs_prefix-ar
$ ln -s $(which $gcc_prefix-gcc) $rbs_prefix-gcc
$ cd ..
$ export PATH=$(pwd)/.shims:$PATH
```

现在你应该可以同时调用`$gcc_prefix-gcc`和`$rbs_prefix-gcc`。例如：

```bash
# My installed cross compiler
$ armv7-unknown-linux-gnueabihf-gcc -v
Using built-in specs.
COLLECT_GCC=armv7-unknown-linux-gnueabihf-gcc
COLLECT_LTO_WRAPPER=/usr/x86_64-pc-linux-gnu/libexec/gcc/armv7-unknown-linux-gnueabihf/5.3.0/lto-wrapper
Target: armv7-unknown-linux-gnueabihf
Configured with: (...)
Thread model: posix
gcc version 5.3.0 (GCC)

# The cross compiler that the RBS expects, which is supplied by the .shims directory
$ arm-linux-gnueabihf-gcc -v
Using built-in specs.
COLLECT_GCC=armv7-unknown-linux-gnueabihf-gcc
COLLECT_LTO_WRAPPER=/usr/x86_64-pc-linux-gnu/libexec/gcc/armv7-unknown-linux-gnueabihf/5.3.0/lto-wrapper
Target: armv7-unknown-linux-gnueabihf
Configured with: (...)
Thread model: posix
gcc version 5.3.0 (GCC)
```

您现在可以使用`make -j$(nproc)`恢复构建。

如果没有意外，构建将成功完成，并且您的交叉编译包将在`$host/stage2/lib/rustlib/$rustc_target/lib`目录中可用。

```bash
# In the build directory
$ ls x86_64-unknown-linux-gnu/stage2/lib/rustlib/arm-unknown-linux-gnueabihf/lib
liballoc-db5a760f.rlib           librand-db5a760f.rlib            stamp.arena
liballoc_jemalloc-db5a760f.rlib  librbml-db5a760f.rlib            stamp.collections
liballoc_system-db5a760f.rlib    librbml-db5a760f.so              stamp.core
libarena-db5a760f.rlib           librustc_bitflags-db5a760f.rlib  stamp.flate
libarena-db5a760f.so             librustc_unicode-db5a760f.rlib   stamp.getopts
libcollections-db5a760f.rlib     libserialize-db5a760f.rlib       stamp.graphviz
libcompiler-rt.a                 libserialize-db5a760f.so         stamp.libc
libcore-db5a760f.rlib            libstd-db5a760f.rlib             stamp.log
libflate-db5a760f.rlib           libstd-db5a760f.so               stamp.rand
libflate-db5a760f.so             libterm-db5a760f.rlib            stamp.rbml
libgetopts-db5a760f.rlib         libterm-db5a760f.so              stamp.rustc_bitflags
libgetopts-db5a760f.so           libtest-db5a760f.rlib            stamp.rustc_unicode
libgraphviz-db5a760f.rlib        libtest-db5a760f.so              stamp.serialize
libgraphviz-db5a760f.so          rustlib                          stamp.std
liblibc-db5a760f.rlib            stamp.alloc                      stamp.term
liblog-db5a760f.rlib             stamp.alloc_jemalloc             stamp.test
liblog-db5a760f.so               stamp.alloc_system
```

下一节将告诉您如何在Rust安装目录中安装这些包。

### 安装交叉编译的标准包

首先，我们需要仔细查看您的Rust安装目录，您可以使用`rustc --print sysroot`获取其路径：

```bash
# 我正在使用rustup.rs，如果你使用了rustup.sh或你的发行包，你会得到一个不同的路径
# manager to install Rust
$ tree -d $(rustc --print sysroot)
~/.multirust/toolchains/nightly
├── bin
├── etc
│   └── bash_completion.d
├── lib
│   └── rustlib
│       ├── etc
│       └── $host
│           └── lib
└── share
    ├── doc
    │   └── (...)
    ├── man
    │   └── man1
    └── zsh
        └── site-functions
```

看到`lib/rustlib/$host`目录了吗？这就是您的本地包存放的地方。必须在该目录旁边安装交叉编译的包。按照上一节中的示例，以下命令将复制RBS在正确位置构建的标准包。

```bash
# In the 'build' directory
$ cp -r \
    $host/stage2/lib/rustlib/$target
    $(rustc --print sysroot)/lib/rustlib
```

最后，我们检查包是否在正确的位置。

```bash
$ tree $(rustc --print sysroot)/lib/rustlib
/home/japaric/.multirust/toolchains/nightly/lib/rustlib
├── (...)
├── uninstall.sh
├── $host
│  └── lib
│       ├── liballoc-fd663c41.rlib
│       ├── (...)
│       ├── libarena-fd663c41.so
│       └── (...)
└── $target
    └── lib
        ├── liballoc-fd663c41.rlib
        ├── (...)
        ├── libarena-fd663c41.so
        └── (...)
```

这样，您可以根据需要为多个目标安装包。 要“卸载”包，只需删除$target 目录即可。

### 目标规范文件

目标规范文件是一个JSON文件，它向Rust编译器提供有关目标的详细信息。该规范文件有五个必填字段和几个可选字段。它的所有键都是字符串，它的值是字符串或布尔值。如下所示为Cortex M3微控制器的最小目标规范文件：

``` json
{
  "0": "注意: 我将使用这些“数字”字段作为注释，但它们不应出现在这些文件中",
  "1": "接下来的五个字段是必选的",
  "arch": "arm",
  "llvm-target": "thumbv7m-none-eabi",
  "os": "none",
  "target-endian": "little",
  "target-pointer-width": "32",

  "2": "这些字段是可选的，此处并未列出所有可能的可选字段。",
  "cpu": "cortex-m3",
  "morestack": false
}
```

可以在[`src/librustc_back/target/mod.rs`](https://github.com/rust-lang/rust/blob/3c9442fc503fe397b8d3495d5a7f9e599ad63cf6/src/librustc_back/target/mod.rs#L70-L207)文件中找到所有可能键及其对编译的影响的列表（注意：链接文件不是最新版本）。

有两种方法可以将这些目标规范文件传递给`rustc`，第一种方法是通过`--target`标志传递完整路径。

```bash
rustc --target path/to/thumbv7m-none-eabi.json (...)
```

另一种方法是简单地将文件的["file stem"](http://doc.rust-lang.org/std/path/struct.Path.html#method.file_stem)传递给`--target`,但文件必须位于工作目录或`RUST_TARGET_PATH`变量指定的目录中。

```bash
# Target specification file is in the working directory
$ ls thumbv7m-none-eabi.json
thumbv7m-none-eabi.json

# Passing just the "file stem" works
$ rustc --target thumbv7m-none-eabi (...)
```

### 交叉编译`no_std`代码

使用`no_std`代码时，您只需要一些像`core`一样的独立包，而您可能正在使用自定义目标，例如一个Cortex-M微控制器，所以您的目标没有官方版本，您也不能使用RBS构建这些包。

获得交叉编译`core`包的简单解决方案是使您的程序/包依赖于[rust-libcore](https://crates.io/crates/rust-libcore)包。这将使构建`core`包作为`cargo build`过程的一部分。但是，这种方法有两个问题：

+ 病毒性：你不能让你的包依赖于另一个`no_std`包，除非它也依赖于`rust-libcore`。

+ 如果您希望您的包依赖于另一个标准包，则需要创建一个新的`rust-lib$crate`包。

没有这些问题的另一种解决方案是使用一个“sysroot”来保存交叉编译的包。 我正在[xargo](https://github.com/japaric/xargo)中实现这种方法。有关详细信息，请查看[存储库](https://github.com/japaric/xargo)。

### 常见问题

> 任何可能出错的事都会出错。  —— 墨菲定律

本节：出现问题时该怎么办。

#### 找不到crate

##### 症状

```bash
$ cargo build --target $rustc_target
error: can't find crate for `$crate`
```

##### 原因

`rustc`无法在Rust安装目录中找到交叉编译的标准包`$crate`。

##### 解法

检查[安装交叉编译的标准包](#安装交叉编译的标准包)部分，确保交叉编译的包安装在正确的位置。

#### crate 与此版本的 rustc 不兼容

##### 症状

```bash
$ cargo build --target $rustc_target
error: the crate `$crate` has been compiled with rustc $version-$channel ($hash $date), which is incompatible with this version of rustc
```

##### 原因

您安装的交叉编译标准包的版本与您的`rustc`版本不匹配

##### 解法

如果你在nightly channel安装了[官方版本](http://static.rust-lang.org/dist/)，你可能会得到错误的日期版本。尝试不同的日期。

如果您从源代码交叉编译包，那么您检出了源代码的错误提交。 您将再次构建包，但确保在正确的提交中检出存储库（它必须与`rustc -Vv`输出的commit-hash字段匹配）。

#### 未定义引用

##### 症状

```bash
$ cargo build --target $rustc_target
/path/to/some/file.c:$line: undefined reference to `$symbol`
```

##### 原因

场景如下：

+ 使用C交叉工具链“A”交叉编译标准包。
+ 然后使用C交叉工具链“B”交叉编译Rust程序，该程序也链接到上一步生成的标准包。

当工具链“A”的libc组件比工具链“B”的libc组件更新时，会出现问题。在这种情况下，用“A”交叉编译的标准包可能取决于“B”的libc中不可用的libc符号。

如果“A”的libc与“B”的libc不同，也会发生此错误。示例：工具链“A”是`mips-linux-gnu`，工具链“B”是`mips-linux-musl`。

##### 解法

如果你用[官方版本](http://static.rust-lang.org/dist/)观察到这一点，那就是一个[bug](https://github.com/rust-lang/rust/issues/30966)。它表明Rust团队必须降级他们用来构建标准包的C交叉工具链的libc组件。

如果您自己交叉编译标准包，那么如果您使用相同的C交叉工具链来构建标准包和交叉编译Rust程序，那将是理想的。

#### 无法加载库

##### 症状

```bash
# On target
$ ./hello
./hello: can't load library 'libpthread.so.0'
```

##### 原因

您的目标系统缺少共享库。您可以使用`ldd`确认：

```bash
# Or `LD_TRACE_LOADED_OBJECTS=1 ./hello` on uClibc-based OpenWRT devices
$ ldd hello
        libdl.so.0 => /lib/libdl.so.0 (0x771ba000)
        libpthread.so.0 => not found
        libgcc_s.so.1 => /lib/libgcc_s.so.1 (0x77196000)
        libc.so.0 => /lib/libc.so.0 (0x77129000)
        ld-uClibc.so.0 => /lib/ld-uClibc.so.0 (0x771ce000)
        libm.so.0 => /lib/libm.so.0 (0x77103000)
```

所有缺少的库都标有“not found”。

##### 解法

在目标系统中安装缺少的共享库。 继续上一个例子：

```bash
# target system is an OpenWRT device
$ opkg install libpthread
$ ./hello
Hello, world!
```

#### `$symbol` 未找到

##### 症状

```bash
# On target
$ ./hello
rustc: /path/to/$c_library.so: version `$symbol' not found (required by /path/to/$rust_library.so).
```

##### 原因

在交叉编译期间动态链接到二进制文件的库与安装在目标中的库之间的ABI不匹配。

##### 解法

更新/更改主机或目标上的库，使它们兼容ABI。理想情况下，主机和目标应具有相同的库版本。

**注意**： 当我在主机上说库时，我指的是`$prefix_gcc-gcc`链接到Rust程序的交叉编译库。我不是指可能安装在主机中的本机库。

#### 非法指令

##### 症状

```bash
# on target
$ ./hello
Illegal instruction
```

##### 原因

**注意**：如果程序达到内存不足（OOM）状态，也可能出现“非法指令”错误。 在某些系统中，当您点击OOM时，您还会看到“致命的运行时错误：内存不足”消息。如果您确定不是您的情况，那么这是一个交叉编译问题。

发生这种情况是因为您的程序包含目标系统不支持的[指令](https://simple.wikipedia.org/wiki/Instruction_(computer_science))。 在这个问题的可能原因中，我们有：

+ 您正在编译一个硬浮点计算目标，例如 `arm-unknown-linux-gnueabihf`，但你的目标不支持硬件浮点操作，它实际上是一个软浮点目标，例如 `arm-unknown-linux-gnueabi`。
    **解决方案**：在此示例中使用正确的三元组：`arm-unknown-linux-gnueabi`。

+ 您正在使用正确的软浮点三元组，例如 `arm-unknown-linux-gnueabi`，适用于您的目标系统。但是你的C交叉工具链是用硬浮点支持编译的，并且正在向你的二进制文件注入硬浮点指令。
    **解决方案**：获取正确的工具链，一个使用软浮点支持构建的工具链。提示：在`$gcc_prefix-gcc -v`的输出中查找标志`--with-float`。

## FAQ

### 我想为Linux，Mac和Windows构建二进制文件。如何从Linux交叉编译到Mac？

简短的回答：不要这样做。

很难在不同的操作系统之间找到交叉C工具链（和交叉编译的C库）（可能从Linux到Windows除外）。一种更简单且不易出错的方法是为这些目标本地构建，因为它们是[一级](https://doc.rust-lang.org/book/getting-started.html#tier-1)平台。 您可能无法直接访问所有这些操作系统，但这不是问题，因为您可以使用[Travis CI](https://travis-ci.org/)和[AppVeyor](https://www.appveyor.com/)等CI服务。查看我的项目[rust-everywhere](https://github.com/japaric/rust-everywhere)，了解如何做到这一点。

### 如何编译完全静态链接的Rust二进制文件？

简短的回答：`cargo build --target x86_64-unknown-linux-musl`

对于`*-*-linux-gnu*`形式的目标，`rustc` 总是生成动态链接到`glibc`和其他库的二进制文件：

```bash
$ cargo new --bin hello
$ cargo build --target x86_64-unknown-linux-gnu
$ file target/x86_64-unknown-linux-gnu/debug/hello
target/x86_64-unknown-linux-gnu/debug/hello: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /usr/x86_64-pc-linux-gnu/lib/ld-linux-x86-64.so.2, for GNU/Linux 2.6.34, BuildID[sha1]=a3fa7281e9ded30372b5131a2feb6f1e78a6f1cd, not stripped
$ ldd target/x86_64-unknown-linux-gnu/debug/hello
        linux-vdso.so.1 (0x00007fff58bf4000)
        libdl.so.2 => /usr/x86_64-pc-linux-gnu/lib/libdl.so.2 (0x00007fc4b2d3f000)
        libpthread.so.0 => /usr/x86_64-pc-linux-gnu/lib/libpthread.so.0 (0x00007fc4b2b22000)
        libgcc_s.so.1 => /usr/x86_64-pc-linux-gnu/lib/libgcc_s.so.1 (0x00007fc4b290c000)
        libc.so.6 => /usr/x86_64-pc-linux-gnu/lib/libc.so.6 (0x00007fc4b2568000)
        /usr/x86_64-pc-linux-gnu/lib/ld-linux-x86-64.so.2 (0x00007fc4b2f43000)
        libm.so.6 => /usr/x86_64-pc-linux-gnu/lib/libm.so.6 (0x00007fc4b2272000)
```

为了生成静态链接的二进制文件，Rust提供了两个目标：`x86_64-unknown-linux-musl`和`i686-unknown-linux-musl`。为这些目标生成的二进制文件静态链接到MUSL C库。示例如下：

```bash
$ cargo new --bin hello
$ cd hello
$ rustup target add x86_64-unknown-linux-musl
$ cargo build --target x86_64-unknown-linux-musl
$ file target/x86_64-unknown-linux-musl/debug/hello
target/x86_64-unknown-linux-musl/debug/hello: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=759d41b9a78d86bff9b6529d12c8fd6b934c0088, not stripped
$ ldd target/x86_64-unknown-linux-musl/debug/hello
        not a dynamic executable
```
