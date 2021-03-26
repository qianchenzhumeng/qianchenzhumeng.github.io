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

### (1) 一个简单的测试用例

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

### (2) setup 和 teardown

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

```
[==========] Running 1 test(s).
[ RUN      ] test_case
setup...
test...
teardown...
[       OK ] test_case
[==========] 1 test(s) run.
[  PASSED  ] 1 test(s).
```

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

### (3) 打桩（mocking）

打桩（mocking）是在链接时完成的。使用 GNU 工具链时，需要使用链接选项 `--warp=<symbol>`，symbol 为要打桩的函数，链接器接收到该选项后，会把符号 `<symbol>` 解析成 `__wrap_<symbol>`。

从一个简单的例子入手。

```c
/* test.c */
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmocka.h>

int get_value() {
    return 1;
}

int add_one() {
    int v;
    v = get_value();
    return v + 1;
}

static void add_test(void **state) {
    assert_int_equal(add_one(), 2);
}

int main(int argc, char *argv[]) {
    const struct CMUnitTest tests[] = {
        cmocka_unit_test(add_test),
    };

    return cmocka_run_group_tests(tests, NULL, NULL);
}
```

用如下命令编译并运行，测试用例可以通过。

```bash
gcc test.c -lcmocka && ./a.out
```

以 mock `get_value` 函数为例，需要删除测试文件中 `get_value` 函数的定义，并编写桩函数 `__wrap_get_value`，返回不同的值：

```c
/* test_mocking.c */
#include <stdarg.h>
#include <stddef.h>
#include <setjmp.h>
#include <cmocka.h>

int get_value();
int __wrap_get_value() {
    int v;
    v = mock_type(int);
    return v;
}

int add_one() {
    int v;
    v = get_value();
    return v + 1;
}

static void add_test(void **state) {
    (void)state;
    int a;

    will_return(__wrap_get_value, 3);

    a = add_one();
    assert_int_equal(a, 4);
}

int main(int argc, char *argv[]) {
    const struct CMUnitTest tests[] = {
        cmocka_unit_test(add_test),
    };

    return cmocka_run_group_tests(tests, NULL, NULL);
}
```

使用 `-Wl` 选项将选项 `--wap=get_value` 传给链接器：

```bash
gcc get_value.c test_mocking.c -I. -Wl,--wrap=get_value -lcmocka && ./a.out
```

反汇编 a.out，可以看到，`add_one` 调用的是 `__wrap_get_value` 函数：

```assembly
0000000000400761 <add_one>:
; ...
  40076e:       e8 0b 00 00 00          callq  40077e <__wrap_get_value>
; ...
```

链接器选项 `--wap=get_value` 只会将未定义的符号 `symbol` 解析成 `__wrap_symbol`，所谓的未定义的符号应该是指编译单元内的，即一个文件内。如果 `get_value` 的定义和桩函数 `__wrap_get_valu` 的定义出现在同一个文件中，通过该选项无法完成符号替换，测试用例不通过 [[1]](https://stackoverflow.com/questions/13961774/gnu-gcc-ld-wrapping-a-call-to-symbol-with-caller-and-callee-defined-in-the-sam)。

```c
/* test_mocking_failed.c */
/* ... */
int get_value() {
    return 1;
}
/* ... */
```

```bash
gcc test_mocking_failed.c -I. -Wl,--wrap=get_value -lcmocka && ./a.out
```

测试用例会失败。

反汇编 a.out，可以看到，`add_one` 调用的仍然是 `get_value` 函数：

```bash
objdump -s -d a.out
```

```assembly
0000000000400761 <add_one>:
; ...
  40076e:       e8 e3 ff ff ff          callq  400756 <get_value>
; ...
```

### (4) 动态内存分配

cmocka 提供了内存申请和释放函数，test_malloc、test_calloc、test_realloc 以及 test_free，分别对应 C 库的 malloc、calloc、realloc 以及 free。

以 test_malloc 和 test_free 为例，它们在 C 库对应的函数上进行了封装，会以链表记录内存申请和释放情况，使用过程中一旦发生内存泄漏，涉及的用例会被标记为失败。需要注意的是，被测函数中使用 test_malloc 申请的内存，必须在被测函数运行结束前释放，否则该测试用例会判定为因内存泄漏而失败。例如，被测函数中申请内存，teardown 释放对应内存的情况是不被允许的。setup 中申请的内存可以在 teardown 中释放。

## 4. 代码覆盖率

使用 gcov 和 lcov 查看代码覆盖率。

```bash
mkdir code_coverage
cd code_coverage
gcc -coverage -O0 -o test_mocking ../test_mocking.c -Wl,--wrap=get_value -lcmocka
./test_mocking
gcov ../test_mocking.c -o .
# 使用 lcov 收集当前目录下的覆盖率数据，将结果储存在 test_mocking.info 中
lcov -d . -t test_mocking -o test_mocking.info -b . -c
# 为 test_mocking.info 中的覆盖率数据生成 html 文档
genhtml -o output test_mocking.info
```

用浏览器打开 output/index.html 即可查看代码覆盖率报告。

## 5. 故障解决

### (1) 编译源码时 cmake 报错

> CMake Error: CMake was unable to find a build program corresponding to "Unix Makefiles".  CMAKE_MAKE_PROGRAM is not set.
>   You probably need to select a different build tool.
> CMake Error: CMAKE_C_COMPILER not set, after EnableLanguage

我的环境是 WSL（Windows Subsystem For Linux，Ubuntu 20.04.1 LTS），安装 `cmocka-1.1.5` 时，cmake会报错，尝试将 `CMAKE_MAKE_PROGRAM` 设置为 make 无济于事，换了 `cmocka-1.1.3` 后可以顺利编译。

### (2) 编译测试代码时找不到 cmocka 相关的符号

> simple_test.c:(.text+0x77): undefined reference to `_cmocka_run_group_tests'

链接时，需要使用选项 `-lcmocka` 链接 cmocka 库。

## 参考

[1] [https://stackoverflow.com/questions/13961774/gnu-gcc-ld-wrapping-a-call-to-symbol-with-caller-and-callee-defined-in-the-sam](https://stackoverflow.com/questions/13961774/gnu-gcc-ld-wrapping-a-call-to-symbol-with-caller-and-callee-defined-in-the-sam)