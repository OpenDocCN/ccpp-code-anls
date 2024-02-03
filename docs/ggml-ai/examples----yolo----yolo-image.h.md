# `ggml\examples\yolo\yolo-image.h`

```cpp
// 防止头文件被重复包含
#pragma once

// 包含必要的头文件
#include <string>
#include <vector>
#include <cassert>

// 定义 yolo_image 结构体
struct yolo_image {
    int w, h, c; // 图像的宽度、高度和通道数
    std::vector<float> data; // 存储图像数据的向量

    // 默认构造函数
    yolo_image() : w(0), h(0), c(0) {}

    // 带参数的构造函数
    yolo_image(int w, int h, int c) : w(w), h(h), c(c), data(w*h*c) {}

    // 获取像素值的函数
    float get_pixel(int x, int y, int c) const {
        assert(x >= 0 && x < w && y >= 0 && y < h && c >= 0 && c < this->c);
        return data[c*w*h + y*w + x];
    }

    // 设置像素值的函数
    void set_pixel(int x, int y, int c, float val) {
        assert(x >= 0 && x < w && y >= 0 && y < h && c >= 0 && c < this->c);
        data[c*w*h + y*w + x] = val;
    }

    // 像素值增加的函数
    void add_pixel(int x, int y, int c, float val) {
        assert(x >= 0 && x < w && y >= 0 && y < h && c >= 0 && c < this->c);
        data[c*w*h + y*w + x] += val;
    }

    // 填充图像数据的函数
    void fill(float val) {
        std::fill(data.begin(), data.end(), val);
    }
};

// 加载图像的函数声明
bool load_image(const char *fname, yolo_image & img);

// 绘制带宽度的框的函数声明
void draw_box_width(yolo_image & a, int x1, int y1, int x2, int y2, int w, float r, float g, float b);

// 调整图像大小的函数声明
yolo_image letterbox_image(const yolo_image & im, int w, int h);

// 保存图像的函数声明
bool save_image(const yolo_image & im, const char *name, int quality);

// 获取标签的函数声明
yolo_image get_label(const std::vector<yolo_image> & alphabet, const std::string & label, int size);

// 绘制标签的函数声明
void draw_label(yolo_image & im, int row, int col, const yolo_image & label, const float * rgb);
```