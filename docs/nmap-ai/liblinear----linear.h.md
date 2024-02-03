# `nmap\liblinear\linear.h`

```cpp
#ifndef _LIBLINEAR_H
#define _LIBLINEAR_H

#ifdef __cplusplus
extern "C" {
#endif

struct feature_node
{
    int index;          // 特征索引
    double value;       // 特征值
};

struct problem
{
    int l, n;           // 样本数和特征数
    int *y;             // 标签数组
    struct feature_node **x;  // 特征数组
    double bias;        /* < 0 if no bias term */  // 偏置项，如果没有则小于0
};

enum { L2R_LR, L2R_L2LOSS_SVC_DUAL, L2R_L2LOSS_SVC, L2R_L1LOSS_SVC_DUAL, MCSVM_CS, L1R_L2LOSS_SVC, L1R_LR, L2R_LR_DUAL }; /* solver_type */  // 求解器类型枚举

struct parameter
{
    int solver_type;    // 求解器类型

    /* these are for training only */
    double eps;         // 停止标准
    double C;           // 惩罚参数
    int nr_weight;      // 权重数量
    int *weight_label;  // 权重标签数组
    double* weight;     // 权重数组
};

struct model
{
    struct parameter param;  // 参数
    int nr_class;        /* number of classes */  // 类别数量
    int nr_feature;      // 特征数量
    double *w;           // 权重数组
    int *label;          /* label of each class */  // 每个类别的标签
    double bias;         // 偏置项
};

struct model* train(const struct problem *prob, const struct parameter *param);  // 训练模型
void cross_validation(const struct problem *prob, const struct parameter *param, int nr_fold, int *target);  // 交叉验证

int predict_values(const struct model *model_, const struct feature_node *x, double* dec_values);  // 预测值
int predict(const struct model *model_, const struct feature_node *x);  // 预测
int predict_probability(const struct model *model_, const struct feature_node *x, double* prob_estimates);  // 预测概率

int save_model(const char *model_file_name, const struct model *model_);  // 保存模型
struct model *load_model(const char *model_file_name);  // 加载模型

int get_nr_feature(const struct model *model_);  // 获取特征数量
int get_nr_class(const struct model *model_);  // 获取类别数量
void get_labels(const struct model *model_, int* label);  // 获取标签

void free_model_content(struct model *model_ptr);  // 释放模型内容
void free_and_destroy_model(struct model **model_ptr_ptr);  // 释放并销毁模型
void destroy_param(struct parameter *param);  // 销毁参数

const char *check_parameter(const struct problem *prob, const struct parameter *param);  // 检查参数
int check_probability_model(const struct model *model);  // 检查概率模型
void set_print_string_function(void (*print_func) (const char*));  // 设置打印字符串函数

#ifdef __cplusplus
}
#endif

#endif /* _LIBLINEAR_H */
```