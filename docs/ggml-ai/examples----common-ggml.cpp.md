# `ggml\examples\common-ggml.cpp`

```cpp
// 包含通用的 GGML 头文件
#include "common-ggml.h"

// 包含正则表达式和映射的头文件
#include <regex>
#include <map>

// 定义文件类型到枚举类型的映射
static const std::map<std::string, enum ggml_ftype> GGML_FTYPE_MAP = {
    {"q4_0", GGML_FTYPE_MOSTLY_Q4_0},
    {"q4_1", GGML_FTYPE_MOSTLY_Q4_1},
    {"q5_0", GGML_FTYPE_MOSTLY_Q5_0},
    {"q5_1", GGML_FTYPE_MOSTLY_Q5_1},
    {"q8_0", GGML_FTYPE_MOSTLY_Q8_0},
    {"q2_k", GGML_FTYPE_MOSTLY_Q2_K},
    {"q3_k", GGML_FTYPE_MOSTLY_Q3_K},
    {"q4_k", GGML_FTYPE_MOSTLY_Q4_K},
    {"q5_k", GGML_FTYPE_MOSTLY_Q5_K},
    {"q6_k", GGML_FTYPE_MOSTLY_Q6_K},
};

// 打印文件类型映射
void ggml_print_ftypes(FILE * fp) {
    for (auto it = GGML_FTYPE_MAP.begin(); it != GGML_FTYPE_MAP.end(); it++) {
        fprintf(fp, "  type = \"%s\" or %d\n", it->first.c_str(), it->second);
    }
}

// 解析文件类型字符串为枚举类型
enum ggml_ftype ggml_parse_ftype(const char * str) {
    enum ggml_ftype ftype;
    if (str[0] == 'q') {
        const auto it = GGML_FTYPE_MAP.find(str);
        if (it == GGML_FTYPE_MAP.end()) {
            fprintf(stderr, "%s: unknown ftype '%s'\n", __func__, str);
            return GGML_FTYPE_UNKNOWN;
        }
        ftype = it->second;
    } else {
        ftype = (enum ggml_ftype) atoi(str);
    }

    return ftype;
}

// 对文件进行量化处理
bool ggml_common_quantize_0(
        std::ifstream & finp,
        std::ofstream & fout,
        const ggml_ftype ftype,
        const std::vector<std::string> & to_quant,
        const std::vector<std::string> & to_skip) {

    // 初始化量化类型为 GGML_TYPE_F32
    ggml_type qtype = GGML_TYPE_F32;
    # 根据文件类型设置量化类型
    switch (ftype) {
        case GGML_FTYPE_MOSTLY_Q4_0: qtype = GGML_TYPE_Q4_0; break;
        case GGML_FTYPE_MOSTLY_Q4_1: qtype = GGML_TYPE_Q4_1; break;
        case GGML_FTYPE_MOSTLY_Q5_0: qtype = GGML_TYPE_Q5_0; break;
        case GGML_FTYPE_MOSTLY_Q5_1: qtype = GGML_TYPE_Q5_1; break;
        case GGML_FTYPE_MOSTLY_Q8_0: qtype = GGML_TYPE_Q8_0; break;
        case GGML_FTYPE_MOSTLY_Q2_K: qtype = GGML_TYPE_Q2_K; break;
        case GGML_FTYPE_MOSTLY_Q3_K: qtype = GGML_TYPE_Q3_K; break;
        case GGML_FTYPE_MOSTLY_Q4_K: qtype = GGML_TYPE_Q4_K; break;
        case GGML_FTYPE_MOSTLY_Q5_K: qtype = GGML_TYPE_Q5_K; break;
        case GGML_FTYPE_MOSTLY_Q6_K: qtype = GGML_TYPE_Q6_K; break;
        case GGML_FTYPE_UNKNOWN:
        case GGML_FTYPE_ALL_F32:
        case GGML_FTYPE_MOSTLY_F16:
        case GGML_FTYPE_MOSTLY_Q4_1_SOME_F16:
                {
                    # 打印错误信息并返回 false
                    fprintf(stderr, "%s: invalid model type %d\n", __func__, ftype);
                    return false;
                }
    };

    # 如果量化类型不合法，打印错误信息并返回 false
    if (!ggml_is_quantized(qtype)) {
        fprintf(stderr, "%s: invalid quantization type %d (%s)\n", __func__, qtype, ggml_type_name(qtype));
        return false;
    }

    # 初始化变量
    size_t total_size_org = 0;
    size_t total_size_new = 0;

    # 创建各种类型的向量
    std::vector<float> work;

    std::vector<uint8_t>     data_u8;
    std::vector<ggml_fp16_t> data_f16;
    std::vector<float>       data_f32;

    # 创建指定大小的 int64_t 向量
    std::vector<int64_t> hist_all(1 << 4, 0);

    }

    # 打印模型大小和量化大小信息
    printf("%s: model size  = %8.2f MB\n", __func__, total_size_org/1024.0/1024.0);
    printf("%s: quant size  = %8.2f MB | ftype = %d (%s)\n", __func__, total_size_new/1024.0/1024.0, ftype, ggml_type_name(qtype));

    {
        # 计算 hist_all 中所有元素的和
        int64_t sum_all = 0;
        for (int i = 0; i < (int) hist_all.size(); ++i) {
            sum_all += hist_all[i];
        }

        # 打印 hist_all 中每个元素的比例
        printf("%s: hist: ", __func__);
        for (int i = 0; i < (int) hist_all.size(); ++i) {
            printf("%5.3f ", hist_all[i] / (float)sum_all);
        }
        printf("\n");
    # 结束 if 语句的代码块，返回 true
    }
    # 返回 true
    return true;
# 闭合前面的函数定义
```