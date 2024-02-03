# `nmap\libssh2\vms\man2help.c`

```cpp
#include <stdio.h>  // 包含标准输入输出库
#include <stdlib.h>  // 包含标准库函数
#include <string.h>  // 包含字符串处理函数
#include <ctype.h>  // 包含字符处理函数
#include <errno.h>  // 包含错误码定义

#include <starlet.h>  // 包含 VMS 系统调用相关头文件
#include <lib$routines.h>  // 包含 VMS 系统调用相关头文件
#include <ssdef.h>  // 包含 VMS 系统调用相关头文件
#include <descrip.h>  // 包含 VMS 系统调用相关头文件
#include <rms.h>  // 包含 VMS 系统调用相关头文件

typedef struct manl{  // 定义结构体 manl
    struct manl *next;  // 指向下一个 manl 结构体的指针
    char *filename;  // 文件名
}man, *manPtr;  // 定义 man 结构体和 manPtr 指针类型

typedef struct pf_fabnam{  // 定义结构体 pf_fabnam
    struct FAB dfab;  // 文件访问块
    struct RAB drab;  // 记录访问块
    struct namldef dnam;  // 文件名定义
    char   expanded_filename[NAM$C_MAXRSS + 1];  // 扩展后的文件名
} pfn, *pfnPtr;  // 定义 pfn 结构体和 pfnPtr 指针类型

/*----------------------------------------------------------*/

fpcopy( char *output, char *input, int len )  // 定义函数 fpcopy，用于复制字符串
{
    char    *is, *os;  // 定义输入字符串和输出字符串的指针
    int i;  // 定义循环变量 i

    if ( len ){  // 如果长度不为 0
        for ( is = input, os = output, i = 0; i < len ; ++i, ++is, ++os){  // 循环复制字符串
                *os = *is;  // 复制字符
        }
        *os = 0;  // 在输出字符串末尾添加结束符
    }else{  // 如果长度为 0
        output[0] = 0;  // 输出字符串为空
    }
}           

/*----------------------------------------------------------*/
/* give part of ilename in partname. See code for proper
   value of i ( 0 = node, 1 = dev, 2 = dir,3 = name etc.
*/ 

int fnamepart( char *inputfile, char *part, int whatpart )  // 定义函数 fnamepart，用于获取文件名的部分
{
    pfnPtr pf;  // 定义 pfnPtr 类型的指针 pf
    int     status;  // 定义状态变量 status
    char    ipart[6][256], *i, *p;  // 定义字符串数组 ipart 和指针变量 i, p

    pf = calloc( 1, sizeof( pfn ) );  // 分配内存给 pf

    pf->dfab = cc$rms_fab;  // 初始化 dfab
    pf->drab = cc$rms_rab;  // 初始化 drab
    pf->dnam = cc$rms_naml;  // 初始化 dnam

    pf->dfab.fab$l_naml = &pf->dnam;  // 设置 fab 的文件名定义
    pf->dfab.fab$l_fna = (char *) -1;  // 设置 fab 的文件名
    pf->dfab.fab$l_dna = (char *) -1;  // 设置 fab 的目录名
    pf->dfab.fab$b_fns = 0;  // 设置 fab 的文件名长度
    pf->dfab.fab$w_ifi = 0;  // 设置 fab 的文件索引

    pf->dnam.naml$l_long_defname = NULL;  // 设置文件名定义的长格式默认文件名
    pf->dnam.naml$l_long_defname_size = 0;  // 设置文件名定义的长格式默认文件名长度

    pf->dnam.naml$l_long_filename = inputfile;  // 设置文件名定义的长格式文件名
    pf->dnam.naml$l_long_filename_size = strlen( inputfile );  // 设置文件名定义的长格式文件名长度

    pf->dnam.naml$l_long_expand = pf->expanded_filename;  // 设置文件名定义的长格式扩展文件名
    pf->dnam.naml$l_long_expand_alloc = NAM$C_MAXRSS ;  // 设置文件名定义的长格式扩展文件名长度

    pf->dnam.naml$b_nop |= NAML$M_SYNCHK | NAML$M_PWD;  // 设置文件名定义的标志位

    status = sys$parse( &pf->dfab, 0,0);  // 解析文件名
    if ( !(status&1) ){  // 如果解析失败
        free( pf );  // 释放内存
        return( status );  // 返回状态
    }

    fpcopy ( ipart[0], pf->dnam.naml$l_long_node , pf->dnam.naml$l_long_node_size);  // 复制节点名
    fpcopy ( ipart[1], pf->dnam.naml$l_long_dev , pf->dnam.naml$l_long_dev_size);  // 复制设备名
fpcopy ( ipart[2], pf->dnam.naml$l_long_dir , pf->dnam.naml$l_long_dir_size);
# 将 pf->dnam.naml$l_long_dir 中的数据复制到 ipart[2] 中

fpcopy ( ipart[3], pf->dnam.naml$l_long_name , pf->dnam.naml$l_long_name_size);
# 将 pf->dnam.naml$l_long_name 中的数据复制到 ipart[3] 中

fpcopy ( ipart[4], pf->dnam.naml$l_long_type , pf->dnam.naml$l_long_type_size);                                               
# 将 pf->dnam.naml$l_long_type 中的数据复制到 ipart[4] 中

fpcopy ( ipart[5], pf->dnam.naml$l_long_ver , pf->dnam.naml$l_long_ver_size);
# 将 pf->dnam.naml$l_long_ver 中的数据复制到 ipart[5] 中

for( i = ipart[ whatpart ], p = part; *i; ++i, ++p){
# 遍历 ipart[whatpart] 中的数据，将大写字母转换为小写字母
   if ( p == part ){
      *p = toupper( *i );
   }else{
      *p = tolower( *i );
   }        
}
*p = 0;
# 将 p 指向的位置设置为 0

free( pf );
# 释放 pf 指向的内存空间
return(1);
# 返回值为 1

int find_file(char *filename,char *gevonden,int *findex)
# 定义函数 find_file，接受文件名、gevonden 和 findex 作为参数

int     status;
struct  dsc$descriptor gevondend;
struct  dsc$descriptor filespec;
char    gevonden_file[NAM$C_MAXRSS + 1];
# 定义变量 status, gevondend, filespec 和 gevonden_file

filespec.dsc$w_length = strlen(filename);
filespec.dsc$b_dtype  = DSC$K_DTYPE_T;
filespec.dsc$b_class  = DSC$K_CLASS_S; 
filespec.dsc$a_pointer = filename;
# 设置 filespec 的属性和指针

gevondend.dsc$w_length = NAM$C_MAXRSS;
gevondend.dsc$b_dtype  = DSC$K_DTYPE_T;
gevondend.dsc$b_class  = DSC$K_CLASS_S; 
gevondend.dsc$a_pointer = gevonden_file;
# 设置 gevondend 的属性和指针

status=lib$find_file(&filespec,&gevondend,findex,0,0,0,0);
# 调用 lib$find_file 函数，查找文件，并将结果存储在 status 中

if ( (status & 1) == 1 ){
       strcpy(gevonden,strtok(gevonden_file," "));
}else{
       gevonden[0] = 0;
}
# 如果查找成功，将 gevonden_file 中的内容复制到 gevonden 中，否则将 gevonden 的第一个字符设置为 0

return(status);
# 返回 status 的值

manPtr addman( manPtr *manroot,char *filename )
# 定义函数 addman，接受 manroot 和 filename 作为参数

manPtr m,f;
# 定义变量 m 和 f

m = calloc( 1, sizeof( man) );
# 分配内存空间给 m

if ( !m ) return( NULL );
# 如果分配失败，返回 NULL

m->filename = strdup( filename );
# 将 filename 的副本存储在 m->filename 中

if ( *manroot == NULL ){
   *manroot = m;    
}else{
   for( f = *manroot; f->next ; f = f->next );
   f->next = m;
}
# 如果 manroot 为空，将 m 存储在 manroot 中，否则将 m 追加到 manroot 的末尾

return(m);
# 返回 m 的值

void freeman( manPtr *manroot )
# 定义函数 freeman，接受 manroot 作为参数

manPtr m,n;
# 定义变量 m 和 n

for( m = *manroot; m ; m = n ){
# 遍历 manroot 中的元素
     free( m->filename );
     n = m->next;
     free ( m );
}
# 释放每个元素的内存空间

*manroot = NULL;
# 将 manroot 设置为 NULL

int listofmans( char *filespec, manPtr *manroot )
# 定义函数 listofmans，接受 filespec 和 manroot 作为参数

manPtr  r;
int     status;
# 定义变量 r 和 status
# 初始化文件索引为0
int     ffindex=0;
# 创建一个字符数组，用于存储找到的文件名
char    gevonden[NAM$C_MAXRSS + 1];

# 进入无限循环
while(1){
    # 查找文件，将结果存储在 gevonden 中，更新文件索引
    status = find_file( filespec, gevonden, &ffindex );

    # 如果找到文件
    if ( (status&1) ){
        # 将找到的文件添加到 manroot 中
        r = addman( manroot, gevonden );
        # 如果添加失败，返回2
        if ( r == NULL ) return(2);
    }else{
        # 如果未找到文件，且状态不为1，跳出循环
        if ( !( status&1)) break;
    }
}

# 结束文件查找
lib$find_file_end( &ffindex);
# 如果状态为 RMS$_NMF，将状态更新为1
if ( status == RMS$_NMF) status = 1;

# 返回状态
return( status );
}

/*--------------------------------------------*/

# 定义函数 convertman，接受文件路径、文件指针、基本级别和是否添加括号作为参数
int convertman ( char *filespec, FILE *hlp , int base_level, int add_parentheses )
{
# 定义变量
FILE    *man;
char    *in, *uit;
char    *m,*h;
size_t  len, thislen, maxlen= 50000;
int     bol,mode, return_status=1;
char subjectname[ NAM$C_MAXRSS + 1 ];

# 分配内存给输入和输出缓冲区
in  = calloc( 1, maxlen + 1 );
uit = calloc( 1, maxlen + 1 );

# 如果内存分配失败，返回2
if ( in == NULL || uit == NULL ) return(2);

# 以只读方式打开文件
man = fopen( filespec, "r");
# 如果文件打开失败，返回错误码
if ( man == NULL ) return(vaxc$errno);

# 读取文件内容到输入缓冲区
for( len = 0; !feof( man ) && len < maxlen ; len += thislen ){
    thislen = fread( in + len, 1, maxlen - len, man );
}

# 关闭文件
fclose (man);

# 初始化指针
m = in;
h = uit;

# 将输入缓冲区的末尾置为0
*(m + len ) = 0;

# 遍历输入缓冲区
for ( mode = 0, bol = 1 ; *m; ++m ){

    } /*end switch mode */

    bol = 0;
    # 如果当前字符为换行符或回车符，将 bol 置为1
    if ( *m == '\n' || *m == '\r') bol = 1;

}/* end for mode */

# 将输出缓冲区的末尾置为0
*h = 0;

# 如果返回状态为2
if ( (return_status&2) ){
    # 将 uit 写入到 hlp 中
    fprintf( hlp, "%s\n\n", uit);
}else{
    # 从文件路径中提取文件名
    fnamepart( filespec, subjectname,3);
    # 如果文件名不为空
    if ( *subjectname ){
        # 将基本级别、文件名和 uit 写入到 hlp 中
        fprintf( hlp, "%d %s\n\n%s\n\n", base_level, subjectname, uit);
    }else{
        # 如果没有文件名（例如逻辑文件），使用第一个单词作为主题名
        char *n,*s;

        for(n = in; isspace( *n );++n);
        for(s = subjectname; !(isspace( *n )); ++n,++s)*s = *n;
        *s = 0;

        # 将基本级别、主题名和 uit 写入到 hlp 中
        fprintf( hlp, "%d %s\n\n%s\n\n", base_level, subjectname, uit);
    }
}

# 释放内存
free( m ); 
free( h ); 

# 返回状态
return ( 1);
}

/*--------------------------------------------*/
# 将 man 文件转换为帮助文件
int convertmans( char *filespec, char *hlpfilename, int base_level, int append, int add_parentheses )
{
# 初始化状态为 1
int status=1;
# 初始化 manroot 为 NULL，m 为指向 man 结构的指针，hlp 为文件指针
manPtr  manroot=NULL, m;
FILE    *hlp;

# 如果需要追加内容，则以追加模式打开帮助文件，否则以写模式打开
if ( append ){
    hlp = fopen( hlpfilename,"a+");
}else{
    hlp = fopen( hlpfilename,"w");
}

# 如果打开文件失败，则返回错误码
if ( hlp == NULL ) return( vaxc$errno );

# 获取 man 文件列表
status = listofmans( filespec, &manroot );
# 如果获取失败，则返回错误码
if ( !(status&1) ) return( status );

# 遍历 man 文件列表
for ( m = manroot ; m ; m = m->next ){
    # 转换 man 文件为帮助文件
    status = convertman( m->filename, hlp , base_level, add_parentheses );
    # 如果转换失败，则在标准错误流中输出错误信息，并跳出循环
    if ( !(status&1) ){
        fprintf(stderr,"Convertman of %s went wrong\n", m->filename);
        break;
    }
}
# 释放 man 文件列表
freeman( &manroot );
# 返回状态
return( status );
}

/*--------------------------------------------*/
# 打印帮助信息
void print_help()
{
   fprintf( stderr, "Usage: [-a] [-b x] convertman <manfilespec> <helptextfile>\n" );
   fprintf( stderr, "       -a append <manfilespec> to <helptextfile>\n" );
   fprintf( stderr, "       -b <baselevel> if no headers found create one with level <baselevel>\n" );
   fprintf( stderr, "          and the filename as title.\n" );
   fprintf( stderr, "       -p add parentheses() to baselevel help items.\n" );

}
/*--------------------------------------------*/

# 主函数
main ( int argc, char **argv )
{
# 初始化变量
int     status;
int     i,j;
int     append, base_level, basechange, add_parentheses;
char    *manfile=NULL;
char    *helpfile=NULL;

# 如果参数少于 3 个，则打印帮助信息并返回状态 1
if ( argc < 3 ){
   print_help();
   return( 1 ) ;
}

# 初始化参数
append     = 0;
base_level = 1;
basechange = 0;
add_parentheses = 0;

# 遍历参数
for ( i = 1; i < argc; ++i){
    # 检查命令行参数是否以'-'开头
    if ( argv[i][0] == '-' ){
        # 遍历命令行参数中的每个字符
        for( j = 1; argv[i][j] ; ++j ){
            # 根据字符的不同进行不同的操作
            switch( argv[i][j] ){
                # 如果字符为'a'，则将append设置为1
                case 'a':
                    append = 1;
                    break;
                # 如果字符为'b'，则检查下一个参数是否存在，并将base_level设置为对应的整数值
                case 'b':   
                    if ( (i+1) < argc ){
                        base_level = atoi( argv[ i + 1 ] );
                        basechange = 1;
                    }
                    break;
                # 如果字符为'p'，则将add_parentheses设置为1
                case 'p':
                    add_parentheses = 1;
                    break;
            }
        }
        # 如果basechange为真，则将其重置为假，并将i增加1
        if ( basechange){
            basechange = 0;
            i = i + 1;
        }
    }else{
        # 如果manfile为空，则将其设置为当前参数的字符串副本
        if ( manfile == NULL ){
            manfile = strdup( argv[i]);
        # 如果helpfile为空，则将其设置为当前参数的字符串副本
        } else if ( helpfile == NULL ){
            helpfile = strdup( argv[i]);
        # 如果manfile和helpfile都不为空，则输出错误信息
        } else {
            fprintf( stderr, "Unrecognized parameter : %s\n", argv[i]);
        }
    }
# 关闭当前的代码块
}

# 输出错误信息到标准错误输出，包括 manfile, helpfile, append, base_level 的值
fprintf( stderr,"manfile: %s, helpfile: %s, append: %d, base_level : %d\n",
        manfile, helpfile, append, base_level);

# 调用 convertmans 函数，传入 manfile, helpfile, base_level, append, add_parentheses 参数，并将返回值赋给 status
status = convertmans( manfile, helpfile, base_level, append, add_parentheses );

# 释放 manfile 和 helpfile 所指向的内存
free( manfile );
free( helpfile );

# 返回 status 变量的值
return( status );
}
```