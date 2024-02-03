# `ggml\examples\yolo\yolov3-tiny.cpp`

```cpp
#include "ggml/ggml.h"  // 引入 ggml 库的头文件
#include "yolo-image.h"  // 引入 yolo-image 的头文件

#include <cmath>  // 引入数学函数库
#include <cstdio>  // 引入标准输入输出库
#include <cstring>  // 引入字符串处理库
#include <ctime>  // 引入时间处理库
#include <string>  // 引入字符串库
#include <vector>  // 引入向量库
#include <algorithm>  // 引入算法库
#include <fstream>  // 引入文件流库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 禁止特定警告
#endif

struct conv2d_layer {
    struct ggml_tensor * weights;  // 权重张量
    struct ggml_tensor * biases;  // 偏置张量
    struct ggml_tensor * scales;  // 缩放张量
    struct ggml_tensor * rolling_mean;  // 滚动均值张量
    struct ggml_tensor * rolling_variance;  // 滚动方差张量
    int padding = 1;  // 填充值，默认为1
    bool batch_normalize = true;  // 是否进行批量归一化，默认为true
    bool activate = true; // 是否激活，true 为 leaky relu，false 为线性
};

struct yolo_model {
    int width = 416;  // 模型宽度，默认为416
    int height = 416;  // 模型高度，默认为416
    std::vector<conv2d_layer> conv2d_layers;  // 卷积层
    struct ggml_context * ctx;  // 上下文
};

struct yolo_layer {
    int classes = 80;  // 类别数，默认为80
    std::vector<int> mask;  // 掩码
    std::vector<float> anchors;  // 锚点
    struct ggml_tensor * predictions;  // 预测张量

    yolo_layer(int classes, const std::vector<int> & mask, const std::vector<float> & anchors, struct ggml_tensor * predictions)
        : classes(classes), mask(mask), anchors(anchors), predictions(predictions)
    { }

    int entry_index(int location, int entry) const {
        int w = predictions->ne[0];  // 宽度
        int h = predictions->ne[1];  // 高度
        int n = location / (w*h);  // n
        int loc = location % (w*h);  // loc
        return n*w*h*(4+classes+1) + entry*w*h + loc;  // 返回索引
    }
};

struct box {
    float x, y, w, h;  // 盒子的坐标和尺寸
};

struct detection {
    box bbox;  // 盒子
    std::vector<float> prob;  // 概率
    float objectness;  // 物体性
};

static bool load_model(const std::string & fname, yolo_model & model) {
    struct gguf_init_params params = {
        /*.no_alloc   =*/ false,  // 不进行分配内存
        /*.ctx        =*/ &model.ctx,  // 上下文
    };
    gguf_context * ctx = gguf_init_from_file(fname.c_str(), params);  // 从文件中初始化上下文
    if (!ctx) {
        fprintf(stderr, "%s: gguf_init_from_file() failed\n", __func__);  // 打印错误信息
        return false;  // 返回失败
    }
    model.width  = 416;  // 设置模型宽度为416
    model.height = 416;  // 设置模型高度为416
    model.conv2d_layers.resize(13);  // 调整卷积层大小为13
    # 设置第7个卷积层的填充为0
    model.conv2d_layers[7].padding = 0;
    # 设置第9个卷积层的填充为0
    model.conv2d_layers[9].padding = 0;
    # 设置第9个卷积层的批量归一化为false
    model.conv2d_layers[9].batch_normalize = false;
    # 设置第9个卷积层的激活函数为false
    model.conv2d_layers[9].activate = false;
    # 设置第10个卷积层的填充为0
    model.conv2d_layers[10].padding = 0;
    # 设置第12个卷积层的填充为0
    model.conv2d_layers[12].padding = 0;
    # 设置第12个卷积层的批量归一化为false
    model.conv2d_layers[12].batch_normalize = false;
    # 设置第12个卷积层的激活函数为false
    model.conv2d_layers[12].activate = false;
    # 遍历卷积层列表
    for (int i = 0; i < (int)model.conv2d_layers.size(); i++) {
        # 定义存储名称的字符数组
        char name[256];
        # 格式化生成权重名称
        snprintf(name, sizeof(name), "l%d_weights", i);
        # 获取并设置卷积层的权重
        model.conv2d_layers[i].weights = ggml_get_tensor(model.ctx, name);
        # 格式化生成偏置名称
        snprintf(name, sizeof(name), "l%d_biases", i);
        # 获取并设置卷积层的偏置
        model.conv2d_layers[i].biases = ggml_get_tensor(model.ctx, name);
        # 如果卷积层使用了批量归一化
        if (model.conv2d_layers[i].batch_normalize) {
            # 格式化生成缩放参数名称
            snprintf(name, sizeof(name), "l%d_scales", i);
            # 获取并设置卷积层的缩放参数
            model.conv2d_layers[i].scales = ggml_get_tensor(model.ctx, name);
            # 格式化生成滚动均值名称
            snprintf(name, sizeof(name), "l%d_rolling_mean", i);
            # 获取并设置卷积层的滚动均值
            model.conv2d_layers[i].rolling_mean = ggml_get_tensor(model.ctx, name);
            # 格式化生成滚动方差名称
            snprintf(name, sizeof(name), "l%d_rolling_variance", i);
            # 获取并设置卷积层的滚动方差
            model.conv2d_layers[i].rolling_variance = ggml_get_tensor(model.ctx, name);
        }
    }
    # 返回true
    return true;
// 加载标签数据到指定的字符串向量中
static bool load_labels(const char * filename, std::vector<std::string> & labels)
{
    // 打开指定文件
    std::ifstream file_in(filename);
    // 如果文件无法打开，则返回false
    if (!file_in) {
        return false;
    }
    // 逐行读取文件内容，将每行字符串添加到标签向量中
    std::string line;
    while (std::getline(file_in, line)) {
        labels.push_back(line);
    }
    // 断言标签数量为80
    GGML_ASSERT(labels.size() == 80);
    return true;
}

// 加载字母表数据到指定的yolo_image向量中
static bool load_alphabet(std::vector<yolo_image> & alphabet)
{
    // 调整字母表向量大小
    alphabet.resize(8 * 128);
    // 遍历字母表
    for (int j = 0; j < 8; j++) {
        for (int i = 32; i < 127; i++) {
            // 构建文件名
            char fname[256];
            sprintf(fname, "data/labels/%d_%d.png", i, j);
            // 如果无法加载图像，则打印错误信息并返回false
            if (!load_image(fname, alphabet[j*128 + i])) {
                fprintf(stderr, "Cannot load '%s'\n", fname);
                return false;
            }
        }
    }
    return true;
}

// 应用卷积操作
static ggml_tensor * apply_conv2d(ggml_context * ctx, ggml_tensor * input, const conv2d_layer & layer)
{
    // 执行卷积操作
    struct ggml_tensor * result = ggml_conv_2d(ctx, layer.weights, input, 1, 1, layer.padding, layer.padding, 1, 1);
    // 如果启用了批量归一化
    if (layer.batch_normalize) {
        // 执行归一化操作
        result = ggml_sub(ctx, result, ggml_repeat(ctx, layer.rolling_mean, result));
        result = ggml_div(ctx, result, ggml_sqrt(ctx, ggml_repeat(ctx, layer.rolling_variance, result)));
        result = ggml_mul(ctx, result, ggml_repeat(ctx, layer.scales, result));
    }
    // 添加偏置
    result = ggml_add(ctx, result, ggml_repeat(ctx, layer.biases, result));
    // 如果启用了激活函数
    if (layer.activate) {
        // 执行Leaky ReLU激活函数
        result = ggml_leaky_relu(ctx, result, 0.1f, true);
    }
    return result;
}

// 激活数组
static void activate_array(float * x, const int n)
{
    // 逐元素执行logistic激活函数
    for (int i = 0; i < n; i++) {
        x[i] = 1./(1. + exp(-x[i]));
    }
}

// 应用YOLO算法
static void apply_yolo(yolo_layer & layer)
{
    // 获取预测数据的宽度和高度
    int w = layer.predictions->ne[0];
    int h = layer.predictions->ne[1];
    // 获取掩码的大小
    int N = layer.mask.size();
    // 获取预测数据的指针
    float * data = ggml_get_data_f32(layer.predictions);
}
    # 遍历从 0 到 N-1 的整数
    for (int n = 0; n < N; n++) {
        # 计算当前层的入口索引，根据当前层的索引和宽高计算
        int index = layer.entry_index(n*w*h, 0);
        # 激活数据数组中从索引位置开始的 2*w*h 个元素
        activate_array(data + index, 2*w*h);
        # 重新计算入口索引，根据当前层的索引和宽高计算
        index = layer.entry_index(n*w*h, 4);
        # 激活数据数组中从索引位置开始的 (1+layer.classes)*w*h 个元素
        activate_array(data + index, (1+layer.classes)*w*h);
    }
# 获取 YOLO 算法的边界框，根据预测结果和参数计算边界框的位置和大小
static box get_yolo_box(const yolo_layer & layer, int n, int index, int i, int j, int lw, int lh, int w, int h, int stride)
{
    # 获取 YOLO 算法的预测结果数据
    float * predictions = ggml_get_data_f32(layer.predictions);
    # 创建边界框对象
    box b;
    # 计算边界框的 x 坐标
    b.x = (i + predictions[index + 0*stride]) / lw;
    # 计算边界框的 y 坐标
    b.y = (j + predictions[index + 1*stride]) / lh;
    # 计算边界框的宽度
    b.w = exp(predictions[index + 2*stride]) * layer.anchors[2*n]   / w;
    # 计算边界框的高度
    b.h = exp(predictions[index + 3*stride]) * layer.anchors[2*n+1] / h;
    # 返回计算得到的边界框
    return b;
}

# 根据图像和网络尺寸，修正 YOLO 算法的边界框
static void correct_yolo_box(box & b, int im_w, int im_h, int net_w, int net_h)
{
    # 初始化新的图像宽度和高度
    int new_w = 0;
    int new_h = 0;
    # 根据图像和网络尺寸比较，计算新的图像宽度和高度
    if (((float)net_w/im_w) < ((float)net_h/im_h)) {
        new_w = net_w;
        new_h = (im_h * net_w)/im_w;
    } else {
        new_h = net_h;
        new_w = (im_w * net_h)/im_h;
    }
    # 修正边界框的 x 和 y 坐标
    b.x = (b.x - (net_w - new_w)/2./net_w) / ((float)new_w/net_w);
    b.y = (b.y - (net_h - new_h)/2./net_h) / ((float)new_h/net_h);
    # 修正边界框的宽度和高度
    b.w *= (float)net_w/new_w;
    b.h *= (float)net_h/new_h;
}

# 获取 YOLO 算法的检测结果
static void get_yolo_detections(const yolo_layer & layer, std::vector<detection> & detections, int im_w, int im_h, int netw, int neth, float thresh)
{
    # 获取预测结果的宽度和高度
    int w = layer.predictions->ne[0];
    int h = layer.predictions->ne[1];
    # 获取 YOLO 算法的掩码数量
    int N = layer.mask.size();
    # 获取 YOLO 算法的预测结果数据
    float * predictions = ggml_get_data_f32(layer.predictions);
    # 创建存储检测结果的向量
    std::vector<detection> result;
    # 遍历图像中的每个像素
    for (int i = 0; i < w*h; i++) {
        # 遍历每个预测框
        for (int n = 0; n < N; n++) {
            # 计算当前预测框在预测数组中的索引
            int obj_index = layer.entry_index(n*w*h + i, 4);
            # 获取当前预测框的置信度
            float objectness = predictions[obj_index];
            # 如果置信度低于阈值，则跳过当前预测框
            if (objectness <= thresh) {
                continue;
            }
            # 创建检测对象
            detection det;
            # 计算当前预测框边界框在预测数组中的索引
            int box_index = layer.entry_index(n*w*h + i, 0);
            # 计算当前像素在图像中的行列位置
            int row = i / w;
            int col = i % w;
            # 获取 YOLO 检测框的边界框
            det.bbox = get_yolo_box(layer, layer.mask[n], box_index, col, row, w, h, netw, neth, w*h);
            # 校正 YOLO 检测框的边界框
            correct_yolo_box(det.bbox, im_w, im_h, netw, neth);
            # 设置检测对象的置信度
            det.objectness = objectness;
            # 调整检测对象的类别概率数组大小
            det.prob.resize(layer.classes);
            # 遍历每个类别
            for (int j = 0; j < layer.classes; j++) {
                # 计算当前类别在预测数组中的索引
                int class_index = layer.entry_index(n*w*h + i, 4 + 1 + j);
                # 计算当前类别的概率
                float prob = objectness*predictions[class_index];
                # 如果概率大于阈值，则将概率赋值给检测对象的类别概率数组
                det.prob[j] = (prob > thresh) ? prob : 0;
            }
            # 将检测对象添加到检测结果数组中
            detections.push_back(det);
        }
    }
static float overlap(float x1, float w1, float x2, float w2)
{
    // 计算第一个矩形的左边界
    float l1 = x1 - w1/2;
    // 计算第二个矩形的左边界
    float l2 = x2 - w2/2;
    // 取两个矩形左边界的最大值
    float left = l1 > l2 ? l1 : l2;
    // 计算第一个矩形的右边界
    float r1 = x1 + w1/2;
    // 计算第二个矩形的右边界
    float r2 = x2 + w2/2;
    // 取两个矩形右边界的最小值
    float right = r1 < r2 ? r1 : r2;
    // 返回重叠部分的宽度
    return right - left;
}

static float box_intersection(const box & a, const box & b)
{
    // 计算两个矩形的重叠部分的宽度
    float w = overlap(a.x, a.w, b.x, b.w);
    // 计算两个矩形的重叠部分的高度
    float h = overlap(a.y, a.h, b.y, b.h);
    // 如果重叠部分的宽度或高度小于0，则返回0
    if (w < 0 || h < 0) return 0;
    // 计算重叠部分的面积
    float area = w*h;
    // 返回重叠部分的面积
    return area;
}

static float box_union(const box & a, const box & b)
{
    // 计算两个矩形的交集面积
    float i = box_intersection(a, b);
    // 计算两个矩形的并集面积
    float u = a.w*a.h + b.w*b.h - i;
    // 返回两个矩形的并集面积
    return u;
}

static float box_iou(const box & a, const box & b)
{
    // 计算两个矩形的交并比
    return box_intersection(a, b)/box_union(a, b);
}

static void do_nms_sort(std::vector<detection> & dets, int classes, float thresh)
{
    // 获取检测结果的数量
    int k = (int)dets.size()-1;
    // 遍历检测结果
    for (int i = 0; i <= k; ++i) {
        // 如果检测结果的置信度为0，则将其与最后一个检测结果交换位置
        if (dets[i].objectness == 0) {
            std::swap(dets[i], dets[k]);
            --k;
            --i;
        }
    }
    // 计算有效的检测结果数量
    int total = k+1;
    // 遍历每个类别
    for (int k = 0; k < classes; ++k) {
        // 对检测结果按照当前类别的置信度进行排序
        std::sort(dets.begin(), dets.begin()+total, [=](const detection & a, const detection & b) {
            return a.prob[k] > b.prob[k];
        });
        // 遍历检测结果
        for (int i = 0; i < total; ++i) {
            // 如果当前检测结果的置信度为0，则跳过
            if (dets[i].prob[k] == 0) {
                continue;
            }
            // 获取当前检测结果的边界框
            box a = dets[i].bbox;
            // 遍历后续的检测结果
            for (int j = i+1; j < total; ++j){
                // 获取后续检测结果的边界框
                box b = dets[j].bbox;
                // 如果当前检测结果与后续检测结果的交并比大于阈值，则将后续检测结果的置信度设为0
                if (box_iou(a, b) > thresh) {
                    dets[j].prob[k] = 0;
                }
            }
        }
    }
}

static float get_color(int c, int x, int max)
{
    // 颜色数组
    float colors[6][3] = { {1,0,1}, {0,0,1}, {0,1,1}, {0,1,0}, {1,1,0}, {1,0,0} };
    // 计算颜色的比例
    float ratio = ((float)x/max)*5;
    // 计算颜色的索引
    int i = floor(ratio);
    int j = ceil(ratio);
    ratio -= i;
    // 根据比例计算颜色值
    float r = (1-ratio) * colors[i][c] + ratio*colors[j][c];
    // 返回颜色值
    return r;
}
// 绘制检测结果
static void draw_detections(yolo_image & im, const std::vector<detection> & dets, float thresh, const std::vector<std::string> & labels, const std::vector<yolo_image> & alphabet)
{
    // 获取类别数量
    int classes = (int)labels.size();
    // 遍历检测结果
    for (int i = 0; i < (int)dets.size(); i++) {
        std::string labelstr;
        int cl = -1;
        // 遍历每个类别的概率
        for (int j = 0; j < (int)dets[i].prob.size(); j++) {
            // 如果概率大于阈值
            if (dets[i].prob[j] > thresh) {
                // 如果是第一个类别
                if (cl < 0) {
                    labelstr = labels[j];
                    cl = j;
                } else {
                    labelstr += ", ";
                    labelstr += labels[j];
                }
                // 打印类别和概率
                printf("%s: %.0f%%\n", labels[j].c_str(), dets[i].prob[j]*100);
            }
        }
        // 如果存在检测结果
        if (cl >= 0) {
            int width = im.h * .006;
            int offset = cl*123457 % classes;
            float red = get_color(2,offset,classes);
            float green = get_color(1,offset,classes);
            float blue = get_color(0,offset,classes);
            float rgb[3];

            rgb[0] = red;
            rgb[1] = green;
            rgb[2] = blue;
            box b = dets[i].bbox;

            int left  = (b.x-b.w/2.)*im.w;
            int right = (b.x+b.w/2.)*im.w;
            int top   = (b.y-b.h/2.)*im.h;
            int bot   = (b.y+b.h/2.)*im.h;

            // 确保边界在图像范围内
            if (left < 0) left = 0;
            if (right > im.w-1) right = im.w-1;
            if (top < 0) top = 0;
            if (bot > im.h-1) bot = im.h-1;

            // 绘制边界框
            draw_box_width(im, left, top, right, bot, width, red, green, blue);
            // 获取标签
            yolo_image label = get_label(alphabet, labelstr, (im.h*.03));
            // 绘制标签
            draw_label(im, top + width, left, label, rgb);
        }
    }
}

// 打印张量的形状
static void print_shape(int layer, const ggml_tensor * t)
{
    // 打印层的输出形状
    printf("Layer %2d output shape:  %3d x %3d x %4d x %3d\n", layer, (int)t->ne[0], (int)t->ne[1], (int)t->ne[2], (int)t->ne[3]);
}
void detect(yolo_image & img, const yolo_model & model, float thresh, const std::vector<std::string> & labels, const std::vector<yolo_image> & alphabet)
{
    // 分配内存空间，用于存储中间结果
    static size_t buf_size = 20000000 * sizeof(float) * 4;
    static void * buf = malloc(buf_size);

    // 初始化神经网络计算环境参数
    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,  // 内存大小
        /*.mem_buffer =*/ buf,       // 内存缓冲区
        /*.no_alloc   =*/ false,     // 是否允许分配内存
    };

    // 初始化神经网络计算环境
    struct ggml_context * ctx0 = ggml_init(params);
    // 创建新的计算图
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);
    // 存储检测结果
    std::vector<detection> detections;

    // 调整图像大小以适应模型输入
    yolo_image sized = letterbox_image(img, model.width, model.height);
    // 创建输入张量
    struct ggml_tensor * input = ggml_new_tensor_4d(ctx0, GGML_TYPE_F32, model.width, model.height, 3, 1);
    // 将图像数据复制到输入张量
    std::memcpy(input->data, sized.data.data(), ggml_nbytes(input));
    ggml_set_name(input, "input");

    // 应用卷积操作
    struct ggml_tensor * result = apply_conv2d(ctx0, input, model.conv2d_layers[0]);
    print_shape(0, result);
    // 对结果进行池化操作
    result = ggml_pool_2d(ctx0, result, GGML_OP_POOL_MAX, 2, 2, 2, 2, 0, 0);
    print_shape(1, result);
    // 应用卷积操作
    result = apply_conv2d(ctx0, result, model.conv2d_layers[1]);
    print_shape(2, result);
    // 对结果进行池化操作
    result = ggml_pool_2d(ctx0, result, GGML_OP_POOL_MAX, 2, 2, 2, 2, 0, 0);
    print_shape(3, result);
    // 应用卷积操作
    result = apply_conv2d(ctx0, result, model.conv2d_layers[2]);
    print_shape(4, result);
    // 对结果进行池化操作
    result = ggml_pool_2d(ctx0, result, GGML_OP_POOL_MAX, 2, 2, 2, 2, 0, 0);
    print_shape(5, result);
    // 应用卷积操作
    result = apply_conv2d(ctx0, result, model.conv2d_layers[3]);
    print_shape(6, result);
    // 对结果进行池化操作
    result = ggml_pool_2d(ctx0, result, GGML_OP_POOL_MAX, 2, 2, 2, 2, 0, 0);
    print_shape(7, result);
    // 应用卷积操作
    result = apply_conv2d(ctx0, result, model.conv2d_layers[4]);
    struct ggml_tensor * layer_8 = result;
    print_shape(8, result);
    // 对结果进行池化操作
    result = ggml_pool_2d(ctx0, result, GGML_OP_POOL_MAX, 2, 2, 2, 2, 0, 0);
    print_shape(9, result);
    // 应用卷积操作
    result = apply_conv2d(ctx0, result, model.conv2d_layers[5]);
    print_shape(10, result);
}
    # 对输入数据进行二维池化操作，使用最大池化方式，步长为2，填充为1，池化核大小为2x2，输出缩放因子为0.5
    result = ggml_pool_2d(ctx0, result, GGML_OP_POOL_MAX, 2, 2, 1, 1, 0.5, 0.5);
    # 打印结果数据的形状
    print_shape(11, result);
    # 对输入数据应用第6个卷积层
    result = apply_conv2d(ctx0, result, model.conv2d_layers[6]);
    # 打印结果数据的形状
    print_shape(12, result);
    # 对输入数据应用第7个卷积层
    result = apply_conv2d(ctx0, result, model.conv2d_layers[7]);
    # 将结果数据赋值给 layer_13
    struct ggml_tensor * layer_13 = result;
    # 打印结果数据的形状
    print_shape(13, result);
    # 对输入数据应用第8个卷积层
    result = apply_conv2d(ctx0, result, model.conv2d_layers[8]);
    # 打印结果数据的形状
    print_shape(14, result);
    # 对输入数据应用第9个卷积层
    result = apply_conv2d(ctx0, result, model.conv2d_layers[9]);
    # 将结果数据赋值给 layer_15
    struct ggml_tensor * layer_15 = result;
    # 打印结果数据的形状
    print_shape(15, result);
    # 对 layer_13 和输入数据应用第10个卷积层
    result = apply_conv2d(ctx0, layer_13, model.conv2d_layers[10]);
    # 打印结果数据的形状
    print_shape(18, result);
    # 对结果数据进行上采样操作，缩放因子为2
    result = ggml_upscale(ctx0, result, 2);
    # 打印结果数据的形状
    print_shape(19, result);
    # 对结果数据和 layer_8 进行拼接操作
    result = ggml_concat(ctx0, result, layer_8);
    # 打印结果数据的形状
    print_shape(20, result);
    # 对结果数据应用第11个卷积层
    result = apply_conv2d(ctx0, result, model.conv2d_layers[11]);
    # 打印结果数据的形状
    print_shape(21, result);
    # 对结果数据应用第12个卷积层
    result = apply_conv2d(ctx0, result, model.conv2d_layers[12]);
    # 将结果数据赋值给 layer_22
    struct ggml_tensor * layer_22 = result;
    # 打印结果数据的形状
    print_shape(22, result);

    # 构建前向扩展操作，对 layer_15 进行处理
    ggml_build_forward_expand(gf, layer_15);
    # 构建前向扩展操作，对 layer_22 进行处理
    ggml_build_forward_expand(gf, layer_22);
    # 使用上下文和计算图进行计算
    ggml_graph_compute_with_ctx(ctx0, gf, 1);

    # 创建 yolo16 层，设置类别数、锚点、探测层索引和输入数据
    yolo_layer yolo16{ 80, {3, 4, 5}, {10, 14, 23, 27, 37,58, 81, 82, 135, 169, 344, 319}, layer_15};
    # 应用 yolo16 层
    apply_yolo(yolo16);
    # 获取 yolo16 层的检测结果
    get_yolo_detections(yolo16, detections, img.w, img.h, model.width, model.height, thresh);

    # 创建 yolo23 层，设置类别数、锚点、探测层索引和输入数据
    yolo_layer yolo23{ 80, {0, 1, 2}, {10, 14, 23, 27, 37,58, 81, 82, 135, 169, 344, 319}, layer_22};
    # 应用 yolo23 层
    apply_yolo(yolo23);
    # 获取 yolo23 层的检测结果
    get_yolo_detections(yolo23, detections, img.w, img.h, model.width, model.height, thresh);

    # 对检测结果进行非极大值抑制和排序
    do_nms_sort(detections, yolo23.classes, .45);
    # 绘制检测结果
    draw_detections(img, detections, thresh, labels, alphabet);
    # 释放上下文资源
    ggml_free(ctx0);
}

// 定义 YOLO 参数结构体
struct yolo_params {
    float thresh          = 0.5; // 设置默认的检测阈值
    std::string model     = "yolov3-tiny.gguf"; // 设置默认的模型路径
    std::string fname_inp = "input.jpg"; // 设置默认的输入文件名
    std::string fname_out = "predictions.jpg"; // 设置默认的输出文件名
};

// 打印 YOLO 使用说明
void yolo_print_usage(int argc, char ** argv, const yolo_params & params) {
    fprintf(stderr, "usage: %s [options]\n", argv[0]); // 打印使用说明
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help            show this help message and exit\n"); // 打印帮助信息
    fprintf(stderr, "  -th T, --thresh T     detection threshold (default: %.2f)\n", params.thresh); // 打印检测阈值
    fprintf(stderr, "  -m FNAME, --model FNAME\n");
    fprintf(stderr, "                        model path (default: %s)\n", params.model.c_str()); // 打印模型路径
    fprintf(stderr, "  -i FNAME, --inp FNAME\n");
    fprintf(stderr, "                        input file (default: %s)\n", params.fname_inp.c_str()); // 打印输入文件名
    fprintf(stderr, "  -o FNAME, --out FNAME\n");
    fprintf(stderr, "                        output file (default: %s)\n", params.fname_out.c_str()); // 打印输出文件名
    fprintf(stderr, "\n");
}

// 解析 YOLO 参数
bool yolo_params_parse(int argc, char ** argv, yolo_params & params) {
    for (int i = 1; i < argc; i++) {
        std::string arg = argv[i]; // 获取参数

        if (arg == "-th" || arg == "--thresh") {
            params.thresh = std::stof(argv[++i]); // 设置检测阈值
        } else if (arg == "-m" || arg == "--model") {
            params.model = argv[++i]; // 设置模型路径
        } else if (arg == "-i" || arg == "--inp") {
            params.fname_inp = argv[++i]; // 设置输入文件名
        } else if (arg == "-o" || arg == "--out") {
            params.fname_out = argv[++i]; // 设置输出文件名
        } else if (arg == "-h" || arg == "--help") {
            yolo_print_usage(argc, argv, params); // 打印使用说明
            exit(0); // 退出程序
        } else {
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str()); // 打印错误信息
            yolo_print_usage(argc, argv, params); // 打印使用说明
            exit(0); // 退出程序
        }
    }

    return true; // 返回解析结果
}

int main(int argc, char *argv[])
{
    ggml_time_init(); // 初始化时间
    yolo_model model;
    # 定义一个名为 params 的 yolo_params 结构体变量
    yolo_params params;
    # 如果无法解析命令行参数到 params 变量中，则返回 1
    if (!yolo_params_parse(argc, argv, params)) {
        return 1;
    }
    # 如果无法从指定路径加载模型到 model 变量中，则输出错误信息并返回 1
    if (!load_model(params.model, model)) {
        fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
        return 1;
    }
    # 创建一个名为 img 的 yolo_image 对象，宽高为 0
    yolo_image img(0,0,0);
    # 如果无法从指定路径加载图像到 img 变量中，则输出错误信息并返回 1
    if (!load_image(params.fname_inp.c_str(), img)) {
        fprintf(stderr, "%s: failed to load image from '%s'\n", __func__, params.fname_inp.c_str());
        return 1;
    }
    # 创建一个名为 labels 的字符串向量
    std::vector<std::string> labels;
    # 如果无法从指定路径加载标签到 labels 变量中，则输出错误信息并返回 1
    if (!load_labels("data/coco.names", labels)) {
        fprintf(stderr, "%s: failed to load labels from 'data/coco.names'\n", __func__);
        return 1;
    }
    # 创建一个名为 alphabet 的 yolo_image 向量
    std::vector<yolo_image> alphabet;
    # 如果无法加载字母表到 alphabet 变量中，则输出错误信息并返回 1
    if (!load_alphabet(alphabet)) {
        fprintf(stderr, "%s: failed to load alphabet\n", __func__);
        return 1;
    }
    # 获取当前时间戳，单位为毫秒
    const int64_t t_start_ms = ggml_time_ms();
    # 对图像进行目标检测
    detect(img, model, params.thresh, labels, alphabet);
    # 计算目标检测所花费的时间
    const int64_t t_detect_ms = ggml_time_ms() - t_start_ms;
    # 如果无法将图像保存到指定路径，则输出错误信息并返回 1
    if (!save_image(img, params.fname_out.c_str(), 80)) {
        fprintf(stderr, "%s: failed to save image to '%s'\n", __func__, params.fname_out.c_str());
        return 1;
    }
    # 输出目标检测结果保存的路径和所花费的时间
    printf("Detected objects saved in '%s' (time: %f sec.)\n", params.fname_out.c_str(), t_detect_ms / 1000.0f);
    # 释放模型上下文资源
    ggml_free(model.ctx);
    # 返回 0，表示程序执行成功
    return 0;
# 闭合前面的函数定义
```