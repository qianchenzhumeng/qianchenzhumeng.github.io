---
layout: post
title:  "Windows 下使用 Rust 和 GTK4 创建 GUI 程序"
date:   2022-03-26 13:50:00 +0800
categories: [Rust]
tags: [Rust]
excerpt: 本文主要介绍如何利用 Rust 和 GTK4 在 Windows 上进行 GUI 程序开发。
---

## 1. 准备开发环境

### (1) 安装 GUN 工具链

MinGW 是在 Windows 上用于 Windows 原生应用开发的 GNU 工具链，安装方式有两种，一种是使用 MinGW 安装管理器进行安装，另一种是通过 MSYS2 进行安装。相比于 MinGW 安装器，MSYS2 提供的功能更丰富，可以访问 MSYS2 的官方网站[[1](https://www.msys2.org/)]获取具体的介绍信息。

MSYS2 的官网首页给出了 MSYS2 的安装方式，其中包含了 MSYS2 安装好之后，使用内部集成的 `pacman` 包管理器安装 `mingw-w64` 的步骤。

将 MinGW 的如下目录添加到环境变量中：

- C:\msys64\mingw64\bin
- C:\msys64\mingw64\include
- C:\msys64\mingw64\lib

之后将 rust 的工具链切换为 gnu 工具链：

```powershell
rustup toolchain install stable-x86_64-pc-windows-gnu
rustup default stable-x86_64-pc-windows-gnu
```

需要注意的是，如果 cargo 的安装路径含有中文，编译时 gcc 可能无法识别 rust 库路径。

### (2) 安装 GTK 4 及 Glade 界面编辑器

打开 MSYS2，使用 `pacman` 安装 GTK4，pkgconf，gcc 以及 Glade 界面编辑器：

```bash
pacman -S mingw-w64-x86_64-gtk4 mingw-w64-x86_64-pkgconf mingw-w64-x86_64-gcc mingw-w64-x86_64-glade
```

## 2. 创建第一个 GUI 程序

新建项目：

```powershell
cargo new gtk-app
```

访问 [https://crates.io/crates/gtk4](https://crates.io/crates/gtk4) 查看 `gtk4` crates 的版本（例如，0.4.7），在配置文件中添加对应的依赖：

```toml
gtk4 = "0.4.7"
```

`gtk-app/src/main.rs` 中输入以下内容：

```rust
use gtk4::prelude::*;
use gtk4::Application;

fn main() {
    // Create a new application
    let app = Application::builder()
        .application_id("org.gtk-rs.example")
        .build();

    // Run the application
    app.run();
}
```

## 3. 使用 Glade 界面编辑器

打开 Glade，创建一个应用窗体，将设计文件保存为 `main_window.glade`，放到 `gtk-app/resources` 目录内。

`gtk-app/resources/main_window.glade` 文件内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- Generated with glade 3.38.2 -->
<interface>
  <requires lib="gtk+" version="3.4"/>
  <object class="GtkApplicationWindow" id="window">
    <property name="can-focus">False</property>
    <property name="title" translatable="yes">Hello World!</property>
    <child>
      <placeholder/>
    </child>
  </object>
</interface>
```

修改 `gtk-app/src/main.rs`，从设计文件中创建窗体：

```rust
use gtk4::prelude::*;
use gtk4::{Application, ApplicationWindow, Builder};

fn build_ui(plication: &Application) {
    let glade_src = include_str!("../resources/main_window.glade");
    let builder = Builder::from_string(glade_src);
    let window: ApplicationWindow = builder.object("window").expect("Couldn't get window");
    window.set_application(Some(plication));
    window.show();
}

fn main() {
    let app = Application::builder()
        .application_id("org.get-rs.examples")
        .build();

    app.connect_activate(build_ui);

    app.run();
}
```

实测使用 Glade 生成的 xml 文件中，有些属性或标签是无法被 `gtk4::Builder` 识别的（例如 `GtkApplicationWindow.window-position` 属性和 `packing` 标签），会导致程序无法运行。

## 4. 使用资源绑定

在目录下创建 `gtk-app/resources/resources.xml` 文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<gresources>
  <gresource prefix="/example">
    <file>main_window.glade</file>
  </gresource>
</gresources>
```

进入 `gtk-app/resources` 目录，运行如下命令编译资源：

```powershell
glib-compile-resources.exe .\resources.xml
```

同目录下会生成 `resources.gresource` 文件。

在项目根目录下创建构建脚本 `build.rs`，以便执行 `Cargo build` 或 `Cargo run` 命令时自动编译资源：

```rust
use std::process::Command;

fn main() {
    let _ = Command::new("cmd")
                        .args(&["/C", "cd resources && glib-compile-resources resources.xml"])
                        .output()
                        .expect("failed to execute process");
}
```

在 `Cargo.toml` 中增加依赖：

```
gio = "0.15.10"
```

`src/main.rs` 内容如下：

```rust
use gtk4::prelude::*;
use gtk4::{Application, ApplicationWindow, Builder};
use gio;

fn build_ui(plication: &Application) {
    let builder = Builder::from_resource("/example/main_window.glade");
    let window: ApplicationWindow = builder.object("window").expect("Couldn't get window");
    window.set_application(Some(plication));
    window.show();
}


fn main() {
    let app = Application::builder()
        .application_id("org.get-rs.examples")
        .build();

    app.connect_activate(build_ui);

    // 加载资源
    let res = gio::Resource::load("resources/resources.gresource").unwrap();
    gio::resources_register(&res);

    app.run();
}
```

## 5. 总结

本来想使用 Glade 完成界面设计，但是使用过程中发现 Glade 生成的 xml 文件中，并不是所有的属性和标签都能被 Gtk4:Builer 识别。除此以外，编译生成的可执行文件启动速度也表较慢。

因此，个人感觉，使用 Rust 和 GTK4 组合进行 Windows 上 GUI 开发，不是个很好的方案。

## 参考

[1] [MSYS2](https://www.msys2.org/)

[2] [GUI development with Rust and GTK 4](https://gtk-rs.org/gtk4-rs/stable/latest/book)

[3] [Getting Started With Rust and GTK](https://blog.sb1.io/getting-started-with-rust-and-gtk/?msclkid=dd878f97acb911ec93ffe9c1b8ba854f)
