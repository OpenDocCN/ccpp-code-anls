# `ggml\examples\yolo\yolo-image.cpp`

```
// 定义 STB_IMAGE_IMPLEMENTATION，用于包含 stb_image 库的实现代码
#define STB_IMAGE_IMPLEMENTATION
// 包含 stb_image 库的头文件
#include "stb_image.h"
// 定义 STB_IMAGE_WRITE_IMPLEMENTATION，用于包含 stb_image_write 库的实现代码
#define STB_IMAGE_WRITE_IMPLEMENTATION
// 包含 stb_image_write 库的头文件
#include "stb_image_write.h"

// 包含自定义的 yolo-image.h 头文件
#include "yolo-image.h"

// 定义一个静态函数，用于在图像上绘制矩形框
static void draw_box(yolo_image & a, int x1, int y1, int x2, int y2, float r, float g, float b)
{
    // 对矩形框的坐标进行边界处理
    if (x1 < 0) x1 = 0;
    if (x1 >= a.w) x1 = a.w-1;
    if (x2 < 0) x2 = 0;
    if (x2 >= a.w) x2 = a.w-1;

    if (y1 < 0) y1 = 0;
    if (y1 >= a.h) y1 = a.h-1;
    if (y2 < 0) y2 = 0;
    if (y2 >= a.h) y2 = a.h-1;

    // 在图像上绘制矩形框
    for (int i = x1; i <= x2; ++i){
        a.data[i + y1*a.w + 0*a.w*a.h] = r;
        a.data[i + y2*a.w + 0*a.w*a.h] = r;

        a.data[i + y1*a.w + 1*a.w*a.h] = g;
        a.data[i + y2*a.w + 1*a.w*a.h] = g;

        a.data[i + y1*a.w + 2*a.w*a.h] = b;
        a.data[i + y2*a.w + 2*a.w*a.h] = b;
    }
    for (int i = y1; i <= y2; ++i){
        a.data[x1 + i*a.w + 0*a.w*a.h] = r;
        a.data[x2 + i*a.w + 0*a.w*a.h] = r;

        a.data[x1 + i*a.w + 1*a.w*a.h] = g;
        a.data[x2 + i*a.w + 1*a.w*a.h] = g;

        a.data[x1 + i*a.w + 2*a.w*a.h] = b;
        a.data[x2 + i*a.w + 2*a.w*a.h] = b;
    }
}

// 定义一个函数，用于在图像上绘制带有指定宽度的矩形框
void draw_box_width(yolo_image & a, int x1, int y1, int x2, int y2, int w, float r, float g, float b)
{
    // 遍历绘制多个宽度的矩形框
    for (int i = 0; i < w; ++i) {
        draw_box(a, x1+i, y1+i, x2-i, y2-i, r, g, b);
    }
}

// 定义一个函数，用于保存图像到文件
bool save_image(const yolo_image & im, const char *name, int quality)
{
    // 分配内存用于存储图像数据
    uint8_t *data = (uint8_t*)calloc(im.w*im.h*im.c, sizeof(uint8_t));
    // 将图像数据转换为合适的格式
    for (int k = 0; k < im.c; ++k) {
        for (int i = 0; i < im.w*im.h; ++i) {
            data[i*im.c+k] = (uint8_t) (255*im.data[i + k*im.w*im.h]);
        }
    }
    // 将图像数据写入文件
    int success = stbi_write_jpg(name, im.w, im.h, im.c, data, quality);
    // 释放内存
    free(data);
    // 检查是否成功写入文件
    if (!success) {
        fprintf(stderr, "Failed to write image %s\n", name);
        return false;
    }
    return true;
}

// 定义一个函数，用于从文件加载图像
bool load_image(const char *fname, yolo_image & img)
{
    int w, h, c;
    // 使用 stb_image 库加载图像数据
    uint8_t * data = stbi_load(fname, &w, &h, &c, 3);
    // 检查是否成功加载图像数据
    if (!data) {
        return false;
    }
    // 其他处理...
}
    # 设置图像的通道数为3
    c = 3;
    # 设置图像的宽度
    img.w = w;
    # 设置图像的高度
    img.h = h;
    # 设置图像的通道数
    img.c = c;
    # 调整图像数据的大小为宽度乘以高度乘以通道数
    img.data.resize(w*h*c);
    # 遍历每个通道
    for (int k = 0; k < c; ++k){
        # 遍历每一行
        for (int j = 0; j < h; ++j){
            # 遍历每一列
            for (int i = 0; i < w; ++i){
                # 计算目标索引
                int dst_index = i + w*j + w*h*k;
                # 计算源索引
                int src_index = k + c*i + c*w*j;
                # 将源数据转换为浮点数并存储到目标索引位置
                img.data[dst_index] = (float)data[src_index]/255.;
            }
        }
    }
    # 释放原始数据的内存
    stbi_image_free(data);
    # 返回处理结果
    return true;
    // 创建一个新的图像对象，用于存储调整大小后的图像
    yolo_image resized(w, h, im.c);
    // 创建一个临时图像对象，用于存储调整大小后的部分图像
    yolo_image part(w, im.h, im.c);
    // 计算宽度和高度的缩放比例
    float w_scale = (float)(im.w - 1) / (w - 1);
    float h_scale = (float)(im.h - 1) / (h - 1);
    // 遍历图像的通道
    for (int k = 0; k < im.c; ++k){
        // 遍历图像的高度
        for (int r = 0; r < im.h; ++r) {
            // 遍历图像的宽度
            for (int c = 0; c < w; ++c) {
                // 初始化像素值
                float val = 0;
                // 如果当前列是最后一列或者图像宽度为1，则直接取最后一个像素的值
                if (c == w-1 || im.w == 1){
                    val = im.get_pixel(im.w-1, r, k);
                } else {
                    // 计算插值像素的值
                    float sx = c*w_scale;
                    int ix = (int) sx;
                    float dx = sx - ix;
                    val = (1 - dx) * im.get_pixel(ix, r, k) + dx * im.get_pixel(ix+1, r, k);
                }
                // 设置部分图像的像素值
                part.set_pixel(c, r, k, val);
            }
        }
    }
    // 遍历图像的通道
    for (int k = 0; k < im.c; ++k){
        // 遍历新高度
        for (int r = 0; r < h; ++r){
            // 计算高度的缩放比例
            float sy = r*h_scale;
            int iy = (int) sy;
            float dy = sy - iy;
            // 遍历新宽度
            for (int c = 0; c < w; ++c){
                // 计算插值像素的值
                float val = (1-dy) * part.get_pixel(c, iy, k);
                // 设置调整大小后图像的像素值
                resized.set_pixel(c, r, k, val);
            }
            // 如果当前行是最后一行或者图像高度为1，则跳过
            if (r == h-1 || im.h == 1) continue;
            // 遍历新宽度
            for (int c = 0; c < w; ++c){
                // 计算插值像素的值
                float val = dy * part.get_pixel(c, iy+1, k);
                // 设置调整大小后图像的像素值
                resized.add_pixel(c, r, k, val);
            }
        }
    }
    // 返回调整大小后的图像
    return resized;
}

// 将源图像嵌入到目标图像的指定位置
static void embed_image(const yolo_image & source, yolo_image & dest, int dx, int dy)
{
    // 遍历图像的通道
    for (int k = 0; k < source.c; ++k) {
        // 遍历源图像的高度
        for (int y = 0; y < source.h; ++y) {
            // 遍历源图像的宽度
            for (int x = 0; x < source.w; ++x) {
                // 获取源图像的像素值
                float val = source.get_pixel(x, y, k);
                // 将源图像的像素值嵌入到目标图像的指定位置
                dest.set_pixel(dx+x, dy+y, k, val);
            }
        }
    }
}

// 在保持图像纵横比的情况下，将图像调整为指定的宽度和高度
yolo_image letterbox_image(const yolo_image & im, int w, int h)
{
    // 初始化新的宽度和高度
    int new_w = im.w;
    int new_h = im.h;
    // 如果宽度的缩放比例小于高度的缩放比例，则重新计算宽度和高度
    if (((float)w/im.w) < ((float)h/im.h)) {
        new_w = w;
        new_h = (im.h * w)/im.w;
    } else {
        // 如果图片的宽高比大于目标宽高比，则以高度为准进行缩放
        new_h = h;
        // 根据原始图片的宽高比和新的高度计算出新的宽度
        new_w = (im.w * h)/im.h;
    }
    // 调用 resize_image 函数对图片进行缩放
    yolo_image resized = resize_image(im, new_w, new_h);
    // 创建一个指定宽高和通道数的黑色图片
    yolo_image boxed(w, h, im.c);
    // 用灰色填充图片
    boxed.fill(0.5);
    // 将缩放后的图片嵌入到黑色图片中心
    embed_image(resized, boxed, (w-new_w)/2, (h-new_h)/2);
    // 返回嵌入后的图片
    return boxed;
# 将两个图像拼接在一起，返回拼接后的图像
static yolo_image tile_images(const yolo_image & a, const yolo_image & b, int dx)
{
    # 如果第一个图像的宽度为0，则返回第二个图像
    if (a.w == 0) {
        return b;
    }
    # 创建一个新的图像，宽度为两个图像宽度之和加上dx，高度为两个图像高度的最大值，通道数为a的通道数
    yolo_image c(a.w + b.w + dx, (a.h > b.h) ? a.h : b.h, a.c);
    # 用1.0填充新图像
    c.fill(1.0f);
    # 将第一个图像嵌入到新图像中，起始位置为(0,0)
    embed_image(a, c, 0, 0);
    # 将第二个图像嵌入到新图像中，起始位置为(a的宽度+dx,0)
    embed_image(b, c, a.w + dx, 0);
    # 返回拼接后的图像
    return c;
}

# 给图像添加边框，返回添加边框后的图像
static yolo_image border_image(const yolo_image & a, int border)
{
    # 创建一个新的图像，宽度为原图像宽度加上两倍的边框宽度，高度为原图像高度加上两倍的边框宽度，通道数为原图像通道数
    yolo_image b(a.w + 2*border, a.h + 2*border, a.c);
    # 用1.0填充新图像
    b.fill(1.0f);
    # 将原图像嵌入到新图像中，起始位置为(border,border)
    embed_image(a, b, border, border);
    # 返回添加边框后的图像
    return b;
}

# 根据给定的字母表和标签，返回标签对应的图像
yolo_image get_label(const std::vector<yolo_image> & alphabet, const std::string & label, int size)
{
    # 将size除以10
    size = size/10;
    # 取size和7中的较小值
    size = std::min(size, 7);
    # 创建一个空的图像
    yolo_image result(0,0,0);
    # 遍历标签中的每个字符
    for (int i = 0; i < (int)label.size(); ++i) {
        # 获取当前字符的图像
        int ch = label[i];
        yolo_image img = alphabet[size*128 + ch];
        # 将当前字符的图像与之前的结果图像拼接在一起
        result = tile_images(result, img, -size - 1 + (size+1)/2);
    }
    # 返回添加边框后的结果图像
    return border_image(result, (int)(result.h*.25));
}

# 在图像上绘制标签
void draw_label(yolo_image & im, int row, int col, const yolo_image & label, const float * rgb)
{
    # 获取标签图像的宽度和高度
    int w = label.w;
    int h = label.h;
    # 如果行号减去标签高度大于等于0，则将行号更新为行号减去标签高度
    if (row - h >= 0) {
        row = row - h;
    }
    # 遍历标签图像的每个像素
    for (int j = 0; j < h && j + row < im.h; j++) {
        for (int i = 0; i < w && i + col < im.w; i++) {
            # 遍历标签图像的每个通道
            for (int k = 0; k < label.c; k++) {
                # 获取标签图像当前像素的值
                float val = label.get_pixel(i, j, k);
                # 将图像im中对应位置的像素值更新为rgb[k]乘以val
                im.set_pixel(i + col, j + row, k, rgb[k] * val);
            }
        }
    }
}
```