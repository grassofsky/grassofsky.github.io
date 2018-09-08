---
date: "2018-04-10" 
title: "fastbuild源码解析-memory-memdebug"
categories: ["fastbuild"] 
---

## 简介

fastbuild整个memory模块包括：
- mem。重载了operator new delete，用于内存泄漏的跟踪
- MemTracker。内存泄漏管理跟踪的具体实现
- MemDebug。
- MemPoolBlock。内存池
- SmallBlockAllocator。小片段内存分配器
`MemDebug.h`和`MemDebug.cpp`对内存片段中的内容进行了标记，可以通过对内存的访问，可以发现该内存片段是被新分配的，还是已经被free不用。分别用到了如下的pattern：`MEM_FILL_NEW_ALLOCATION_PATTERN`和`MEM_FILL_FREED_ALLOCATION_PATTERN`。

## MemDebug.h源码分析

```c++
// MemDebug.h
//------------------------------------------------------------------------------
#pragma once

// Includes
//------------------------------------------------------------------------------
#include "Core/Env/Types.h"

// 通过宏定义的方式来确定是不是需要使用MEM_DEBUG功能
#if defined( DEBUG )
    #define MEM_DEBUG_ENABLED
#endif
#if defined( MEM_DEBUG_ENABLED )
    #define MEM_FILL_NEW_ALLOCATIONS
    #define MEM_FILL_FREED_ALLOCATIONS  // Will be applied where possible
#endif

// MemDebug
//------------------------------------------------------------------------------
#if defined( MEM_DEBUG_ENABLED )
    class MemDebug
    {
    public:
        // 作者在注释中，给出了MemDebug的三种用途，目前对该注释的意思还不是很理解?
        // Patterns used are:
        // - Signalling floats
        // - Unaligned
        // - Unlikely to be valid addresses
        static const uint32_t MEM_FILL_NEW_ALLOCATION_PATTERN = 0x7F8BAAAD;
        static const uint32_t MEM_FILL_FREED_ALLOCATION_PATTERN = 0x7F8BDDDD;

        // 在内存中打上标记
        static void FillMem( void * ptr, const size_t size, const uint32_t pattern );
    };
#endif

//------------------------------------------------------------------------------
```

## MemDebug.cpp

```c++
// MemDebug.cpp
//------------------------------------------------------------------------------

// Includes
//------------------------------------------------------------------------------
#include "Core/PrecompiledHeader.h"

#include "MemDebug.h"

#if defined( MEM_DEBUG_ENABLED )

// Core
#include "Core/Env/Assert.h"

// FillMem
//------------------------------------------------------------------------------
void MemDebug::FillMem( void * ptr, const size_t size, const uint32_t pattern )
{
    // 该方法只适用于地址是32位对齐的
    // this function assumes at least 32bit alignment
    ASSERT( uintptr_t( ptr ) % sizeof( uint32_t ) == 0 );

    // fill whole words
    // 将能够为sizeof(unit32_t)整除的部分用pattern进行赋值
    const size_t numWords = size / sizeof( uint32_t );
    uint32_t * it = static_cast< uint32_t * >( ptr );
    const uint32_t * end = it + numWords;
    while ( it != end )
    {
        *it = pattern;
        ++it;
    }

    // fill remaining bytes
    // 填充剩余的部分
    const size_t remainder =  size - ( numWords * sizeof( uint32_t ) );
    if ( remainder )
    {
        // assuming little-endian format
        // 此处假设内存是以小端结尾的格式
        char bytes[ 3 ] = { (char)( ( pattern & 0x000000FF ) ),
                            (char)( ( pattern & 0x0000FF00 ) >> 8 ),
                            (char)( ( pattern & 0x00FF0000 ) >> 16 ) };
        const char * b = bytes;
        char * cit = static_cast< char * >( static_cast< void * >( it ) );
        // 此处没有break，利用了switch case在没有break的情况下，
        // 会继续运行下一个case的特性
        switch( remainder )
        {
            case 3: *cit = *b; ++cit; ++b;
            case 2: *cit = *b; ++cit; ++b;
            case 1: *cit = *b;
        }
    }
}

//------------------------------------------------------------------------------
#endif // MEM_DEBUG_ENABLED
```