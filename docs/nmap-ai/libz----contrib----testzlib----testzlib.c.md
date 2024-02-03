# `nmap\libz\contrib\testzlib\testzlib.c`

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <windows.h>

#include "zlib.h"


void MyDoMinus64(LARGE_INTEGER *R,LARGE_INTEGER A,LARGE_INTEGER B)
{
    // 计算两个64位整数的差值
    R->HighPart = A.HighPart - B.HighPart;
    if (A.LowPart >= B.LowPart)
        R->LowPart = A.LowPart - B.LowPart;
    else
    {
        R->LowPart = A.LowPart - B.LowPart;
        R->HighPart --;
    }
}

#ifdef _M_X64
// see http://msdn2.microsoft.com/library/twchhe95(en-us,vs.80).aspx for __rdtsc
unsigned __int64 __rdtsc(void);
void BeginCountRdtsc(LARGE_INTEGER * pbeginTime64)
{
    // 使用 __rdtsc 函数获取时间戳并存储到指定的变量中
    pbeginTime64->QuadPart=__rdtsc();
}

LARGE_INTEGER GetResRdtsc(LARGE_INTEGER beginTime64,BOOL fComputeTimeQueryPerf)
{
    LARGE_INTEGER LIres;
    unsigned _int64 res=__rdtsc()-((unsigned _int64)(beginTime64.QuadPart));
    LIres.QuadPart=res;
    // 返回时间戳的差值
    return LIres;
}
#else
#ifdef _M_IX86
void myGetRDTSC32(LARGE_INTEGER * pbeginTime64)
{
    DWORD dwEdx,dwEax;
    _asm
    {
        rdtsc
        mov dwEax,eax
        mov dwEdx,edx
    }
    pbeginTime64->LowPart=dwEax;
    pbeginTime64->HighPart=dwEdx;
}

void BeginCountRdtsc(LARGE_INTEGER * pbeginTime64)
{
    // 使用内联汇编获取时间戳并存储到指定的变量中
    myGetRDTSC32(pbeginTime64);
}

LARGE_INTEGER GetResRdtsc(LARGE_INTEGER beginTime64,BOOL fComputeTimeQueryPerf)
{
    LARGE_INTEGER LIres,endTime64;
    myGetRDTSC32(&endTime64);

    LIres.LowPart=LIres.HighPart=0;
    // 计算时间戳的差值
    MyDoMinus64(&LIres,endTime64,beginTime64);
    return LIres;
}
#else
void myGetRDTSC32(LARGE_INTEGER * pbeginTime64)
{
}

void BeginCountRdtsc(LARGE_INTEGER * pbeginTime64)
{
}

LARGE_INTEGER GetResRdtsc(LARGE_INTEGER beginTime64,BOOL fComputeTimeQueryPerf)
{
    LARGE_INTEGER lr;
    lr.QuadPart=0;
    return lr;
}
#endif
#endif

void BeginCountPerfCounter(LARGE_INTEGER * pbeginTime64,BOOL fComputeTimeQueryPerf)
{
    if ((!fComputeTimeQueryPerf) || (!QueryPerformanceCounter(pbeginTime64)))
    # 设置 pbeginTime64 指针所指向的结构体中的 LowPart 字段为当前系统运行时间的低位值
    pbeginTime64->LowPart = GetTickCount();
    # 设置 pbeginTime64 指针所指向的结构体中的 HighPart 字段为 0
    pbeginTime64->HighPart = 0;
}
DWORD GetMsecSincePerfCounter(LARGE_INTEGER beginTime64,BOOL fComputeTimeQueryPerf)
{
    LARGE_INTEGER endTime64,ticksPerSecond,ticks;
    DWORDLONG ticksShifted,tickSecShifted;
    DWORD dwLog=16+0;
    DWORD dwRet;
    if ((!fComputeTimeQueryPerf) || (!QueryPerformanceCounter(&endTime64)))
        dwRet = (GetTickCount() - beginTime64.LowPart)*1;
    else
    {
        MyDoMinus64(&ticks,endTime64,beginTime64);
        QueryPerformanceFrequency(&ticksPerSecond);


        {
            ticksShifted = Int64ShrlMod32(*(DWORDLONG*)&ticks,dwLog);
            tickSecShifted = Int64ShrlMod32(*(DWORDLONG*)&ticksPerSecond,dwLog);

        }

        dwRet = (DWORD)((((DWORD)ticksShifted)*1000)/(DWORD)(tickSecShifted));
        dwRet *=1;
    }
    return dwRet;
}

int ReadFileMemory(const char* filename,long* plFileSize,unsigned char** pFilePtr)
{
    FILE* stream;
    unsigned char* ptr;
    int retVal=1;
    stream=fopen(filename, "rb");
    if (stream==NULL)
        return 0;

    fseek(stream,0,SEEK_END);

    *plFileSize=ftell(stream);
    fseek(stream,0,SEEK_SET);
    ptr=malloc((*plFileSize)+1);
    if (ptr==NULL)
        retVal=0;
    else
    {
        if (fread(ptr, 1, *plFileSize,stream) != (*plFileSize))
            retVal=0;
    }
    fclose(stream);
    *pFilePtr=ptr;
    return retVal;
}

int main(int argc, char *argv[])
{
    int BlockSizeCompress=0x8000;
    int BlockSizeUncompress=0x8000;
    int cprLevel=Z_DEFAULT_COMPRESSION ;
    long lFileSize;
    unsigned char* FilePtr;
    long lBufferSizeCpr;
    long lBufferSizeUncpr;
    long lCompressedSize=0;
    unsigned char* CprPtr;
    unsigned char* UncprPtr;
    long lSizeCpr,lSizeUncpr;
    DWORD dwGetTick,dwMsecQP;
    LARGE_INTEGER li_qp,li_rdtsc,dwResRdtsc;

    if (argc<=1)
    {
        printf("run TestZlib <File> [BlockSizeCompress] [BlockSizeUncompress] [compres. level]\n");
        return 0;
    }

    if (ReadFileMemory(argv[1],&lFileSize,&FilePtr)==0)
    {
        // 如果读取文件出错，输出错误信息并返回 1
        printf("error reading %s\n",argv[1]);
        return 1;
    }
    // 读取文件成功，输出文件名和文件大小
    else printf("file %s read, %u bytes\n",argv[1],lFileSize);

    // 如果有第三个参数，将其转换为整数并赋值给 BlockSizeCompress
    if (argc>=3)
        BlockSizeCompress=atol(argv[2]);

    // 如果有第四个参数，将其转换为整数并赋值给 BlockSizeUncompress
    if (argc>=4)
        BlockSizeUncompress=atol(argv[3]);

    // 如果有第五个参数，将其转换为整数并赋值给 cprLevel
    if (argc>=5)
        cprLevel=(int)atol(argv[4]);

    // 计算压缩和解压缩缓冲区的大小
    lBufferSizeCpr = lFileSize + (lFileSize/0x10) + 0x200;
    lBufferSizeUncpr = lBufferSizeCpr;

    // 分配压缩缓冲区的内存
    CprPtr=(unsigned char*)malloc(lBufferSizeCpr + BlockSizeCompress);

    // 开始计时
    BeginCountPerfCounter(&li_qp,TRUE);
    dwGetTick=GetTickCount();
    BeginCountRdtsc(&li_rdtsc);
    {
        // 创建压缩流对象
        z_stream zcpr;
        int ret=Z_OK;
        long lOrigToDo = lFileSize;
        long lOrigDone = 0;
        int step=0;
        // 初始化压缩流对象
        memset(&zcpr,0,sizeof(z_stream));
        deflateInit(&zcpr,cprLevel);

        // 设置压缩流对象的输入和输出
        zcpr.next_in = FilePtr;
        zcpr.next_out = CprPtr;

        // 执行压缩
        do
        {
            long all_read_before = zcpr.total_in;
            zcpr.avail_in = min(lOrigToDo,BlockSizeCompress);
            zcpr.avail_out = BlockSizeCompress;
            ret=deflate(&zcpr,(zcpr.avail_in==lOrigToDo) ? Z_FINISH : Z_SYNC_FLUSH);
            lOrigDone += (zcpr.total_in-all_read_before);
            lOrigToDo -= (zcpr.total_in-all_read_before);
            step++;
        } while (ret==Z_OK);

        // 获取压缩后的大小
        lSizeCpr=zcpr.total_out;
        // 结束压缩
        deflateEnd(&zcpr);
        // 停止计时
        dwGetTick=GetTickCount()-dwGetTick;
        dwMsecQP=GetMsecSincePerfCounter(li_qp,TRUE);
        dwResRdtsc=GetResRdtsc(li_rdtsc,TRUE);
        // 输出压缩结果信息
        printf("total compress size = %u, in %u step\n",lSizeCpr,step);
        printf("time = %u msec = %f sec\n",dwGetTick,dwGetTick/(double)1000.);
        printf("defcpr time QP = %u msec = %f sec\n",dwMsecQP,dwMsecQP/(double)1000.);
        printf("defcpr result rdtsc = %I64x\n\n",dwResRdtsc.QuadPart);
    }

    // 重新分配压缩缓冲区的内存
    CprPtr=(unsigned char*)realloc(CprPtr,lSizeCpr);
    // 分配解压缩缓冲区的内存
    UncprPtr=(unsigned char*)malloc(lBufferSizeUncpr + BlockSizeUncompress);

    // 开始计时
    BeginCountPerfCounter(&li_qp,TRUE);
    # 获取当前系统时间，用于计算程序执行时间
    dwGetTick=GetTickCount();
    # 开始计算 CPU 周期计数
    BeginCountRdtsc(&li_rdtsc);
    {
        # 创建用于解压缩的 z_stream 对象
        z_stream zcpr;
        # 定义变量并初始化
        int ret=Z_OK;
        long lOrigToDo = lSizeCpr;
        long lOrigDone = 0;
        int step=0;
        # 将 zcpr 对象的内存清零
        memset(&zcpr,0,sizeof(z_stream));
        # 初始化解压缩对象
        inflateInit(&zcpr);

        # 设置输入和输出缓冲区
        zcpr.next_in = CprPtr;
        zcpr.next_out = UncprPtr;

        # 开始解压缩
        do
        {
            # 记录当前已读取的输入字节数
            long all_read_before = zcpr.total_in;
            # 设置输入和输出缓冲区的大小
            zcpr.avail_in = min(lOrigToDo,BlockSizeUncompress);
            zcpr.avail_out = BlockSizeUncompress;
            # 执行解压缩
            ret=inflate(&zcpr,Z_SYNC_FLUSH);
            # 更新已处理的输入和剩余的输入大小
            lOrigDone += (zcpr.total_in-all_read_before);
            lOrigToDo -= (zcpr.total_in-all_read_before);
            # 记录解压缩步骤
            step++;
        } while (ret==Z_OK);

        # 记录解压缩后的数据大小
        lSizeUncpr=zcpr.total_out;
        # 结束解压缩
        inflateEnd(&zcpr);
        # 计算程序执行时间
        dwGetTick=GetTickCount()-dwGetTick;
        # 获取自系统启动以来的毫秒数
        dwMsecQP=GetMsecSincePerfCounter(li_qp,TRUE);
        # 获取 CPU 周期计数的结果
        dwResRdtsc=GetResRdtsc(li_rdtsc,TRUE);
        # 打印解压缩结果信息
        printf("total uncompress size = %u, in %u step\n",lSizeUncpr,step);
        printf("time = %u msec = %f sec\n",dwGetTick,dwGetTick/(double)1000.);
        printf("uncpr  time QP = %u msec = %f sec\n",dwMsecQP,dwMsecQP/(double)1000.);
        printf("uncpr  result rdtsc = %I64x\n\n",dwResRdtsc.QuadPart);
    }

    # 比较解压缩后的数据和原始文件数据是否一致
    if (lSizeUncpr==lFileSize)
    {
        if (memcmp(FilePtr,UncprPtr,lFileSize)==0)
            # 打印比较结果
            printf("compare ok\n");
    }

    # 返回执行结果
    return 0;
# 闭合前面的函数定义
```