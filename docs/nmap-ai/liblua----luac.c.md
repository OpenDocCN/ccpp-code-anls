# `nmap\liblua\luac.c`

```cpp
/*
** $Id: luac.c $
** Lua compiler (saves bytecodes to files; also lists bytecodes)
** See Copyright Notice in lua.h
*/

#define luac_c
#define LUA_CORE

#include "lprefix.h"

#include <ctype.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "lua.h"
#include "lauxlib.h"

#include "ldebug.h"
#include "lobject.h"
#include "lopcodes.h"
#include "lopnames.h"
#include "lstate.h"
#include "lundump.h"

// 声明函数 PrintFunction，用于打印函数信息
static void PrintFunction(const Proto* f, int full);
#define luaU_print    PrintFunction

#define PROGNAME    "luac"        /* default program name */
#define OUTPUT        PROGNAME ".out"    /* default output file */

static int listing=0;            /* list bytecodes? */
static int dumping=1;            /* dump bytecodes? */
static int stripping=0;            /* strip debug information? */
static char Output[]={ OUTPUT };    /* default output file name */
static const char* output=Output;    /* actual output file name */
static const char* progname=PROGNAME;    /* actual program name */
static TString **tmname;

// 打印致命错误信息并退出程序
static void fatal(const char* message)
{
 fprintf(stderr,"%s: %s\n",progname,message);
 exit(EXIT_FAILURE);
}

// 打印无法完成操作的错误信息并退出程序
static void cannot(const char* what)
{
 fprintf(stderr,"%s: cannot %s %s: %s\n",progname,what,output,strerror(errno));
 exit(EXIT_FAILURE);
}

// 打印用法信息并退出程序
static void usage(const char* message)
{
 if (*message=='-')
  fprintf(stderr,"%s: unrecognized option '%s'\n",progname,message);
 else
  fprintf(stderr,"%s: %s\n",progname,message);
 fprintf(stderr,
  "usage: %s [options] [filenames]\n"
  "Available options are:\n"
  "  -l       list (use -l -l for full listing)\n"
  "  -o name  output to file 'name' (default is \"%s\")\n"
  "  -p       parse only\n"
  "  -s       strip debug information\n"
  "  -v       show version information\n"
  "  --       stop handling options\n"
  "  -        stop handling options and process stdin\n"
  ,progname,Output);
 exit(EXIT_FAILURE);
}

// 判断参数是否为指定字符串
#define IS(s)    (strcmp(argv[i],s)==0)
# 处理命令行参数，返回处理后的参数个数
static int doargs(int argc, char* argv[])
{
 int i;
 int version=0;
 # 如果程序名不为空，则使用参数中的程序名
 if (argv[0]!=NULL && *argv[0]!=0) progname=argv[0];
 # 遍历参数
 for (i=1; i<argc; i++)
 {
  # 如果参数不是以 '-' 开头，则表示选项结束，保留该参数
   break;
  else if (IS("--"))            /* end of options; skip it */
  {
   ++i;
   if (version) ++version;
   break;
  }
  else if (IS("-"))            /* end of options; use stdin */
   break;
  else if (IS("-l"))            /* list */
   ++listing;
  else if (IS("-o"))            /* output file */
  {
   output=argv[++i];
   if (output==NULL || *output==0 || (*output=='-' && output[1]!=0))
    usage("'-o' needs argument");
   if (IS("-")) output=NULL;
  }
  else if (IS("-p"))            /* parse only */
   dumping=0;
  else if (IS("-s"))            /* strip debug information */
   stripping=1;
  else if (IS("-v"))            /* show version */
   ++version;
  else                    /* unknown option */
   usage(argv[i]);
 }
 # 如果参数遍历结束且有列表或者不需要转储，则设置转储为0，参数为输出文件名
 if (i==argc && (listing || !dumping))
 {
  dumping=0;
  argv[--i]=Output;
 }
 # 如果有版本信息，则打印版本信息
 if (version)
 {
  printf("%s\n",LUA_COPYRIGHT);
  if (version==argc-1) exit(EXIT_SUCCESS);
 }
 return i;
}

# 定义一个 Lua 代码字符串
#define FUNCTION "(function()end)();"

# 读取 Lua 代码的回调函数
static const char* reader(lua_State* L, void* ud, size_t* size)
{
 UNUSED(L);
 # 如果传入的整数指针减到0以下，则返回 Lua 代码字符串
 if ((*(int*)ud)--)
 {
  *size=sizeof(FUNCTION)-1;
  return FUNCTION;
 }
 else
 {
  *size=0;
  return NULL;
 }
}

# 获取函数原型
#define toproto(L,i) getproto(s2v(L->top+(i)))

# 合并多个 Lua 函数原型
static const Proto* combine(lua_State* L, int n)
{
 if (n==1)
  return toproto(L,-1);
 else
 {
  Proto* f;
  int i=n;
  # 使用 lua_load 函数加载 Lua 代码，获取函数原型
  if (lua_load(L,reader,&i,"=(" PROGNAME ")",NULL)!=LUA_OK) fatal(lua_tostring(L,-1));
  f=toproto(L,-1);
  for (i=0; i<n; i++)
  {
   f->p[i]=toproto(L,i-n-1);
   if (f->p[i]->sizeupvalues>0) f->p[i]->upvalues[0].instack=0;
  }
  luaM_freearray(L,f->lineinfo,f->sizelineinfo);
  f->sizelineinfo=0;
  return f;
 }
}

# 写入 Lua 函数原型的回调函数
static int writer(lua_State* L, const void* p, size_t size, void* u)
{
 UNUSED(L);
 return (fwrite(p,size,1,(FILE*)u)!=1) && (size!=0);
}
static int pmain(lua_State* L)
{
 int argc=(int)lua_tointeger(L,1);  // 从 Lua 栈中获取参数个数
 char** argv=(char**)lua_touserdata(L,2);  // 从 Lua 栈中获取参数数组
 const Proto* f;  // 定义 Proto 结构体指针
 int i;  // 定义循环变量 i
 tmname=G(L)->tmname;  // 获取全局状态机的 tmname 字段
 if (!lua_checkstack(L,argc)) fatal("too many input files");  // 检查 Lua 栈是否有足够的空间
 for (i=0; i<argc; i++)  // 遍历参数数组
 {
  const char* filename=IS("-") ? NULL : argv[i];  // 判断参数是否为“-”，如果是则设置文件名为 NULL，否则设置为参数数组中的值
  if (luaL_loadfile(L,filename)!=LUA_OK) fatal(lua_tostring(L,-1));  // 加载文件到 Lua 环境中
 }
 f=combine(L,argc);  // 调用 combine 函数，将结果赋值给 f
 if (listing) luaU_print(f,listing>1);  // 如果 listing 为真，则打印 f 的内容
 if (dumping)  // 如果 dumping 为真
 {
  FILE* D= (output==NULL) ? stdout : fopen(output,"wb");  // 打开输出文件流
  if (D==NULL) cannot("open");  // 如果文件流为空，则输出错误信息
  lua_lock(L);  // 锁定 Lua 环境
  luaU_dump(L,f,writer,D,stripping);  // 转储 Lua 函数到文件流中
  lua_unlock(L);  // 解锁 Lua 环境
  if (ferror(D)) cannot("write");  // 如果文件流出错，则输出错误信息
  if (fclose(D)) cannot("close");  // 关闭文件流
 }
 return 0;  // 返回 0
}

int main(int argc, char* argv[])
{
 lua_State* L;  // 定义 Lua 状态机指针
 int i=doargs(argc,argv);  // 调用 doargs 函数，将结果赋值给 i
 argc-=i; argv+=i;  // 调整参数数组和参数个数
 if (argc<=0) usage("no input files given");  // 如果参数个数小于等于 0，则输出错误信息
 L=luaL_newstate();  // 创建一个新的 Lua 状态机
 if (L==NULL) fatal("cannot create state: not enough memory");  // 如果状态机为空，则输出错误信息
 lua_pushcfunction(L,&pmain);  // 将 pmain 函数推入 Lua 栈
 lua_pushinteger(L,argc);  // 将参数个数推入 Lua 栈
 lua_pushlightuserdata(L,argv);  // 将参数数组推入 Lua 栈
 if (lua_pcall(L,2,0,0)!=LUA_OK) fatal(lua_tostring(L,-1));  // 调用 Lua 函数，处理错误信息
 lua_close(L);  // 关闭 Lua 状态机
 return EXIT_SUCCESS;  // 返回成功状态
}

/*
** print bytecodes
*/

#define UPVALNAME(x) ((f->upvalues[x].name) ? getstr(f->upvalues[x].name) : "-")  // 定义宏，获取 upvalues 的名称
#define VOID(p) ((const void*)(p))  // 定义宏，将指针转换为 const void*
#define eventname(i) (getstr(tmname[i]))  // 定义宏，获取事件名称

static void PrintString(const TString* ts)
{
 const char* s=getstr(ts);  // 获取字符串的值
 size_t i,n=tsslen(ts);  // 定义变量 i 和 n，获取字符串长度
 printf("\"");  // 打印引号
 for (i=0; i<n; i++)  // 遍历字符串
 {
  int c=(int)(unsigned char)s[i];  // 获取字符的 ASCII 值
  switch (c)  // 开始 switch 分支
  {
   case '"':  // 如果是双引号
    printf("\\\"");  // 打印转义字符
    break;
   case '\\':  // 如果是反斜杠
    printf("\\\\");  // 打印转义字符
    break;
   case '\a':  // 如果是响铃
    printf("\\a");  // 打印转义字符
    break;
   case '\b':  // 如果是退格
    printf("\\b");  // 打印转义字符
    break;
   case '\f':  // 如果是换页
    printf("\\f");  // 打印转义字符
    break;
   case '\n':  // 如果是换行
    printf("\\n");  // 打印转义字符
    break;
   case '\r':  // 如果是回车
    printf("\\r");  // 打印转义字符
    break;
   case '\t':  // 如果是制表符
    printf("\\t");  // 打印转义字符
    break;
   case '\v':  // 如果是垂直制表符
    printf("\\v");  // 打印转义字符
    break;
   default:  // 默认情况
    if (isprint(c)) printf("%c",c); else printf("\\%03d",c);  // 如果是可打印字符，则直接打印，否则打印转义字符
    break;
  }
 }
 printf("\"");  // 打印引号
}
static void PrintType(const Proto* f, int i)
{
 const TValue* o=&f->k[i];  # 获取函数原型中常量数组的第 i 个元素的地址
 switch (ttypetag(o))  # 根据常量的类型标签进行判断
 {
  case LUA_VNIL:  # 如果是 nil 类型
    printf("N");  # 打印 N
    break;
  case LUA_VFALSE:  # 如果是 false 类型
  case LUA_VTRUE:   # 或者是 true 类型
    printf("B");    # 打印 B
    break;
  case LUA_VNUMFLT:  # 如果是浮点数类型
    printf("F");      # 打印 F
    break;
  case LUA_VNUMINT:  # 如果是整数类型
    printf("I");      # 打印 I
    break;
  case LUA_VSHRSTR:   # 如果是短字符串类型
  case LUA_VLNGSTR:   # 或者是长字符串类型
    printf("S");      # 打印 S
    break;
  default:                /* cannot happen */  # 默认情况下，不可能发生
    printf("?%d",ttypetag(o));  # 打印 ? 和类型标签的值
    break;
 }
 printf("\t");  # 打印制表符
}

static void PrintConstant(const Proto* f, int i)
{
 const TValue* o=&f->k[i];  # 获取函数原型中常量数组的第 i 个元素的地址
 switch (ttypetag(o))  # 根据常量的类型标签进行判断
 {
  case LUA_VNIL:  # 如果是 nil 类型
    printf("nil");  # 打印 nil
    break;
  case LUA_VFALSE:  # 如果是 false 类型
    printf("false");  # 打印 false
    break;
  case LUA_VTRUE:   # 如果是 true 类型
    printf("true");   # 打印 true
    break;
  case LUA_VNUMFLT:  # 如果是浮点数类型
    {
    char buff[100];  # 定义一个长度为 100 的字符数组
    sprintf(buff,LUA_NUMBER_FMT,fltvalue(o));  # 将浮点数值格式化为字符串存储到 buff 中
    printf("%s",buff);  # 打印 buff 中的内容
    if (buff[strspn(buff,"-0123456789")]=='\0') printf(".0");  # 如果 buff 中的内容只包含数字和负号，则打印 .0
    break;
    }
  case LUA_VNUMINT:  # 如果是整数类型
    printf(LUA_INTEGER_FMT,ivalue(o));  # 打印整数值
    break;
  case LUA_VSHRSTR:   # 如果是短字符串类型
  case LUA_VLNGSTR:   # 或者是长字符串类型
    PrintString(tsvalue(o));  # 调用 PrintString 函数打印字符串值
    break;
  default:                /* cannot happen */  # 默认情况下，不可能发生
    printf("?%d",ttypetag(o));  # 打印 ? 和类型标签的值
    break;
 }
}

#define COMMENT        "\t; "  # 定义注释的格式
#define EXTRAARG    GETARG_Ax(code[pc+1])  # 获取指令中的扩展参数
#define EXTRAARGC    (EXTRAARG*(MAXARG_C+1))  # 计算扩展参数的值
#define ISK        (isk ? "k" : "")  # 如果是常量索引，返回 "k"，否则返回空字符串

static void PrintCode(const Proto* f)
{
 const Instruction* code=f->code;  # 获取函数原型中的指令数组
 int pc,n=f->sizecode;  # 定义变量 pc 和 n，分别表示指令索引和指令数组的大小
 for (pc=0; pc<n; pc++)  # 遍历指令数组
 {
  Instruction i=code[pc];  # 获取当前指令
  OpCode o=GET_OPCODE(i);  # 获取当前指令的操作码
  int a=GETARG_A(i);  # 获取当前指令的 A 参数
  int b=GETARG_B(i);  # 获取当前指令的 B 参数
  int c=GETARG_C(i);  # 获取当前指令的 C 参数
  int ax=GETARG_Ax(i);  # 获取当前指令的 Ax 参数
  int bx=GETARG_Bx(i);  # 获取当前指令的 Bx 参数
  int sb=GETARG_sB(i);  # 获取当前指令的 signed B 参数
  int sc=GETARG_sC(i);  # 获取当前指令的 signed C 参数
  int sbx=GETARG_sBx(i);  # 获取当前指令的 signed Bx 参数
  int isk=GETARG_k(i);  # 获取当前指令的常量索引标志
  int line=luaG_getfuncline(f,pc);  # 获取当前指令所在的行号
  printf("\t%d\t",pc+1);  # 打印指令索引
  if (line>0) printf("[%d]\t",line); else printf("[-]\t");  # 如果有行号，打印行号，否则打印 -
  printf("%-9s\t",opnames[o]);  # 打印操作码对应的名称
  switch (o)  # 根据操作码进行判断
  {
   case OP_MOVE:  # 如果是 MOVE 操作
    printf("%d %d",a,b);  # 打印 A 和 B 参数
    break;
   case OP_LOADI:  # 如果是 LOADI 操作
    printf("%d %d",a,sbx);  # 打印 A 和 signed Bx 参数
    break;
   case OP_LOADF:  # 如果是 LOADF 操作
    # 输出变量 a 和 sbx 的值
    printf("%d %d",a,sbx);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_LOADK
   case OP_LOADK:
    # 输出变量 a 和 bx 的值
    printf("%d %d",a,bx);
    # 输出注释
    printf(COMMENT); PrintConstant(f,bx);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_LOADKX
   case OP_LOADKX:
    # 输出变量 a 的值
    printf("%d",a);
    # 输出注释
    printf(COMMENT); PrintConstant(f,EXTRAARG);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_LOADFALSE
   case OP_LOADFALSE:
    # 输出变量 a 的值
    printf("%d",a);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_LFALSESKIP
   case OP_LFALSESKIP:
    # 输出变量 a 的值
    printf("%d",a);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_LOADTRUE
   case OP_LOADTRUE:
    # 输出变量 a 的值
    printf("%d",a);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_LOADNIL
   case OP_LOADNIL:
    # 输出变量 a 和 b 的值
    printf("%d %d",a,b);
    # 输出注释
    printf(COMMENT "%d out",b+1);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_GETUPVAL
   case OP_GETUPVAL:
    # 输出变量 a 和 b 的值
    printf("%d %d",a,b);
    # 输出注释
    printf(COMMENT "%s",UPVALNAME(b));
    # 跳出当前循环
    break;
   # 如果操作码是 OP_SETUPVAL
   case OP_SETUPVAL:
    # 输出变量 a 和 b 的值
    printf("%d %d",a,b);
    # 输出注释
    printf(COMMENT "%s",UPVALNAME(b));
    # 跳出当前循环
    break;
   # 如果操作码是 OP_GETTABUP
   case OP_GETTABUP:
    # 输出变量 a, b, 和 c 的值
    printf("%d %d %d",a,b,c);
    # 输出注释
    printf(COMMENT "%s",UPVALNAME(b));
    # 输出常量
    printf(" "); PrintConstant(f,c);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_GETTABLE
   case OP_GETTABLE:
    # 输出变量 a, b, 和 c 的值
    printf("%d %d %d",a,b,c);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_GETI
   case OP_GETI:
    # 输出变量 a, b, 和 c 的值
    printf("%d %d %d",a,b,c);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_GETFIELD
   case OP_GETFIELD:
    # 输出变量 a, b, 和 c 的值
    printf("%d %d %d",a,b,c);
    # 输出注释
    printf(COMMENT); PrintConstant(f,c);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_SETTABUP
   case OP_SETTABUP:
    # 输出变量 a, b, c, 和 ISK 的值
    printf("%d %d %d%s",a,b,c,ISK);
    # 输出注释
    printf(COMMENT "%s",UPVALNAME(a));
    # 输出常量
    printf(" "); PrintConstant(f,b);
    # 如果 ISK 为真，输出常量
    if (isk) { printf(" "); PrintConstant(f,c); }
    # 跳出当前循环
    break;
   # 如果操作码是 OP_SETTABLE
   case OP_SETTABLE:
    # 输出变量 a, b, c, 和 ISK 的值
    printf("%d %d %d%s",a,b,c,ISK);
    # 如果 ISK 为真，输出注释
    if (isk) { printf(COMMENT); PrintConstant(f,c); }
    # 跳出当前循环
    break;
   # 如果操作码是 OP_SETI
   case OP_SETI:
    # 输出变量 a, b, c, 和 ISK 的值
    printf("%d %d %d%s",a,b,c,ISK);
    # 如果 ISK 为真，输出注释
    if (isk) { printf(COMMENT); PrintConstant(f,c); }
    # 跳出当前循环
    break;
   # 如果操作码是 OP_SETFIELD
   case OP_SETFIELD:
    # 输出变量 a, b, c, 和 ISK 的值
    printf("%d %d %d%s",a,b,c,ISK);
    # 输出注释
    printf(COMMENT); PrintConstant(f,b);
    # 如果 ISK 为真，输出常量
    if (isk) { printf(" "); PrintConstant(f,c); }
    # 跳出当前循环
    break;
   # 如果操作码是 OP_NEWTABLE
   case OP_NEWTABLE:
    # 输出变量 a, b, 和 c 的值
    printf("%d %d %d",a,b,c);
    # 输出注释
    printf(COMMENT "%d",c+EXTRAARGC);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_SELF
   case OP_SELF:
    # 输出变量 a, b, c, 和 ISK 的值
    printf("%d %d %d%s",a,b,c,ISK);
    # 如果 ISK 为真，输出注释
    if (isk) { printf(COMMENT); PrintConstant(f,c); }
    # 跳出当前循环
    break;
   # 如果操作码是 OP_ADDI
   case OP_ADDI:
    # 输出变量 a, b, 和 sc 的值
    printf("%d %d %d",a,b,sc);
    # 跳出当前循环
    break;
   # 如果操作码是 OP_ADDK
   case OP_ADDK:
    # 输出变量 a, b, 和 c 的值
    printf("%d %d %d",a,b,c);
    # 输出注释
    printf(COMMENT); PrintConstant(f,c);
    # 结束当前的 switch 语句块
    break;
   # 如果操作码是 OP_SUBK
   case OP_SUBK:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 打印注释
    printf(COMMENT); PrintConstant(f,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_MULK
   case OP_MULK:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 打印注释
    printf(COMMENT); PrintConstant(f,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_MODK
   case OP_MODK:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 打印注释
    printf(COMMENT); PrintConstant(f,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_POWK
   case OP_POWK:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 打印注释
    printf(COMMENT); PrintConstant(f,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_DIVK
   case OP_DIVK:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 打印注释
    printf(COMMENT); PrintConstant(f,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_IDIVK
   case OP_IDIVK:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 打印注释
    printf(COMMENT); PrintConstant(f,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_BANDK
   case OP_BANDK:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 打印注释
    printf(COMMENT); PrintConstant(f,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_BORK
   case OP_BORK:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 打印注释
    printf(COMMENT); PrintConstant(f,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_BXORK
   case OP_BXORK:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 打印注释
    printf(COMMENT); PrintConstant(f,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_SHRI
   case OP_SHRI:
    # 打印 a、b、sc 的值
    printf("%d %d %d",a,b,sc);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_SHLI
   case OP_SHLI:
    # 打印 a、b、sc 的值
    printf("%d %d %d",a,b,sc);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_ADD
   case OP_ADD:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_SUB
   case OP_SUB:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_MUL
   case OP_MUL:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_MOD
   case OP_MOD:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_POW
   case OP_POW:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_DIV
   case OP_DIV:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_IDIV
   case OP_IDIV:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_BAND
   case OP_BAND:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_BOR
   case OP_BOR:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_BXOR
   case OP_BXOR:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_SHL
   case OP_SHL:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_SHR
   case OP_SHR:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_MMBIN
   case OP_MMBIN:
    # 打印 a、b、c 的值
    printf("%d %d %d",a,b,c);
    # 打印注释
    printf(COMMENT "%s",eventname(c));
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_MMBINI
   case OP_MMBINI:
    # 打印 a、sb、c、isk 的值
    printf("%d %d %d %d",a,sb,c,isk);
    # 打印注释
    printf(COMMENT "%s",eventname(c));
    # 如果 isk 为真，打印 " flip"
    if (isk) printf(" flip");
    # 结束当前的 case 块
    break;
   # 如果操作码是 OP_MMBINK
    # 打印变量 a, b, c, isk 的值
    printf("%d %d %d %d",a,b,c,isk);
    # 打印注释和事件名
    printf(COMMENT "%s ",eventname(c)); PrintConstant(f,b);
    # 如果 isk 为真，打印 " flip"
    if (isk) printf(" flip");
    # 结束当前 case
    break;
   # 下面的 case 类似，都是打印不同的变量值或常量
   case OP_UNM:
    printf("%d %d",a,b);
    break;
   case OP_BNOT:
    printf("%d %d",a,b);
    break;
   case OP_NOT:
    printf("%d %d",a,b);
    break;
   case OP_LEN:
    printf("%d %d",a,b);
    break;
   case OP_CONCAT:
    printf("%d %d",a,b);
    break;
   case OP_CLOSE:
    printf("%d",a);
    break;
   case OP_TBC:
    printf("%d",a);
    break;
   case OP_JMP:
    # 打印变量和注释
    printf("%d",GETARG_sJ(i));
    printf(COMMENT "to %d",GETARG_sJ(i)+pc+2);
    break;
   # 其他 case 类似，都是打印不同的变量值或常量
    # 打印带有注释的字符串，指示跳转到指定位置
    printf(COMMENT "exit to %d",pc+bx+3);
    # 跳出当前循环
    break;
   case OP_TFORPREP:
    # 打印两个整数
    printf("%d %d",a,bx);
    # 打印带有注释的字符串，指示跳转到指定位置
    printf(COMMENT "to %d",pc+bx+2);
    # 跳出当前循环
    break;
   case OP_TFORCALL:
    # 打印两个整数
    printf("%d %d",a,c);
    # 跳出当前循环
    break;
   case OP_TFORLOOP:
    # 打印两个整数
    printf("%d %d",a,bx);
    # 打印带有注释的字符串，指示跳转到指定位置
    printf(COMMENT "to %d",pc-bx+2);
    # 跳出当前循环
    break;
   case OP_SETLIST:
    # 打印三个整数
    printf("%d %d %d",a,b,c);
    # 如果 c 是常量索引，打印带有注释的字符串，指示额外参数的数量
    if (isk) printf(COMMENT "%d",c+EXTRAARGC);
    # 跳出当前循环
    break;
   case OP_CLOSURE:
    # 打印两个整数
    printf("%d %d",a,bx);
    # 打印带有注释的字符串，指示闭包函数的地址
    printf(COMMENT "%p",VOID(f->p[bx]));
    # 跳出当前循环
    break;
   case OP_VARARG:
    # 打印两个整数
    printf("%d %d",a,c);
    # 打印带有注释的字符串，指示变长参数的数量
    printf(COMMENT);
    if (c==0) printf("all out"); else printf("%d out",c-1);
    # 跳出当前循环
    break;
   case OP_VARARGPREP:
    # 打印一个整数
    printf("%d",a);
    # 跳出当前循环
    break;
   case OP_EXTRAARG:
    # 打印一个整数
    printf("%d",ax);
    # 跳出当前循环
    break;
#if 0
   default:
    printf("%d %d %d",a,b,c);
    printf(COMMENT "not handled");
    break;
#endif
  }
  printf("\n");
 }
}


#define SS(x)    ((x==1)?"":"s")  # 定义宏，根据参数 x 的值返回空字符串或者 "s"
#define S(x)    (int)(x),SS(x)  # 定义宏，将参数 x 转换为整数并添加后缀 "s"

static void PrintHeader(const Proto* f)  # 定义函数，打印函数头部信息
{
 const char* s=f->source ? getstr(f->source) : "=?";  # 如果函数的源文件存在，则获取源文件名，否则返回 "=?"
 if (*s=='@' || *s=='=')
  s++;
 else if (*s==LUA_SIGNATURE[0])
  s="(bstring)";
 else
  s="(string)";
 printf("\n%s <%s:%d,%d> (%d instruction%s at %p)\n",
    (f->linedefined==0)?"main":"function",s,
    f->linedefined,f->lastlinedefined,
    S(f->sizecode),VOID(f));  # 打印函数信息，包括函数类型、源文件名、起始行号、结束行号、指令数量和指令地址
 printf("%d%s param%s, %d slot%s, %d upvalue%s, ",
    (int)(f->numparams),f->is_vararg?"+":"",SS(f->numparams),
    S(f->maxstacksize),S(f->sizeupvalues));  # 打印参数数量、是否可变参数、局部变量数量、upvalue数量
 printf("%d local%s, %d constant%s, %d function%s\n",
    S(f->sizelocvars),S(f->sizek),S(f->sizep));  # 打印局部变量数量、常量数量、子函数数量
}

static void PrintDebug(const Proto* f)  # 定义函数，打印调试信息
{
 int i,n;
 n=f->sizek;
 printf("constants (%d) for %p:\n",n,VOID(f));  # 打印常量数量和指针地址
 for (i=0; i<n; i++)
 {
  printf("\t%d\t",i);
  PrintType(f,i);  # 打印常量类型
  PrintConstant(f,i);  # 打印常量值
  printf("\n");
 }
 n=f->sizelocvars;
 printf("locals (%d) for %p:\n",n,VOID(f));  # 打印局部变量数量和指针地址
 for (i=0; i<n; i++)
 {
  printf("\t%d\t%s\t%d\t%d\n",
  i,getstr(f->locvars[i].varname),f->locvars[i].startpc+1,f->locvars[i].endpc+1);  # 打印局部变量索引、名称、起始行号、结束行号
 }
 n=f->sizeupvalues;
 printf("upvalues (%d) for %p:\n",n,VOID(f));  # 打印upvalue数量和指针地址
 for (i=0; i<n; i++)
 {
  printf("\t%d\t%s\t%d\t%d\n",
  i,UPVALNAME(i),f->upvalues[i].instack,f->upvalues[i].idx);  # 打印upvalue索引、名称、是否在栈内、索引位置
 }
}

static void PrintFunction(const Proto* f, int full)  # 定义函数，打印函数信息
{
 int i,n=f->sizep;
 PrintHeader(f);  # 调用打印函数头部信息的函数
 PrintCode(f);  # 调用打印函数指令的函数
 if (full) PrintDebug(f);  # 如果需要打印完整信息，则调用打印调试信息的函数
 for (i=0; i<n; i++) PrintFunction(f->p[i],full);  # 递归打印子函数信息
}
```