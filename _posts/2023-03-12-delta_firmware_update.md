---
layout: post
title:  "单片机差分升级"
date:   2023-03-12 0:37:00 +0800
categories: [Embedded, MCU]
tags: [MCU]
excerpt: 记录一次为 MCU 增加差分升级功能的过程。
---

## 1. 背景

之前开发 OTA 升级功能的时候，为了升级后的版本出现严重问题后能够自动回滚（比如刚起来就挂了），在片内 FLASH 上开了三个区：运行区、下载区、备份区，每个区大小一样，后来二进制文件的大小超过了单个区的大小，不得不扩区，但是由于同时每个区都要储存一份二进制文件，三个区的存在严重限制了文件的大小。因此，了解了一下 MCU 上差分升级实现方案，使用 [bsdiff](http://www.daemonology.net/bsdiff/) 或 [JojoDiff](https://jojodiff.sourceforge.net/) 在上位机上 diff，在 MCU 上 patch。

## 2. 二进制差分工具介绍

### (1) bsdiff

这篇论文《[Naive differences of executable code](http://www.daemonology.net/papers/bsdiff.pdf)》阐述了 bsdiff 背后算法的设计思路。

产生差分文件（diff）时，对旧文件进行后缀排序，构建一个后缀数组，然后使用这个后缀数组搜索新文件，找到一系列和旧文件近似匹配的区域，最终的补丁数据主要包含三部分内容：由一系列 ADD 指令和 INSERT 指令组成的“控制区块”、由一系列近似匹配区域按字节计算的差值构成的“差分区块”（和可执行文件中未做修改的源码对应的区块相关）、由不属于近似匹配区域的字节构成的“额外区块”（和可执行文件中与被修改源码对应的区块相关）。补丁数据会比原始的文件稍微大一些，但是，“控制区块”和“差分区块”数据的可压缩的程度很高。因此，可以在输出补丁文件的时候使用合适的压缩算法对补丁数据进行压缩，最终产生的补丁文件会比较小。比如，bsdiff 工具使用了 bzip 压缩算法，在一些在 MCU 上使用 bsdiff 算法进行差分升级的方案上也看到了选用 LZ77、LZMA 的情况。

打补丁（patch）时，按“控制区块”的 ADD 指令从“差分区块”以及旧文件的相同位置分别读取一定数量的字节后按字节相加后写入新文件，或者按 INSERT 指令从“额外区块”读取一定数量的字节直接写入新文件。

具体实现细节需要查看源代码，也可以通过这篇博客《[bsdiff源码解析](https://zhuyie.github.io/posts/bsdiff-annotated/)》进行快速了解。

如 bsdiff 工具的介绍中提到的，代码实现上，抛开数据压缩算法消耗的内存，bsdiff 占用的内存较多（$max(17n,9n+m)+O(1)$），bspatch 占用的内存较小（$n+m+O(1)$），其中，n 是旧文件字节数，m 是新文件的字节数。产生差分文件通常是在上位机上完成的，打补丁是在目标机器上完成的。具体使用时，还需要根据内存资源情况选用合适的压缩算法。

在单片机上实现时，可以从这个仓库 [mendsley/bsdiff](https://github.com/mendsley/bsdiff) 克隆代码。数据压缩在 `bsdiff_stream` 的 `write` 回调函数中完成：

```c
struct bsdiff_stream
{
    void *opaque;

    void *(*malloc)(size_t size);
    void (*free)(void *ptr);
    int (*write)(struct bsdiff_stream *stream, const void *buffer, int size);
};
```

数据解压在`bspatch_stream` 的 `read` 回调函数中完成：

```c
struct bspatch_stream
{
    void *opaque;
    int (*read)(const struct bspatch_stream *stream, void *buffer, int length);
};
```

如前文提到的，补丁如果未经压缩（即 `write`、`read`不对数据进行压缩、解压），要比原始的文件稍微大一些。

### (2) JojoDiff

如上一节提到的，Bsdiff 如果不经修改直接用到嵌入式系统上，加上解压算法，需要消耗数量可观的内存。单从节省内存的角度考虑，可能[JojoDiff](https://jojodiff.sourceforge.net/)更合适一些。JojoDiff 的介绍资料比较少，我也没有探究其具体的实现细节。好消息是，[Jan Jongboom](https://github.com/janjongboom) 对 JojoDiff 生成的补丁文件格式进行了逆向工程（为了避免被 JojoDiff 使用的 GPLv3 感染，没有使用 JojoDiff 的源码），特意为嵌入式系统开发了 [JanPatch](https://github.com/janjongboom/janpatch) ，可以很好的用在单片机上。

JanPath 的主体是一个 C 头文件 `janpatch.h`，所有的代码都在这个头文件里，使用的时候，`#include "janpatch.h"` 到代码里面。使用方式是构造一个 `janpatch_ctx` 结构体，通过这个结构体完成 patch 动作。

```c
typedef struct {
    // fread/fwrite buffers
    janpatch_buffer source_buffer;
    janpatch_buffer patch_buffer;
    janpatch_buffer target_buffer;

    // function signatures
    size_t (*fread)(void*, size_t, size_t, JANPATCH_STREAM*);
    size_t (*fwrite)(const void*, size_t, size_t, JANPATCH_STREAM*);
    int    (*fseek)(JANPATCH_STREAM*, long int, int);
    long   (*ftell)(JANPATCH_STREAM*);

    // progress callback
    void   (*progress)(uint8_t);

    // the combination of the size of both the source + patch files (that's the max. the target file can be)
    long   max_file_size;
} janpatch_ctx;
```

首先是 `source`、`patch`、`target` 缓冲区。这个好处理，定义 3 个数组即可（大小与升级镜像无关），例如：

```c
#define JANPATCH_CTX_BUFFER_SIZE 1024

uint8_t source_buffer[JANPATCH_CTX_BUFFER_SIZE], patch_buffer[JANPATCH_CTX_BUFFER_SIZE], target_buffer[JANPATCH_CTX_BUFFER_SIZE];
```

然后是执行文件操作的四个函数指针，用于读取旧的 bin 文件、补丁文件，写打了补丁后生成的新 bin 文件。在 POSIX 系统上，传入 `fread`、`fwrite`、`fseek`、`ftell` 函数就行，并且将 `JANPATCH_STREAM` 定义为 `FILE`。然而，单片机上如果没有用文件系统的话，需要我们自己实现这些函数，用来操作 bin 文件对应的内存或者 flash 区块。由于有 `fseek` 和 `ftell`，我们还需要维护 bin 文件对应内存的偏移量（就像 FILE 类型的偏移一样），所以需要将 `JANPATCH_STREAM` 定义为结构体，向 `FILE` 一样，记录缓冲区的起始位置、大小以及文件指针的偏移。比如，定义一个这样的结构体，用于缓冲区在 RAM 中的情况：

```c
typedef struct {
    uint8_t     *buffer;
    size_t      size;       /* buffer 的长度 */
    long int    position;   /* 0：文件开头，size：文件结尾*/
}Stream_t;


#define JANPATCH_STREAM Stream_t

#include "janpatch.h" // 里面用到了 JANPATCH_STREAM 宏，所以要在 JANPATCH_STREAM 宏定义之后
```

四个函数指针模仿标准库函数 `fread`、`fwrite`、`fseek`、`ftell` 的行为，并按照对应的函数原型定义即可：

```c
/**
 * @brief 从 JANPATCH 流缓冲区读取数据
 * 
 * @param dst 指向目标区域的指针
 * @param sz 单个数据单元的大小
 * @param n 要读取的数据单元的数量
 * @param js JANPATCH 流指针
 * @return size_t 实际上读取到的数据单元的数量
 */
size_t janpatch_stream_read(void *dst, size_t sz, size_t n, JANPATCH_STREAM *js)
{
    size_t require = 0, read = 0, remain = 0;

    if ((dst == NULL) || (js == NULL))
    {
        return 0;
    }

    require = sz * n;
    remain = js->size - js->position;
    read = (require <= remain) ? require : remain;

    memcpy(dst, (uint8_t *)(js->buffer) + js->position, read);

    js->position += read;

    return read;
}

/**
 * @brief 向 JANPATCH 流缓冲区写入数据
 * 
 * @param src 指向源数据区域的指针
 * @param sz 单个数据单元的大小
 * @param n 要读取的数据单元的数量
 * @param js JANPATCH 流指针
 * @return size_t 实际上写入的数据单元的数量
 */
size_t janpatch_stream_write(const void *src, size_t sz, size_t n, JANPATCH_STREAM *js)
{
    size_t require = 0, write = 0, remain = 0;

    if ((src == NULL) || (js == NULL))
    {
        return 0;
    }

    require = sz * n;
    remain = js->size - js->position;
    write = (require <= remain) ? require : remain;

    memcpy((uint8_t *)(js->buffer) + js->position, src, write);

    js->position += write;

    return write;
}

/**
 * @brief 重新设置 JANPATCH 流指针的偏移量
 * 
 * @param js JANPATCH 流指针
 * @param offset 指定的偏移量
 * @param origin 开始添加偏移量 offset 的位置（SEEK_SET：流的开始；SEEK_CUR：流当前的位置；SEEK_END：流的结束）
 * @return int 成功则返回 0，否则返回 -1。
 */
int janpatch_stream_seek(JANPATCH_STREAM *js, long offset, int origin)
{
    int ret = -1;
    long require = 0;

    if (js == NULL)
    {
        return ret;
    }

    switch (origin)
    {
    case SEEK_SET:
    {
        /* 开头 */
        if ((offset >= 0) && (offset <= js->size))
        {
            js->position = offset;
            ret = 0;
        }
        else
        {
            ret = -1;
        }
    }
    break;
    case SEEK_CUR:
    {
        /* 当前位置 */
        require = offset + js->position;
        if ((require >= 0) && (require <= js->size))
        {
            js->position = require;
            ret = 0;
        }
        else
        {
            ret = -1;
        }
    }
    break;
    case SEEK_END:
    {
        /* 结尾 */
        require = offset + (long)js->size;
        if ((offset <= 0) && (require <= js->size))
        {
            js->position = require;
            ret = 0;
        }
        else
        {
            ret = -1;
        }
    }
    break;
    default:
        ret = -1;
        break;
    }

    return ret;
}

/**
 * @brief 返回 JANPATCH 流的当前位置
 *
 * @param js 指向 JANPATCH 流的指针
 * @return long 当前位置。如果出错，返回 -1。
 */
long janpatch_stream_tell(JANPATCH_STREAM *js)
{
    long ret = -1;

    if (js == NULL)
    {
        return ret;
    }

    return js->position;
}
```

然后使用上面的内容构造 `janpatch_ctx` 结构体：

```c
janpatch_ctx ctx = {
    {source_buffer, JANPATCH_CTX_BUFFER_SIZE},
    {patch_buffer, JANPATCH_CTX_BUFFER_SIZE},
    {target_buffer, JANPATCH_CTX_BUFFER_SIZE},

    &janpatch_stream_read,
    &janpatch_stream_write,
    &janpatch_stream_seek,
    &janpatch_stream_tell,
};
```

最后调用 `janpatch` 函数，该函数的原型如下：

```c
int janpatch(janpatch_ctx ctx, JANPATCH_STREAM *source, JANPATCH_STREAM *patch, JANPATCH_STREAM *target);
```

除了 `ctx` 外，还要传入三个 `JANPATCH_STREAM` 类型数据的指针，也就是前面我们定义的结构体 `Stream_t` 类型数据的指针：

```c
JANPATCH_STREAM source = {
    .buffer = (uint8_t *)&old_bin_buffer_addr,      /* 储存旧文件的缓冲区的地址 */
    .size = (size_t)old_bin_size,                   /* 旧文件的字节数 */
    .position = 0,
};
JANPATCH_STREAM patch = {
    .buffer = (uint8_t *)&patch_bin_buffer_addr,    /* 储存 ptach 文件的缓冲区的地址 */
    .size = (size_t)patch_bin_size,                 /* patch 文件的字节数*/
    .position = 0,
};
JANPATCH_STREAM target = {
    .buffer = (uint8_t *)&target_bin_buffer_addr,   /* 储存将要生成的新文件的缓冲区的地址*/
    .size = (size_t)target_bin_buffer_size,         /* 缓冲区的大小 */
    .position = 0,
};

if (0 != janpatch(ctx, &old, &patch, &target))
{
    /* 出错后要执行的代码 */
}

int len = janpatch_stream_tell(&target);            /* 获取生成的新文件的大小 */
```

补丁文件使用 JDiff 生成。

## 3. 总结

阅读、分析 bsdiff 和 janpatch 的代码，明显能够感受到，bsdiff patch 时所需内存要比 janpatch patch 时所需的内存多，而且，选用 bsdiff 的话还需要使用合适的压缩算法，否则，patch 比原始文件还要大一些，这样的话，解压缩还要消耗一部分内存，相比于 janpatch 可以直接使用 JojoDiff 生成的 patch 文件，使用 bsdiff 还要改造其源码以加入合适的压缩算法，因此，janpatch 可能更适用 MCU 差分升级（当然，此处没有考虑 bsdiff 和 JojoDiff 生成的 patch 文件的大小，具体选用哪种方案还需要综合考虑）。