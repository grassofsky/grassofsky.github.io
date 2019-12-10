---
date: "2018-04-04" 
title: "fastbuild源码解析-test framework"
categories: ["fastbuild"] 
---

# fastbuild源码解析-test framework

## test framework使用来干什么的？

test framework是一款类似于gtest的单元测试框架，但是相比较而言更加的轻量级，仅用于对fastbuild中各个模块的代码进行单元测试。

## test framework使用示例

```c++
// StdStringTest.cpp
//------------------------------------------------------------------------------

// Inludes
//------------------------------------------------------------------------------
#include "TestFramework/UnitTest.h"

#include <string>

/**
 * \brief 每个一继承鱼UnitTest的类是一个testgraoup，
 *        它的内部可以实现多个测试用例
 */
class TestStdString : public UnitTest
{
private:
    // 对继承的函数进行声明，包括
    // virtual void RunTests();
    // virtual const char* GetName() const;
    DECLARE_TESTS

    void TestEmpty() const; //< 独立的测试用例
    void TestEqual() const;
};

// Register Tests
//------------------------------------------------------------------------------
// 实现成员函数，此处借用宏定义，简化具体实现
REGISTER_TESTS_BEGIN(TestStdString)
    REGISTER_TEST(TestEmpty)
    REGISTER_TEST(TestEqual)
REGISTER_TESTS_END

// 编写具体的测试用例
void TestStdString::TestEmpty() const
{
    std::string strEmpty;
    TEST_ASSERT(strEmpty.empty());
}

void TestStdString::TestEqual() const
{
    std::string strFirst = "hello";
    std::string strSecond;
    TEST_ASSERT(strFirst == strSecond);
    strSecond = strFirst;
    TEST_ASSERT(strFirst == strSecond);
}

int main()
{
    // 将testgroup注册到UnitTestManager中
    REGISTER_TESTGROUP(TestStdString);

    // 构造unitTestManager，调用RunTests来运行上述注册的testgroup
    UnitTestManager utm;
    bool allPassed = utm.RunTests();
    return allPassed ? 0 : -1;
}
```

从示例代码中可以看出testgramework使用的步骤，基本上包括：

- 从UnitTest派生出针对特定测试的子类
- 实现自己的测试用例
- 将测试用例进行注册
- 将子类进行注册
- 运行所有注册了的testgroup

## test framework具体实现

test framework实现比较简单，包括两个类，一个抽象类，`UnitTest`，对它的继承就可以实现自己的单元测试；另一个类是`UnitTestManager`，用来对`UnitTest`进行管理。

### UnitTest.h代码浏览

```c++
// UnitTest.h - interface for a unit test
//------------------------------------------------------------------------------
#pragma once

#include "UnitTestManager.h"

// UnitTest - Tests derive from this interface
//------------------------------------------------------------------------------
/**
 * UnitTest是一个抽象类，从它的实现中，可以看出链表的痕迹，成员变量
 * `UnitTest * m_NextTestGroup;`,
 * 可以从当前的UnitTest获取下一个UnitTest。
 */
class UnitTest
{
protected:
    // explicit表示这个类的构造不支持隐式转换的构造
    explicit        UnitTest() { m_NextTestGroup = nullptr; }
    // default属于c++11引入的特性
    inline virtual ~UnitTest() = default;

    // 运行当前testgroup中的所有test case
    virtual void RunTests() = 0;
    // 获取Test类的名字
    virtual const char * GetName() const = 0;

    // Run before and after each test
    virtual void PreTest() const {}
    virtual void PostTest() const {}

private:
    // UnitTestManager是UnitTest的友元类，
    // 能够访问UnitTest中的私有成员
    friend class UnitTestManager;
    // 指向下一个TestGroup
    UnitTest * m_NextTestGroup;
};

// macros
//------------------------------------------------------------------------------
// assert宏，如果传入的表达式为true，则不进行处理
// 否则输出哪个文件哪一行出错的内容
#define TEST_ASSERT( expression )                                   \
    do {                                                            \
    PRAGMA_DISABLE_PUSH_MSVC(4127)                                  \
        if ( !( expression ) )                                      \
        {                                                           \
            if ( UnitTestManager::AssertFailure(  #expression, __FILE__, __LINE__ ) ) \
            {                                                       \
                BREAK_IN_DEBUGGER;                                      \
            }                                                       \
        }                                                           \
    } while ( false );                                              \
    PRAGMA_DISABLE_POP_MSVC

// 在子类中声明函数
#define DECLARE_TESTS                                               \
    virtual void RunTests();                                        \
    virtual const char * GetName() const;

// 用于注册单个的测试用例，Begin,Register，and end
#define REGISTER_TESTS_BEGIN( testGroupName )                       \
    void testGroupName##Register()                                  \
    {                                                               \
        UnitTestManager::RegisterTestGroup( new testGroupName );    \
    }                                                               \
    const char * testGroupName::GetName() const                     \
    {                                                               \
        return #testGroupName;                                      \
    }                                                               \
    void testGroupName::RunTests()                                  \
    {                                                               \
        UnitTestManager & utm = UnitTestManager::Get();             \
        (void)utm;

#define REGISTER_TEST( testFunction )                               \
        utm.TestBegin( this, #testFunction );                       \
        testFunction();                                             \
        utm.TestEnd();

#define REGISTER_TESTS_END                                          \
    }

// 向UnitTestManager中注册特定的testgroup
#define REGISTER_TESTGROUP( testGroupName )                         \
        extern void testGroupName##Register();                      \
        testGroupName##Register();

//------------------------------------------------------------------------------
```

### UnitTestManager.h代码浏览

```c++
// UnitTestManager
//------------------------------------------------------------------------------
#pragma once

// Includes
//------------------------------------------------------------------------------
#include "Core/Env/Assert.h"
#include "Core/Env/Types.h"
#include "Core/Time/Timer.h"  //< 用于运行时间的计算

// Forward Declarations
//------------------------------------------------------------------------------
class UnitTest;

// UnitTestManager
//------------------------------------------------------------------------------
class UnitTestManager
{
public:
    UnitTestManager();
    ~UnitTestManager();

    // run all tests, or tests from a group
    bool RunTests( const char * testGroup = nullptr );

    // singleton behaviour
    #ifdef RELEASE
        static inline UnitTestManager & Get() { return *s_Instance; }
    #else
        static        UnitTestManager & Get();
    #endif
    static inline bool                  IsValid() { return ( s_Instance != 0 ); }

    // tests register (using the test declaration macros) via this interface
    static void RegisterTestGroup( UnitTest * testGroup );

    // When tests are being executed, they are wrapped with these
    void TestBegin( UnitTest * testGroup, const char * testName );
    void TestEnd();

    // TEST_ASSERT uses this interface to notify of assertion failures
    static bool AssertFailure( const char * message, const char * file, uint32_t line );

private:
    Timer       m_Timer;

    enum : uint32_t { MAX_TESTS = 1024 };  //< 支持的最大testgroup数量
    struct TestInfo
    {
        TestInfo() :m_TestGroup( nullptr ), m_TestName( nullptr ), m_Passed( false ), m_MemoryLeaks( false ), m_TimeTaken( 0.0f ) {}

        UnitTest *      m_TestGroup;      // 含有的UnitTest
        const char *    m_TestName;       // UnitTest的名字
        bool            m_Passed;         // 记录该测试是否通过
        bool            m_MemoryLeaks;    // 记录该测试是否有内存泄漏
        float           m_TimeTaken;      // 记录该测试花费的时间
    };
    static uint32_t     s_NumTests;                // 记录有多少个UnitTest
    static TestInfo     s_TestInfos[ MAX_TESTS ];  // 记录对应的TestInfo

    static UnitTestManager * s_Instance;  // UnitTestManager实例
    static UnitTest * s_FirstTest;        // 起始的UnitTest
};

//------------------------------------------------------------------------------

```

### UnitTestManager.cpp代码浏览

```c++
// UnitTestManager.cpp
//------------------------------------------------------------------------------

// Includes
//------------------------------------------------------------------------------
#include "UnitTestManager.h"
#include "UnitTest.h"

#include "Core/Env/Assert.h"
#include "Core/Env/Types.h"
#include "Core/Mem/MemTracker.h"     // 内存泄漏监控
#include "Core/Profile/Profile.h"
#include "Core/Strings/AString.h"
#include "Core/Tracing/Tracing.h"

#include <string.h>
#if defined( __WINDOWS__ )
    #include <windows.h>
#endif

// Static Data
//------------------------------------------------------------------------------
/*static*/ uint32_t UnitTestManager::s_NumTests( 0 );    //< 记录当前处理第几个Test Case
/*static*/ UnitTestManager::TestInfo UnitTestManager::s_TestInfos[ MAX_TESTS ];
/*static*/ UnitTestManager * UnitTestManager::s_Instance = nullptr;
/*static*/ UnitTest * UnitTestManager::s_FirstTest = nullptr;

// CONSTRUCTOR
//------------------------------------------------------------------------------
UnitTestManager::UnitTestManager()
{
    // manage singleton
    ASSERT( s_Instance == nullptr );
    s_Instance = this;

    // if we're running outside the debugger, we don't want
    // failures to pop up a dialog.  We want them to throw so
    // the test framework can catch the exception
    #ifdef ASSERTS_ENABLED
        if ( IsDebuggerAttached() == false )
        {
            AssertHandler::SetThrowOnAssert( true );
        }
    #endif
}

// DESTRUCTOR
//------------------------------------------------------------------------------
UnitTestManager::~UnitTestManager()
{
    #ifdef ASSERTS_ENABLED
        if ( IsDebuggerAttached() == false )
        {
            AssertHandler::SetThrowOnAssert( false );
        }
    #endif

    // free all registered tests
    UnitTest * testGroup = s_FirstTest;
    while ( testGroup )
    {
        UnitTest * next = testGroup->m_NextTestGroup;
        FDELETE testGroup;
        testGroup = next;
    }

    // manage singleton
    ASSERT( s_Instance == this );
    s_Instance = nullptr;
}

// Get
//------------------------------------------------------------------------------
#ifdef DEBUG
    /*static*/ UnitTestManager & UnitTestManager::Get()
    {
        ASSERT( s_Instance );
        return *s_Instance;
    }
#endif

// RegisterTest
//------------------------------------------------------------------------------
/*static*/ void UnitTestManager::RegisterTestGroup( UnitTest * testGroup )
{
    // first ever test? place as head of list
    if ( s_FirstTest == nullptr )
    {
        s_FirstTest = testGroup;
        return;
    }

    // link to end of list
    UnitTest * thisGroup = s_FirstTest;
loop:
    if ( thisGroup->m_NextTestGroup == nullptr )
    {
        thisGroup->m_NextTestGroup = testGroup;
        return;
    }
    thisGroup = thisGroup->m_NextTestGroup;
    goto loop;
}

// RunTests
//------------------------------------------------------------------------------
bool UnitTestManager::RunTests( const char * testGroup )
{
    // check for compile time filter
    UnitTest * test = s_FirstTest;
    while ( test )
    {
        if ( testGroup != nullptr )
        {
            // is this test the one we want?
            if ( AString::StrNCmp( test->GetName(), testGroup, strlen( testGroup ) ) != 0 )
            {
                // no -skip it
                test = test->m_NextTestGroup;
                continue;
            }
        }

        OUTPUT( "------------------------------\n" );
        OUTPUT( "Test Group: %s\n", test->GetName() );
        try
        {
            test->RunTests();
        }
        catch (...)
        {
            OUTPUT( " - Test '%s' *** FAILED ***\n", s_TestInfos[ s_NumTests - 1 ].m_TestName );
            s_TestInfos[ s_NumTests - 1 ].m_TestGroup->PostTest();
        }
        test = test->m_NextTestGroup;
    }

    OUTPUT( "------------------------------------------------------------\n" );
    OUTPUT( "Summary For All Tests\n" );

    // 在所有测试结束之后，对s_TestInfos数组进行遍历，打印出结果
    uint32_t numPassed = 0;
    float totalTime = 0.0f;
    UnitTest * lastGroup = nullptr;
    for ( size_t i=0; i<s_NumTests; ++i )
    {
        const TestInfo& info = s_TestInfos[ i ];
        if ( info.m_TestGroup != lastGroup )
        {
            OUTPUT( "------------------------------------------------------------\n" );
            OUTPUT( "             : %s\n", info.m_TestGroup->GetName() );
            lastGroup = info.m_TestGroup;
        }

        const char * status = "OK";
        if ( info.m_Passed )
        {
            ++numPassed;
        }
        else
        {
            status = ( info.m_MemoryLeaks ) ? "FAIL (LEAKS)" : "FAIL";
        }

        OUTPUT( "%12s : %5.3fs : %s\n", status, info.m_TimeTaken, info.m_TestName );
        totalTime += info.m_TimeTaken;
    }
    OUTPUT( "------------------------------------------------------------\n" );
    OUTPUT( "Passed: %u / %u (%u failures) in %2.3fs\n", numPassed, s_NumTests, ( s_NumTests - numPassed ), totalTime );
    OUTPUT( "------------------------------------------------------------\n" );

    return ( s_NumTests == numPassed );
}

// TestBegin
//------------------------------------------------------------------------------
void UnitTestManager::TestBegin( UnitTest * testGroup, const char * testName )
{
    // record info for this test
    TestInfo& info = s_TestInfos[ s_NumTests ];
    info.m_TestGroup = testGroup;
    info.m_TestName = testName;
    ++s_NumTests;

    OUTPUT( " - Test '%s'\n", testName );
    #ifdef MEMTRACKER_ENABLED
        MemTracker::Reset();
    #endif
    m_Timer.Start();

    #ifdef PROFILING_ENABLED
        ProfileManager::Start( testName );
    #endif

    testGroup->PreTest();
}

// TestEnd
//------------------------------------------------------------------------------
void UnitTestManager::TestEnd()
{
    TestInfo& info = s_TestInfos[ s_NumTests - 1 ];

    info.m_TestGroup->PostTest();

    #ifdef PROFILING_ENABLED
        ProfileManager::Stop();

        // flush the profiling buffers, but don't profile the sync itself
        // (because it leaves an outstanding memory alloc, it looks like a leak)
        ProfileManager::SynchronizeNoTag();
    #endif

    float timeTaken = m_Timer.GetElapsed();

    info.m_TimeTaken = timeTaken;

    #ifdef MEMTRACKER_ENABLED
        if ( MemTracker::GetCurrentAllocationCount() != 0 )
        {
            info.m_MemoryLeaks = true;
            OUTPUT( " - Test '%s' in %2.3fs : *** FAILED (Memory Leaks)***\n", info.m_TestName, timeTaken );
            MemTracker::DumpAllocations();
            TEST_ASSERT( false );
            return;
        }
    #endif

    OUTPUT( " - Test '%s' in %2.3fs : PASSED\n", info.m_TestName, timeTaken );
    info.m_Passed = true;
}

// Assert
//------------------------------------------------------------------------------
/*static*/ bool UnitTestManager::AssertFailure( const char * message,
                                                const char * file,
                                                uint32_t line )
{
    OUTPUT( "\n-------- TEST ASSERTION FAILED --------\n" );
    OUTPUT( "%s(%i): Assert: %s", file, line, message );
    OUTPUT( "\n-----^^^ TEST ASSERTION FAILED ^^^-----\n" );

    if ( IsDebuggerAttached() )
    {
        return true; // tell the calling code to break at the failure sight
    }

    // throw will be caught by the unit test framework and noted as a failure
    throw "Test Failed";
}

//------------------------------------------------------------------------------

```