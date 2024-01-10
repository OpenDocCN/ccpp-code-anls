# `nmap\liblinear\train.c`

```
#include <stdio.h>  // 包含标准输入输出库
#include <math.h>   // 包含数学函数库
#include <stdlib.h> // 包含标准库函数
#include <string.h> // 包含字符串处理函数库
#include <ctype.h>  // 包含字符处理函数库
#include <errno.h>  // 包含错误处理库
#include "linear.h" // 包含自定义的 linear.h 文件
#define Malloc(type,n) (type *)malloc((n)*sizeof(type))  // 定义一个宏，用于分配内存
#define INF HUGE_VAL  // 定义一个无穷大的值

void print_null(const char *s) {}  // 定义一个函数，用于打印空字符串

void exit_with_help()  // 定义一个函数，用于打印帮助信息并退出程序
{
    printf(
    "Usage: train [options] training_set_file [model_file]\n"  // 打印使用方法
    "options:\n"  // 打印选项
    "-s type : set type of solver (default 1)\n"  // 设置求解器类型
    "    0 -- L2-regularized logistic regression (primal)\n"  // 求解器类型为 0
    "    1 -- L2-regularized L2-loss support vector classification (dual)\n"  // 求解器类型为 1
    "    2 -- L2-regularized L2-loss support vector classification (primal)\n"  // 求解器类型为 2
    "    3 -- L2-regularized L1-loss support vector classification (dual)\n"  // 求解器类型为 3
    "    4 -- multi-class support vector classification by Crammer and Singer\n"  // 求解器类型为 4
    "    5 -- L1-regularized L2-loss support vector classification\n"  // 求解器类型为 5
    "    6 -- L1-regularized logistic regression\n"  // 求解器类型为 6
    "    7 -- L2-regularized logistic regression (dual)\n"  // 求解器类型为 7
    "-c cost : set the parameter C (default 1)\n"  // 设置参数 C
    "-e epsilon : set tolerance of termination criterion\n"  // 设置终止条件的容差
    "    -s 0 and 2\n"  // 当求解器类型为 0 或 2 时
    "        |f'(w)|_2 <= eps*min(pos,neg)/l*|f'(w0)|_2,\n"  // 计算条件
    "        where f is the primal function and pos/neg are # of\n"  // 计算条件
    "        positive/negative data (default 0.01)\n"  // 默认值
    "    -s 1, 3, 4 and 7\n"  // 当求解器类型为 1, 3, 4 或 7 时
    "        Dual maximal violation <= eps; similar to libsvm (default 0.1)\n"  // 计算条件
    "    -s 5 and 6\n"  // 当求解器类型为 5 或 6 时
    "        |f'(w)|_1 <= eps*min(pos,neg)/l*|f'(w0)|_1,\n"  // 计算条件
    "        where f is the primal function (default 0.01)\n"  // 默认值
    "-B bias : if bias >= 0, instance x becomes [x; bias]; if < 0, no bias term added (default -1)\n"  // 设置偏置
    "-wi weight: weights adjust the parameter C of different classes (see README for details)\n"  // 设置权重
    "-v n: n-fold cross validation mode\n"  // 设置交叉验证模式
    "-q : quiet mode (no outputs)\n"  // 设置安静模式
    );
    exit(1);  // 退出程序
}

void exit_input_error(int line_num)  // 定义一个函数，用于打印输入错误信息并退出程序
{
    fprintf(stderr,"Wrong input format at line %d\n", line_num);  // 打印错误的输入格式
    exit(1);  // 退出程序
}

static char *line = NULL;  // 定义一个静态字符指针变量
// 定义静态变量 max_line_len
static int max_line_len;

// 读取一行文本
static char* readline(FILE *input)
{
    int len;
    
    // 读取一行文本到 line 中，如果为空则返回 NULL
    if(fgets(line,max_line_len,input) == NULL)
        return NULL;

    // 循环直到读取到换行符为止
    while(strrchr(line,'\n') == NULL)
    {
        // 将 max_line_len 增加一倍
        max_line_len *= 2;
        // 重新分配 line 的内存空间
        line = (char *) realloc(line,max_line_len);
        // 计算 line 的长度
        len = (int) strlen(line);
        // 继续读取文本到 line 中
        if(fgets(line+len,max_line_len-len,input) == NULL)
            break;
    }
    return line;
}

// 解析命令行参数
void parse_command_line(int argc, char **argv, char *input_file_name, char *model_file_name);
// 读取问题数据
void read_problem(const char *filename);
// 执行交叉验证
void do_cross_validation();

// 定义结构体指针变量
struct feature_node *x_space;
struct parameter param;
struct problem prob;
struct model* model_;
int flag_cross_validation;
int nr_fold;
double bias;

// 主函数
int main(int argc, char **argv)
{
    // 定义文件名变量
    char input_file_name[1024];
    char model_file_name[1024];
    const char *error_msg;

    // 解析命令行参数
    parse_command_line(argc, argv, input_file_name, model_file_name);
    // 读取问题数据
    read_problem(input_file_name);
    // 检查参数
    error_msg = check_parameter(&prob,&param);

    // 如果有错误信息则输出错误信息并退出程序
    if(error_msg)
    {
        fprintf(stderr,"Error: %s\n",error_msg);
        exit(1);
    }

    // 如果需要进行交叉验证则执行交叉验证
    if(flag_cross_validation)
    {
        do_cross_validation();
    }
    // 否则执行训练并保存模型
    else
    {
        model_=train(&prob, &param);
        if(save_model(model_file_name, model_))
        {
            fprintf(stderr,"can't save model to file %s\n",model_file_name);
            exit(1);
        }
        free_and_destroy_model(&model_);
    }
    // 释放参数内存
    destroy_param(&param);
    // 释放问题数据内存
    free(prob.y);
    free(prob.x);
    // 释放特征空间内存
    free(x_space);
    // 释放 line 内存
    free(line);

    return 0;
}

// 执行交叉验证
void do_cross_validation()
{
    int i;
    int total_correct = 0;
    int *target = Malloc(int, prob.l);

    // 执行交叉验证
    cross_validation(&prob,&param,nr_fold,target);

    // 统计正确的个数
    for(i=0;i<prob.l;i++)
        if(target[i] == prob.y[i])
            ++total_correct;
    // 输出交叉验证准确率
    printf("Cross Validation Accuracy = %g%%\n",100.0*total_correct/prob.l);

    // 释放 target 内存
    free(target);
}

// 解析命令行参数
void parse_command_line(int argc, char **argv, char *input_file_name, char *model_file_name)
{
    // 声明整型变量 i
    int i;
    // 声明函数指针 print_func，初始化为 NULL，用于默认打印到标准输出
    void (*print_func)(const char*) = NULL;

    // 设置默认数值
    param.solver_type = L2R_L2LOSS_SVC_DUAL;
    param.C = 1;
    param.eps = INF; // 详见下面的设置
    param.nr_weight = 0;
    param.weight_label = NULL;
    param.weight = NULL;
    flag_cross_validation = 0;
    bias = -1;

    // 解析选项
    for(i=1;i<argc;i++)
    {
        // 如果命令行参数不是以 '-' 开头，则跳出循环
        if(argv[i][0] != '-') break;
        // 如果下一个参数不存在，则调用 exit_with_help 函数
        if(++i>=argc)
            exit_with_help();
        // 根据选项进行不同的处理
        switch(argv[i-1][1])
        {
            case 's':
                param.solver_type = atoi(argv[i]);
                break;

            case 'c':
                param.C = atof(argv[i]);
                break;

            case 'e':
                param.eps = atof(argv[i]);
                break;

            case 'B':
                bias = atof(argv[i]);
                break;

            case 'w':
                // 增加权重
                ++param.nr_weight;
                param.weight_label = (int *) realloc(param.weight_label,sizeof(int)*param.nr_weight);
                param.weight = (double *) realloc(param.weight,sizeof(double)*param.nr_weight);
                param.weight_label[param.nr_weight-1] = atoi(&argv[i-1][2]);
                param.weight[param.nr_weight-1] = atof(argv[i]);
                break;

            case 'v':
                // 开启交叉验证
                flag_cross_validation = 1;
                nr_fold = atoi(argv[i]);
                if(nr_fold < 2)
                {
                    fprintf(stderr,"n-fold cross validation: n must >= 2\n");
                    exit_with_help();
                }
                break;

            case 'q':
                // 设置打印函数为 print_null，并将 i 减一
                print_func = &print_null;
                i--;
                break;

            default:
                // 输出未知选项错误信息，并调用 exit_with_help 函数
                fprintf(stderr,"unknown option: -%c\n", argv[i-1][1]);
                exit_with_help();
                break;
        }
    }

    // 设置打印字符串的函数
    set_print_string_function(print_func);

    // 确定文件名
    # 如果参数 i 大于等于参数个数 argc，则调用 exit_with_help 函数退出程序
    if(i>=argc)
        exit_with_help();

    # 将参数 argv[i] 的值复制给 input_file_name
    strcpy(input_file_name, argv[i]);

    # 如果 i 小于 argc-1，则将参数 argv[i+1] 的值复制给 model_file_name
    # 否则，根据 argv[i] 中最后一个 '/' 后面的字符串生成默认的 model_file_name
    if(i<argc-1)
        strcpy(model_file_name,argv[i+1]);
    else
    {
        char *p = strrchr(argv[i],'/');
        if(p==NULL)
            p = argv[i];
        else
            ++p;
        sprintf(model_file_name,"%s.model",p);
    }

    # 如果参数 param.eps 的值为无穷大 INF
    # 根据不同的 solver_type 设置不同的默认值
    if(param.eps == INF)
    {
        if(param.solver_type == L2R_LR || param.solver_type == L2R_L2LOSS_SVC)
            param.eps = 0.01;
        else if(param.solver_type == L2R_L2LOSS_SVC_DUAL || param.solver_type == L2R_L1LOSS_SVC_DUAL || param.solver_type == MCSVM_CS || param.solver_type == L2R_LR_DUAL)
            param.eps = 0.1;
        else if(param.solver_type == L1R_L2LOSS_SVC || param.solver_type == L1R_LR)
            param.eps = 0.01;
    }
// 读取一个问题（以libsvm格式）
void read_problem(const char *filename)
{
    int max_index, inst_max_index, i; // 定义变量
    long int elements, j; // 定义变量
    FILE *fp = fopen(filename,"r"); // 打开文件
    char *endptr; // 定义指针变量
    char *idx, *val, *label; // 定义指针变量

    if(fp == NULL) // 如果文件指针为空
    {
        fprintf(stderr,"can't open input file %s\n",filename); // 输出错误信息
        exit(1); // 退出程序
    }

    prob.l = 0; // 初始化变量
    elements = 0; // 初始化变量
    max_line_len = 1024; // 初始化变量
    line = Malloc(char,max_line_len); // 分配内存
    while(readline(fp)!=NULL) // 循环读取文件内容
    {
        char *p = strtok(line," \t"); // 以空格和制表符为分隔符，分割字符串

        // features
        while(1) // 进入无限循环
        {
            p = strtok(NULL," \t"); // 以空格和制表符为分隔符，继续分割字符串
            if(p == NULL || *p == '\n') // 如果分割结果为空或者为换行符
                break; // 退出循环
            elements++; // 元素数量加一
        }
        elements++; // 元素数量加一（为偏置项）
        prob.l++; // 问题数量加一
    }
    rewind(fp); // 将文件指针重新指向文件开头

    prob.bias=bias; // 设置偏置项

    prob.y = Malloc(int,prob.l); // 分配内存
    prob.x = Malloc(struct feature_node *,prob.l); // 分配内存
    x_space = Malloc(struct feature_node,elements+prob.l); // 分配内存

    max_index = 0; // 初始化变量
    j=0; // 初始化变量
    for(i=0;i<prob.l;i++) // 循环
    {
        // 初始化最大索引为0，如果格式错误则strtol返回0
        inst_max_index = 0; // strtol gives 0 if wrong format
        // 读取文件的一行内容
        readline(fp);
        // 将prob.x[i]指向x_space[j]
        prob.x[i] = &x_space[j];
        // 使用空格和制表符对行进行分割
        label = strtok(line," \t\n");
        // 如果label为空，则表示空行，抛出输入错误
        if(label == NULL) // empty line
            exit_input_error(i+1);
    
        // 将label转换为整数类型，存入prob.y[i]
        prob.y[i] = (int) strtol(label,&endptr,10);
        // 如果endptr等于label或者endptr不是空字符，则表示输入错误
        if(endptr == label || *endptr != '\0')
            exit_input_error(i+1);
    
        // 循环读取每个特征值
        while(1)
        {
            // 以冒号为分隔符，获取特征索引
            idx = strtok(NULL,":");
            // 以空格或制表符为分隔符，获取特征值
            val = strtok(NULL," \t");
    
            // 如果val为空，则跳出循环
            if(val == NULL)
                break;
    
            // 将idx转换为整数类型，存入x_space[j].index
            errno = 0;
            x_space[j].index = (int) strtol(idx,&endptr,10);
            // 如果endptr等于idx或者errno不为0或者endptr不是空字符或者索引小于等于最大索引，则表示输入错误
            if(endptr == idx || errno != 0 || *endptr != '\0' || x_space[j].index <= inst_max_index)
                exit_input_error(i+1);
            else
                inst_max_index = x_space[j].index;
    
            // 将val转换为双精度浮点数类型，存入x_space[j].value
            errno = 0;
            x_space[j].value = strtod(val,&endptr);
            // 如果endptr等于val或者errno不为0或者endptr不是空字符且不是空白字符，则表示输入错误
            if(endptr == val || errno != 0 || (*endptr != '\0' && !isspace(*endptr)))
                exit_input_error(i+1);
    
            // j自增
            ++j;
        }
    
        // 如果最大索引大于max_index，则更新max_index
        if(inst_max_index > max_index)
            max_index = inst_max_index;
    
        // 如果prob.bias大于等于0，则将prob.bias添加到x_space[j].value中
        if(prob.bias >= 0)
            x_space[j++].value = prob.bias;
    
        // 将x_space[j].index设置为-1
        x_space[j++].index = -1;
    }
    
    // 如果prob.bias大于等于0
    if(prob.bias >= 0)
    {
        // 设置prob.n为max_index+1
        prob.n=max_index+1;
        // 遍历prob.x，将索引为2的元素的index设置为prob.n
        for(i=1;i<prob.l;i++)
            (prob.x[i]-2)->index = prob.n; 
        // 设置x_space[j-2].index为prob.n
        x_space[j-2].index = prob.n;
    }
    else
        // 设置prob.n为max_index
        prob.n=max_index;
    
    // 关闭文件
    fclose(fp);
# 闭合前面的函数定义
```