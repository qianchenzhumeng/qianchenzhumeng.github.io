---
layout: post
title:  "使用 cmocka 进行单元测试"
date:   2021-02-23 21:46:00 +0800
categories: [C, Unit Testing]
tags: [Unit Testing Framework]
---

## 1. cmocka 介绍

cmocka 是一款简洁的 C 单元测试框架，支持打桩。它只依赖 C 标准库，可以运行在多种平台上（包括嵌入式环境）。

## 2. 安装 cmocka

从 [cmocka.org](https://cmocka.org) 下载 cmocka 安装包或源码，例如，linux 上下载 [cmocka-1.1.3.tar.xz](https://cmocka.org/files/1.1/cmocka-1.1.3.tar.xz) 并解压：

```bash
wget https://cmocka.org/files/1.1/cmocka-1.1.3.tar.xz
tar -xvJf cmocka-1.1.3.tar.xz
```

按照源码包中的 `INSTALL.md` 内的指导进行编译安装，例如：

```bash
cd cmocka-1.1.3
mkdir build
cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE=Debug ..
make
sudo make install
```

## 3. 使用 cmocka

### (1)一个简单的测试用例

[cmocka.org](https://cmocka.org) 上有使用教程 [Unit testing and mocking with cmocka](https://cmocka.org/talks/cmocka_unit_testing_and_mocking.pdf)，里面介绍了如何用 cmocka 进行单元测试以及如何打桩（教程第 14 页里面有一处头文件包含错误，`stdint.h` 被误写为 `sdtint.h` ）。

```c
/* example/simple_test.c */
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <stdint.h>
#include <cmocka.h>

/* A test case that does nothing and succeeds. */
static void null_test_success(void **state) {
    (void) state; /* unused */
}

int main(void) {
    const struct CMUnitTest tests[] = {
        cmocka_unit_test(null_test_success),
    };

    return cmocka_run_group_tests(tests, NULL, NULL);
}
```

编译时需要链接 cmocka 库，例如：

```bash
cd example
gcc simple_test.c -lcmocka
./a.out
```

### (2) setup 和 teardown 的用法

setup 函数和 teardown 函数分别在测试用例前、后执行。setup 用来做一些执行测试用例前的准备工作，例如，申请内存、打开文件；teardown 用来做一些执行测试用例后的清理工作，例如，释放内存、关闭文件。使用 setup 和 teardown 的好处是不用在每个测试用例中写重复的代码。

cmocka允许为每个测试用例指定 setup 和 teardown 函数。setup 函数通过宏 `cmocka_unit_test_setup` 或 `cmocka_unit_test_setup_teardown` 指定，teardown 函数通过宏 `cmocka_unit_test_teardown` 或 `cmoka_unit_test_setup_teardown` 指定，即使测试用例失败，teardown 函数也会执行。

setup 和 teardown 的函数原型：

```c
/* Function prototype for setup and teardown functions. */
typedef int (*CMFixtureFunction)(void **state);
```

示例程序：

```c
/* main.c */
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmocka.h>
#include <stdio.h>

int setup(void **state) {
    printf("setup...\n");
    return 0;
}

int teardown(void **state) {
    printf("teardown...\n");
    return 0;
}

void test_case(void **state) {
    printf("test...\n");
    // assert_int_equal(3, 4); /* 如果这句生效，该测试用例会失败，但是指定的 teardown 函数仍然会运行 */
}

int main(int argc, char *argv[]) {
    const struct CMUnitTest tests[] = {
        cmocka_unit_test_setup_teardown(test_case, setup, teardown),
    };
    return cmocka_run_group_tests(tests, NULL, NULL);
}
```

编译运行：

```bash
gcc main.c -lcmocka
./a.out
```

运行结果：

> [==========] Running 1 test(s).
> [ RUN      ] test_case
> setup...
> test...
> teardown...
> [       OK ] test_case
> [==========] 1 test(s) run.
> [  PASSED  ] 1 test(s).

cmocka 源码包内的 tests/test_fixtures.c 中有详细的使用示例。

可以用 state 指针传递资源。例如，setup 函数内部申请内存，通过 state 指针将内存地址传给测试函数和 teardown 函数。

```c
/* main.c */
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmocka.h>
#include <stdlib.h>

static int setup(void **state) {
    int *answer = malloc(sizeof(int));

    assert_non_null(answer);
    *answer = 42;
    *state = answer;

    return 0;
}

static int teardown(void **state) {
    free(*state);

    return 0;
}

static void int_test_success(void **state) {
    int *answer = *state;

    assert_int_equal(*answer, 42);
}

int main(int argc, char *argv[]) {
    const struct CMUnitTest tests[] = {
        cmocka_unit_test_setup_teardown(int_test_success, setup, teardown),
    };
    return cmocka_run_group_tests(tests, NULL, NULL);
}
```

源码包的 `tests` 和 `example` 目录下有丰富的使用示例，可以进行参考。

## 4. 故障解决

### (1) 编译源码时 cmake 报错

> CMake Error: CMake was unable to find a build program corresponding to "Unix Makefiles".  CMAKE_MAKE_PROGRAM is not set.
>   You probably need to select a different build tool.
> CMake Error: CMAKE_C_COMPILER not set, after EnableLanguage

我的环境是 WSL（Windows Subsystem For Linux，Ubuntu 20.04.1 LTS），安装 `cmocka-1.1.5` 时，cmake会报错，尝试将 `CMAKE_MAKE_PROGRAM` 设置为 make 无济于事，换了 `cmocka-1.1.3` 后可以顺利编译。

### (2) 编译测试代码时找不到 cmocka 相关的符号

> simple_test.c:(.text+0x77): undefined reference to `_cmocka_run_group_tests'

链接时，需要使用选项 `-lcmocka` 链接 cmocka 库。