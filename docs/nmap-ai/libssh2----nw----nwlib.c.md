# `nmap\libssh2\nw\nwlib.c`

```
/*********************************************************************
 * Universal NetWare library stub.                                   *
 * written by Ulrich Neuman and given to OpenSource copyright-free.  *
 * Extended for CLIB support by Guenter Knauf.                       *
 *********************************************************************/

#ifdef NETWARE /* Novell NetWare */

#include <stdlib.h>

#ifdef __NOVELL_LIBC__
/* For native LibC-based NLM we need to register as a real lib. */
#include <errno.h>
#include <string.h>
#include <library.h>
#include <netware.h>
#include <screen.h>
#include <nks/thread.h>
#include <nks/synch.h>

typedef struct
{
  int     _errno;
  void    *twentybytes;
} libthreaddata_t;

typedef struct
{
  int         x;
  int         y;
  int         z;
  void        *tenbytes;
  NXKey_t     perthreadkey;   /* if -1, no key obtained... */
  NXMutex_t   *lock;
} libdata_t;

int         gLibId      = -1;
void        *gLibHandle = (void *) NULL;
rtag_t      gAllocTag   = (rtag_t) NULL;
NXMutex_t   *gLibLock   = (NXMutex_t *) NULL;

/* internal library function prototypes... */
int     DisposeLibraryData ( void * );
void    DisposeThreadData ( void * );
int     GetOrSetUpData ( int id, libdata_t **data, libthreaddata_t **threaddata );
# 定义一个名为 _NonAppStart 的函数，接受多个参数
int _NonAppStart( void        *NLMHandle,
                  void        *errorScreen,
                  const char  *cmdLine,
                  const char  *loadDirPath,
                  size_t      uninitializedDataLength,
                  void        *NLMFileHandle,
                  int         (*readRoutineP)( int conn,
                                               void *fileHandle, size_t offset,
                                               size_t nbytes,
                                               size_t *bytesRead,
                                               void *buffer ),
                  size_t      customDataOffset,
                  size_t      customDataSize,
                  int         messageCount,
                  const char  **messages )
{
  # 为每个应用程序数据分配一个锁
  NX_LOCK_INFO_ALLOC(liblock, "Per-Application Data Lock", 0);

  # 如果不是使用 GNU 编译器，则忽略以下未使用的参数
  # 忽略 cmdLine, loadDirPath, uninitializedDataLength, NLMFileHandle, readRoutineP, customDataOffset, customDataSize, messageCount, messages
#ifndef __GNUC__
#pragma unused(cmdLine)
#pragma unused(loadDirPath)
#pragma unused(uninitializedDataLength)
#pragma unused(NLMFileHandle)
#pragma unused(readRoutineP)
#pragma unused(customDataOffset)
#pragma unused(customDataSize)
#pragma unused(messageCount)
#pragma unused(messages)
#endif

  '''
  在这里我们处理命令行参数，向错误屏幕输出错误信息，
  执行初始化操作以及其他在能够接受调用之前需要做的事情。
  如果成功，返回非零值，NetWare Loader 将保持我们处于加载状态，
  否则加载失败并被卸载。
  '''
  # 为库的内存分配分配资源标签
  gAllocTag = AllocateResourceTag(NLMHandle, "<library-name> memory allocations", AllocSignature);

  # 如果分配资源标签失败，则向错误屏幕输出错误信息，并返回 -1
  if (!gAllocTag) {
    OutputToScreen(errorScreen, "Unable to allocate resource tag for library memory allocations.\n");
    return -1;
  }

  # 注册库，并将 DisposeLibraryData 作为参数传递
  gLibId = register_library(DisposeLibraryData);

  # 如果注册库失败，则向错误屏幕输出错误信息，并返回 -1
  if (gLibId < -1) {
    OutputToScreen(errorScreen, "Unable to register library with kernel.\n");
    return -1;
  }

  # 将 NLMHandle 赋值给 gLibHandle
  gLibHandle = NLMHandle;

  # 分配一个互斥锁给 gLibLock
  gLibLock = NXMutexAlloc(0, 0, &liblock);

  # 如果分配互斥锁失败，则执行以下操作
  if (!gLibLock) {
    # 将错误信息输出到屏幕上
    OutputToScreen(errorScreen, "Unable to allocate library data lock.\n");
    # 返回错误代码 -1
    return -1;
  }
  # 如果成功分配了库数据锁，则返回 0
  return 0;
}

/*
** 在这里清理我们分配的任何资源。资源标签是我们创建的一部分，但 NetWare 不要求我们释放它们。
*/
void _NonAppStop( void )
{
  (void) unregister_library(gLibId);
  NXMutexFree(gLibLock);
}

/*
** 这个函数不能是文件中的第一个函数，因为如果文件首先被链接，那么检查卸载函数的偏移量将是 nlmname.nlm+0，这就是告诉我们没有这个函数的方法。当检查函数是链接对象中的第一个函数时，它是模棱两可的。因此，我们将把它放在停止函数之后。
**
** 在这里，我们检查是否可以卸载自己。如果不行，我们返回一个非零值。目前，没有任何理由不允许它。
*/
int _NonAppCheckUnload( void )
{
  return 0;
}

int GetOrSetUpData(int id, libdata_t **appData, libthreaddata_t **threadData)
{
  int              err;
  libdata_t        *app_data;
  libthreaddata_t  *thread_data;
  NXKey_t          key;
  NX_LOCK_INFO_ALLOC(liblock, "Application Data Lock", 0);

  err         = 0;
  thread_data = (libthreaddata_t *) NULL;

/*
** 尝试获取调用我们的应用程序的数据。这是我们存储任何应用程序特定信息的地方，以支持调用应用程序。
*/
  app_data = (libdata_t *) get_app_data(id);

  if (!app_data) {
/*
** 这个应用程序以前没有调用过我们；设置应用程序和每个线程的数据。当然，以防这个同一个应用程序的线程同时调用我们，我们最好锁定我们的应用程序数据创建互斥锁。我们还需要在获取锁之后重新检查数据，因为我们可能是那个太迟创建数据的其他线程，而第一个线程已经创建了数据。
*/
    NXLock(gLibLock);
    # 如果应用数据不存在
    if (!(app_data = (libdata_t *) get_app_data(id))) {
      # 分配内存给应用数据
      app_data = (libdata_t *) malloc(sizeof(libdata_t));

      # 如果成功分配内存
      if (app_data) {
        # 将分配的内存清零
        memset(app_data, 0, sizeof(libdata_t));

        # 分配内存给 tenbytes
        app_data->tenbytes = malloc(10);
        # 分配互斥锁给 lock
        app_data->lock     = NXMutexAlloc(0, 0, &liblock);

        # 如果 tenbytes 或 lock 分配失败
        if (!app_data->tenbytes || !app_data->lock) {
          # 如果 lock 分配成功，则释放 lock
          if (app_data->lock)
            NXMutexFree(app_data->lock);

          # 释放 app_data 内存
          free(app_data);
          # 将 app_data 设置为 NULL
          app_data = (libdata_t *) NULL;
          # 设置错误码为内存不足
          err      = ENOMEM;
        }

        # 如果 app_data 存在
        if (app_data) {
/*
** 在这里，我们将尝试通过调用 get_app_data() 来获取的应用程序数据写入。下次调用第一个函数时，我们将得到刚刚设置的数据。我们还在这里为调用线程建立了每个线程的数据，这是我们在每个应用程序线程第一次调用我们时必须做的事情。
*/
          err = set_app_data(gLibId, app_data);

          if (err) {
            free(app_data);
            app_data = (libdata_t *) NULL;
            err      = ENOMEM;
          }
          else {
            /* 创建线程特定数据的键... */
            err = NXKeyCreate(DisposeThreadData, (void *) NULL, &key);

            if (err)                /* (没有更多的键了？) */
              key = -1;

            app_data->perthreadkey = key;
          }
        }
      }
    }

    NXUnlock(gLibLock);
  }

  if (app_data) {
    key = app_data->perthreadkey;

    if (key != -1 /* 无法创建键？没有线程数据 */
        && !(err = NXKeyGetValue(key, (void **) &thread_data))
        && !thread_data) {
/*
** 为调用线程分配每个线程的数据。无论之前是否已经有应用程序数据，这可能是新线程的第一次调用。在指针上分配20个字节并不是很重要，这只是帮助演示我们可以拥有任意复杂的每个线程数据。
*/
      thread_data = (libthreaddata_t *) malloc(sizeof(libthreaddata_t));

      if (thread_data) {
        thread_data->_errno      = 0;
        thread_data->twentybytes = malloc(20);

        if (!thread_data->twentybytes) {
          free(thread_data);
          thread_data = (libthreaddata_t *) NULL;
          err         = ENOMEM;
        }

        if ((err = NXKeySetValue(key, thread_data))) {
          free(thread_data->twentybytes);
          free(thread_data);
          thread_data = (libthreaddata_t *) NULL;
        }
      }
    }
  }

  if (appData)
    # 将 app_data 的值赋给 appData 指针所指向的内存地址
    *appData = app_data;

  # 如果 threadData 指针不为空
  if (threadData)
    # 将 thread_data 的值赋给 threadData 指针所指向的内存地址
    *threadData = thread_data;

  # 返回错误码
  return err;
# 释放库数据的函数，参数为指向数据的指针
int DisposeLibraryData( void *data )
{
  # 如果数据存在
  if (data) {
    # 获取数据结构中的 tenbytes 指针
    void *tenbytes = ((libdata_t *) data)->tenbytes;

    # 如果 tenbytes 存在，释放内存
    if (tenbytes)
      free(tenbytes);

    # 释放数据结构内存
    free(data);
  }

  # 返回 0
  return 0;
}

# 释放线程数据的函数，参数为指向数据的指针
void DisposeThreadData( void *data )
{
  # 如果数据存在
  if (data) {
    # 获取数据结构中的 twentybytes 指针
    void *twentybytes = ((libthreaddata_t *) data)->twentybytes;

    # 如果 twentybytes 存在，释放内存
    if (twentybytes)
      free(twentybytes);

    # 释放数据结构内存
    free(data);
  }
}

#else /* __NOVELL_LIBC__ */
/* 对于基于 CLib 的 NLM，似乎可以更简单一些。 */
#include <nwthread.h>

# 主函数
int main ( void )
{
  /* 在这里初始化任何全局变量... */

  /* 如果有全局初始化操作，执行同步开始操作 */
  SynchronizeStart();
  
  # 退出线程
  ExitThread (TSR_THREAD, 0);
  # 返回 0
  return 0;
}

#endif /* __NOVELL_LIBC__ */

#endif /* NETWARE */
```