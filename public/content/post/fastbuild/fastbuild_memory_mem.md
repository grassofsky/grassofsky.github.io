---
date: "2018-04-05" 
title: "fastbuild源码解析-memory-mem"
categories: ["fastbuild"] 
---

## 简介

fastbuild整个memory模块包括：
- mem。重载了operator new delete，用于内存泄漏的跟踪
- MemTracker。内存泄漏管理跟踪的具体实现
- MemDebug。
- MemPoolBlock。内存池
- SmallBlockAllocator。小片段内存分配器
`mem.h`和`mem.cpp`主要提供了一种如何对new出来的内存片段进行追踪的策略，可以用于检测c++工程的内存泄漏问题。

## 其他相关内存检测介绍的内容

- [A Cross-Platform Memory Leak Detector](http://wyw.dcweb.cn/leakage.htm)
- [一个跨平台的 C++ 内存泄漏检测器](https://www.ibm.com/developerworks/cn/linux/l-mleak2/index.html)
- [small collection of c++ utilities](https://github.com/adah1972/nvwa)

## mem.h源码解析

```c++
// Mem.h
//------------------------------------------------------------------------------
#pragma once

// Includes
//------------------------------------------------------------------------------
#include "Core/Env/Types.h"
#include "Core/Mem/MemTracker.h"

// Placement new/delete
//------------------------------------------------------------------------------
/**
 * 在c++中new关键词用于内存的分配，delete关键词用于内存的释放
 * 
 * 其实在执行：
 * MyClass * p = new MyClass;
 * 时，执行了三个步骤。
 * 1. 调用opertor new分配内存空间（此处称为operator new，能被自定义重载）
 * 2. 调用类的构造函数在分配的内存空间中构造对象（此处称为placement new，不能被自定义重载）
 * 3. 返回对应的指针
 * 
 * operator new：
 * void* operator new (size_t, std::string);
 * 
 * placement new:
 * void* operator new(size_t,oid* ptr);
 * plancement new 在执行的时候忽略地一个参数size_t，返回第二个参数，并在第二个参数指向的内存空间构造对象。
 * 
 * 因此上述过程可以展示如下：
 * MyClass* p = (MyClass*)::operator new(sizeof(MyClass));
 * new(p) MyClass();
 */
#define INPLACE_NEW new
inline void * operator new( size_t, void * ptr ) { return ptr; }
inline void * operator new[]( size_t, void * ptr ) { return ptr; }
inline void operator delete( void *, void * ) {}
inline void operator delete[]( void *, void * ) {}

// new/delete
//------------------------------------------------------------------------------
// 如果MEMTRACKER_ENABLE，使用自定义的operator new(size_t, const char*, int); 否则使用默认函数
// 同时也对alloc进行重定义
#if defined( MEMTRACKER_ENABLED )
    #define FNEW( code )        new ( __FILE__, __LINE__ ) code
    #define FNEW_ARRAY( code )  new ( __FILE__, __LINE__ ) code
    #define FDELETE             delete
    #define FDELETE_ARRAY       delete[]

    #define ALLOC( ... )        ::AllocFileLine( __VA_ARGS__, __FILE__, __LINE__ )
    #define FREE( ptr )         ::Free( ptr )
#else
    #define FNEW( code )        new code
    #define FNEW_ARRAY( code )  new code
    #define FDELETE             delete
    #define FDELETE_ARRAY       delete[]

    #define ALLOC( ... )        ::Alloc( __VA_ARGS__ )
    #define FREE( ptr )         ::Free( ptr )
#endif

// Alloc/Free
//------------------------------------------------------------------------------
void * Alloc( size_t size );
void * Alloc( size_t size, size_t alignment );
void * AllocFileLine( size_t size, const char * file, int line );
void * AllocFileLine( size_t size, size_t alignment, const char * file, int line );
void Free( void * ptr );

// global new/delete
//------------------------------------------------------------------------------
// delete与new必须相互对应
#if defined( MEMTRACKER_ENABLED )
    void * operator new( size_t size, const char * file, int line );
    void * operator new[]( size_t size, const char * file, int line );
    void operator delete( void * ptr, const char *, int );
    void operator delete[]( void * ptr, const char *, int );
#endif
void * operator new( size_t size );
void * operator new[]( size_t size );
void operator delete( void * ptr );
void operator delete[]( void * ptr );

//------------------------------------------------------------------------------
```

## mem.cpp源码解析

```c++
// Mem.cpp
//------------------------------------------------------------------------------

// Includes
//------------------------------------------------------------------------------
#include "Core/PrecompiledHeader.h"

#include "Mem.h"
#include "Core/Env/Assert.h"
#include "Core/Env/Types.h"
#include "Core/Mem/MemDebug.h"
#include "Core/Mem/MemTracker.h"
#include "Core/Mem/SmallBlockAllocator.h"

#include <stdlib.h>

// Defines
//------------------------------------------------------------------------------

// Alloc
//------------------------------------------------------------------------------
void * Alloc( size_t size )
{
    return AllocFileLine( size, sizeof( void * ), "Unknown", 0 );
}

// Alloc
//------------------------------------------------------------------------------
void * Alloc( size_t size, size_t alignment )
{
    return AllocFileLine( size, alignment, "Unknown", 0 );
}

// AllocFileLine
//------------------------------------------------------------------------------
void * AllocFileLine( size_t size, const char * file, int line )
{
    return AllocFileLine( size, sizeof( void * ), file, line );
}

// AllocFileLine
// 关于内存分配的具体实现位于这个函数中。
// 该函数用于分配内存空间，并记录对应的文件名以及行号
//------------------------------------------------------------------------------
void * AllocFileLine( size_t size, size_t alignment, const char * file, int line )
{
    void * mem( nullptr );

    #if defined( SMALL_BLOCK_ALLOCATOR_ENABLED )
        // 关于SmallBlockAllocator的实现在其对应的源码中进行介绍
        // 用于分配小的对齐的内存空间
        mem = SmallBlockAllocator::Alloc( size, alignment );
        if ( mem == nullptr )
    #endif
    {
        #if defined( __LINUX__ ) || defined( __APPLE__ )
            VERIFY( posix_memalign( &mem, alignment, size ) == 0 );
        #else
            // 返回的地址空间能够被alignment整除，即按照alignment进行对齐的
            mem = _aligned_malloc( size, alignment );
            __assume( mem );
        #endif

        // 在MemDebug进行介绍，在内存区域填充特定的内容，用于标记
        #ifdef MEM_FILL_NEW_ALLOCATIONS
            MemDebug::FillMem( mem, size, MemDebug::MEM_FILL_NEW_ALLOCATION_PATTERN );
        #endif
    }

    // 将当前的内存分配的操作记录到memtracker中
    MEMTRACKER_ALLOC( mem, size, file, line );
    (void)file; (void)line; // TODO: strip args in release

    return mem;
}

// Free
//------------------------------------------------------------------------------
void Free( void * ptr )
{
    if ( ptr == nullptr )
    {
        return;
    }

    // 在memtracker中释放原来记录的内存分配
    MEMTRACKER_FREE( ptr );

    #if defined( SMALL_BLOCK_ALLOCATOR_ENABLED )
        if ( SmallBlockAllocator::Free( ptr ) == false )
    #endif
    {
        #if defined( __LINUX__ ) || defined( __APPLE__ )
            free( ptr );
        #else
            _aligned_free( ptr );
        #endif
    }
}

// Operators
// 重载内存分配，删除操作
//------------------------------------------------------------------------------
#if defined( MEMTRACKER_ENABLED )
    void * operator new( size_t size, const char * file, int line ) { return AllocFileLine( size, file, line ); }
    void * operator new[]( size_t size, const char * file, int line ) { return AllocFileLine( size, file, line ); }
    void operator delete( void * ptr, const char *, int ) { Free( ptr ); }
    void operator delete[]( void * ptr, const char *, int ) { Free( ptr ); }
#endif
void * operator new( size_t size ) { return Alloc( size ); }
void * operator new[]( size_t size ) { return Alloc( size ); }
void operator delete( void * ptr ) { Free( ptr ); }
void operator delete[]( void * ptr ) { Free( ptr ); }

//------------------------------------------------------------------------------

```