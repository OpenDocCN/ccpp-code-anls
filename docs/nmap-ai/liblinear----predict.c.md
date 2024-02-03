# `nmap\liblinear\predict.c`

```cpp
#include <stdio.h>  // 包含标准输入输出头文件
#include <ctype.h>  // 包含字符处理头文件
#include <stdlib.h>  // 包含标准库头文件
#include <string.h>  // 包含字符串处理头文件
#include <errno.h>  // 包含错误处理头文件
#include "linear.h"  // 包含自定义头文件

struct feature_node *x;  // 定义特征节点指针变量 x
int max_nr_attr = 64;  // 定义最大属性数为 64

struct model* model_;  // 定义模型指针变量 model_
int flag_predict_probability=0;  // 定义预测概率标志变量，初始值为 0

void exit_input_error(int line_num)  // 定义输入错误退出函数，参数为行号
{
    fprintf(stderr,"Wrong input format at line %d\n", line_num);  // 输出错误信息到标准错误流
    exit(1);  // 退出程序
}

static char *line = NULL;  // 定义静态字符指针变量 line，初始值为 NULL
static int max_line_len;  // 定义静态整型变量 max_line_len

static char* readline(FILE *input)  // 定义读取一行数据的函数，参数为输入文件指针
{
    int len;
    
    if(fgets(line,max_line_len,input) == NULL)  // 如果从输入文件中读取一行数据失败
        return NULL;  // 返回空指针

    while(strrchr(line,'\n') == NULL)  // 当行数据中没有换行符时
    {
        max_line_len *= 2;  // 最大行长度乘以 2
        line = (char *) realloc(line,max_line_len);  // 重新分配内存空间
        len = (int) strlen(line);  // 获取行数据长度
        if(fgets(line+len,max_line_len-len,input) == NULL)  // 如果从输入文件中读取数据失败
            break;  // 跳出循环
    }
    return line;  // 返回行数据
}

void do_predict(FILE *input, FILE *output, struct model* model_)  // 定义进行预测的函数，参数为输入文件指针、输出文件指针和模型指针
{
    int correct = 0;  // 定义正确预测数，初始值为 0
    int total = 0;  // 定义总预测数，初始值为 0

    int nr_class=get_nr_class(model_);  // 获取模型中的类别数
    double *prob_estimates=NULL;  // 定义概率估计数组，初始值为空指针
    int j, n;  // 定义整型变量 j 和 n
    int nr_feature=get_nr_feature(model_);  // 获取模型中的特征数
    if(model_->bias>=0)  // 如果模型的偏置大于等于 0
        n=nr_feature+1;  // n 等于特征数加 1
    else
        n=nr_feature;  // 否则 n 等于特征数

    if(flag_predict_probability)  // 如果预测概率标志为真
    {
        int *labels;  // 定义整型指针变量 labels

        if(!check_probability_model(model_))  // 如果模型不支持概率输出
        {
            fprintf(stderr, "probability output is only supported for logistic regression\n");  // 输出错误信息到标准错误流
            exit(1);  // 退出程序
        }

        labels=(int *) malloc(nr_class*sizeof(int));  // 分配内存空间给 labels
        get_labels(model_,labels);  // 获取模型中的标签
        prob_estimates = (double *) malloc(nr_class*sizeof(double));  // 分配内存空间给概率估计数组
        fprintf(output,"labels");  // 输出标签到输出文件
        for(j=0;j<nr_class;j++)  // 遍历类别数
            fprintf(output," %d",labels[j]);  // 输出标签到输出文件
        fprintf(output,"\n");  // 输出换行符到输出文件
        free(labels);  // 释放 labels 的内存空间
    }

    max_line_len = 1024;  // 最大行长度设为 1024
    line = (char *)malloc(max_line_len*sizeof(char));  // 分配内存空间给行数据
    while(readline(input) != NULL)  // 当从输入文件中读取行数据不为空时
    }
    printf("Accuracy = %g%% (%d/%d)\n",(double) correct/total*100,correct,total);  // 输出准确率到标准输出流
    if(flag_predict_probability)  // 如果预测概率标志为真
        free(prob_estimates);  // 释放概率估计数组的内存空间
}

void exit_with_help()  // 定义帮助退出函数
{
    printf(  // 输出帮助信息到标准输出流
    # 打印使用说明
    "Usage: predict [options] test_file model_file output_file\n"
    # 打印选项说明
    "options:\n"
    # 打印选项列表
    "-b probability_estimates: whether to output probability estimates, 0 or 1 (default 0)\n"
    # 退出程序并返回错误码1
    );
    exit(1);
int main(int argc, char **argv)
{
    FILE *input, *output;  // 声明文件指针变量 input 和 output
    int i;  // 声明整型变量 i

    // 解析命令行参数
    for(i=1;i<argc;i++)  // 循环遍历命令行参数
    {
        if(argv[i][0] != '-') break;  // 如果参数不是以 - 开头，则跳出循环
        ++i;  // i 自增
        switch(argv[i-1][1])  // 根据参数的第二个字符进行判断
        {
            case 'b':  // 如果是 -b 参数
                flag_predict_probability = atoi(argv[i]);  // 将参数转换为整型并赋值给 flag_predict_probability
                break;  // 跳出 switch
            default:  // 如果是其它参数
                fprintf(stderr,"unknown option: -%c\n", argv[i-1][1]);  // 输出错误信息
                exit_with_help();  // 调用 exit_with_help 函数
                break;  // 跳出 switch
        }
    }
    if(i>=argc)  // 如果 i 大于等于参数个数
        exit_with_help();  // 调用 exit_with_help 函数

    input = fopen(argv[i],"r");  // 打开文件，只读模式
    if(input == NULL)  // 如果文件打开失败
    {
        fprintf(stderr,"can't open input file %s\n",argv[i]);  // 输出错误信息
        exit(1);  // 退出程序
    }

    output = fopen(argv[i+2],"w");  // 打开文件，写入模式
    if(output == NULL)  // 如果文件打开失败
    {
        fprintf(stderr,"can't open output file %s\n",argv[i+2]);  // 输出错误信息
        exit(1);  // 退出程序
    }

    if((model_=load_model(argv[i+1]))==0)  // 载入模型文件
    {
        fprintf(stderr,"can't open model file %s\n",argv[i+1]);  // 输出错误信息
        exit(1);  // 退出程序
    }

    x = (struct feature_node *) malloc(max_nr_attr*sizeof(struct feature_node));  // 分配内存
    do_predict(input, output, model_);  // 调用 do_predict 函数
    free_and_destroy_model(&model_);  // 释放并销毁模型
    free(line);  // 释放内存
    free(x);  // 释放内存
    fclose(input);  // 关闭文件
    fclose(output);  // 关闭文件
    return 0;  // 返回 0
}
```